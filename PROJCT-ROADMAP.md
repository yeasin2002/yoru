# `openkit` — Decisions, Strategy & Roadmap

---

## Architecture decisions

The pipeline is a strict linear sequence. Every stage is independently replaceable and every future output module gets filtering for free.

```
Input → Parser/Deref → Filter engine → Transformer(s) → Post-processor → Output
```

| Layer | Responsibility | Key packages |
|---|---|---|
| Input | Resolve file path, URL, or stdin to a raw JS object | `fs`, `node-fetch` |
| Parser / deref | Validate spec, dereference all `$ref` | `@apidevtools/swagger-parser` |
| Filter engine | Reduce spec to a relevant subset | `openapi-filter`, `openapi-format` |
| Transformer | Convert filtered spec to target output | per module |
| Post-processor | Format and prettify output | `prettier`, `eta` templates |
| Output | Write to file, stdout, or return string | `fs`, `process.stdout` |

### Core transformer contract

Every transformer is a pure async function. Adding a new output type means implementing one function and registering it — no changes to the pipeline.

```typescript
type Transformer<TOptions = Record<string, unknown>> = (
  spec: OpenAPIObject,  // already parsed + dereffed
  options?: TOptions
) => Promise<string | Record<string, string>>
// string        = single file output
// Record<k, v>  = multi-file output (path → content)
```

### Decision log

**Filter runs before every transformer, not inside them.**
The filter layer executes exactly once. All transformers downstream receive the same already-filtered spec. This means filtering logic is never duplicated across markdown, types, or Zod output modules.

**Two filtering packages, not one.**
`openapi-filter` and `openapi-format` solve different sub-problems. `openapi-filter` handles flag-based filtering (annotations like `x-internal` inside the spec). `openapi-format` handles structural filtering from outside the spec — tags, path regex, methods, operationIDs. They compose cleanly without overlap.

**Always validate after filtering.**
Aggressive filtering can leave dangling `$ref` references. The pipeline re-validates with `swagger-parser` after `openapi-format --removeUnusedComponents` runs, before passing to any transformer.

**Commander over Oclif.**
Startup time matters for `npx` invocations: commander adds ~18ms, Oclif adds ~85–120ms. A tool of this scope (5–6 subcommands, no third-party plugin system in v1) does not need Oclif's overhead. The transformer registry is designed so third-party transformers can be added without a CLI rewrite.

**Atomic writes.**
When multiple output types are generated in one run, the pipeline writes to a temp directory first, then renames atomically on success. A mid-run failure leaves the previous outputs untouched.

**AI enrichment is a post-processing step, not a transformer.**
The AI layer augments output from existing transformers — it does not replace them. It merges AI-generated content (descriptions, examples) into the spec object before a transformer runs, and caches results by operation hash so unchanged operations make zero API calls on re-runs.

---

## Package recommendation matrix

### Parsing & schema manipulation

| Package | Role | Purpose |
|---|---|---|
| `@apidevtools/swagger-parser` | **core** | Validate + dereference OpenAPI 3.x / Swagger 2. Resolves all `$ref` before any transformation. Non-negotiable dependency. |
| `json-schema-ref-parser` | optional | Alternative dereferencer for split multi-file schemas or circular refs. |

### Filtering

| Package | Role | Purpose |
|---|---|---|
| `openapi-filter` | **core** | Flag-based filtering (`x-internal`, custom flags) and `--inverse` mode. Use when the spec is annotated with extension flags. |
| `openapi-format` | **core** | Structural filtering by tags, path regex, methods, operationIDs. Also handles unused component cleanup. Use when filter criteria come from outside the spec. |

### Markdown output

| Package | Role | Purpose |
|---|---|---|
| `@scalar/openapi-to-markdown` | **core** | Primary markdown transformer. Already in use. |

### TypeScript types

| Package | Role | Purpose |
|---|---|---|
| `openapi-typescript` | **core** | Generates zero-runtime TypeScript types. MIT, no Java, runs in milliseconds. Pairs with `openapi-fetch` for type-safe HTTP clients. |

> **Do not use** `openapi-typescript-codegen` (Ferdi Koomen) — deprecated May 2024. `@hey-api/openapi-ts` is the maintained fork.

### Zod schemas

| Package | Role | Purpose |
|---|---|---|
| `openapi-zod-client` | **core** | OpenAPI → Zod schemas + Zodios client. Programmable + CLI. Supports Handlebars templates for custom output shape. |
| `orval` | optional | Alternative when React Query or SWR hooks are needed alongside schemas. Heavier but opinionated. |

### SDK / client generation

| Package | Role | Purpose |
|---|---|---|
| `@hey-api/openapi-ts` | optional | Plugin-based codegen (used by Vercel, PayPal). Generates SDK clients, Zod schemas, TanStack Query hooks. For "full client" output mode. |

### CLI framework

| Package | Role | Purpose |
|---|---|---|
| `commander` | **core** | 35M weekly downloads, ~180KB, zero deps. Best startup performance for `npx` use. |
| `inquirer` | optional | Interactive prompts for `--interactive` / wizard mode. Pair with commander for guided first-run. |

### Formatting & templates

| Package | Role | Purpose |
|---|---|---|
| `prettier` | **core** | Post-processes all output types: markdown, TypeScript, JSON. Consumer's `.prettierrc` applies automatically. |
| `eta` | optional | 2KB template engine for custom markdown layouts. TypeScript-native, async-capable, no dependencies. |

### AI enrichment (future)

| Package | Role | Purpose |
|---|---|---|
| `@anthropic-ai/sdk` | future | Enrich endpoint descriptions, generate usage examples, produce changelog summaries between spec versions. |

---

## Extension strategy

The tool is designed to grow in capability without rewriting the core. There are three extension surfaces.

### 1. New output types (transformers)

Register a new transformer function in the transformer map. The pipeline, filter engine, post-processor, and output writer are all reused automatically.

```typescript
// src/transformers/my-output.ts
import type { Transformer } from '../pipeline'

export const myOutput: Transformer<{ someOption: boolean }> = async (spec, options) => {
  // spec is already parsed + filtered
  return generateSomethingFrom(spec)
}

// src/transformers/index.ts — register
export const transformers = {
  markdown:  () => import('./markdown'),
  'ts-types': () => import('./ts-types'),
  zod:       () => import('./zod'),
  sdk:       () => import('./sdk'),
  'my-output': () => import('./my-output'),  // ← add here
}
```

No changes to `pipeline.ts`, `filter.ts`, `cli.ts`, or `config.ts`.

### 2. Custom markdown templates

Supply an `eta` template via `--template` or the config file. The full filtered spec object is passed as template data.

```typescript
// openkit.config.ts
{
  type: 'markdown',
  outFile: './docs/admin-api.md',
  template: './templates/api-docs.eta',  // ← custom layout
}
```

Inside the template, every OpenAPI object is available:

```eta
# <%= it.info.title %> API Reference

<% for (const [path, item] of Object.entries(it.paths)) { %>
## `<%= path %>`
<% for (const [method, op] of Object.entries(item)) { %>
### <%= method.toUpperCase() %> — <%= op.summary %>
<%= op.description %>
<% } %>
<% } %>
```

### 3. AI enrichment callbacks (future)

The AI layer is opt-in per output type. It is implemented as a pre-transform hook that mutates the spec object before the transformer sees it.

```typescript
// openkit.config.ts
{
  type: 'markdown',
  outFile: './docs/admin-api.md',
  enrich: {
    descriptions: true,   // fill missing operation descriptions
    examples: true,        // generate request/response examples
    model: 'claude-sonnet-4-6',
    // batch: true — use Anthropic batch API for cost efficiency on large specs
  },
}
```

Enrichment responses are cached to `.openkit-cache/` by a hash of each operation object. Re-running against an unchanged spec makes zero API calls.

### 4. Programmatic composition

The low-level API is exposed so consumers can use parts of the pipeline without adopting the whole tool:

```typescript
import { parseSpec, filterSpec, transformers } from '@yourscope/openkit'

const spec = await parseSpec('./openapi.json')

// filter to admin section only
const filtered = await filterSpec(spec, {
  tags: ['admin'],
  removeUnusedComponents: true,
})

// run multiple transformers on the same filtered spec
const [markdown, types] = await Promise.all([
  transformers.markdown(filtered),
  transformers['ts-types'](filtered),
])
```

---

## Roadmap

### v0.1 — Core CLI

Target: 4 weeks from start

- `openkit generate` with `--output markdown` and full filter support (tags, paths, methods, flags)
- `openkit filter` — outputs filtered OpenAPI JSON/YAML
- `openkit validate` — wrapper over swagger-parser
- `openkit.config.ts` auto-discovery
- Published to npm as `@yourscope/openkit`
- TypeScript, ESM + CJS dual build via `tsup`

### v0.2 — Multi-output types

Target: +4 weeks

- `--output ts-types` via `openapi-typescript`
- `--output zod` via `openapi-zod-client`
- `--output sdk` via `@hey-api/openapi-ts`
- Multiple output types in a single run: `--output markdown,ts-types`
- `prettier` post-processing on all output types
- Custom `eta` template support for markdown

### v1.0 — DX polish & public release

Target: +3 weeks

- `--watch` mode with debounced re-generation on spec file changes
- `openkit diff` — human-readable changelog between two spec versions
- `--interactive` wizard mode via `inquirer`
- Published to JSR alongside npm
- GitHub Actions example workflow in README
- Full documentation site

### v2.x — AI enrichment

Timeline: TBD

- `openkit enrich` command
- Per-operation AI description generation (fill missing fields)
- Request/response example generation per endpoint
- Changelog generation from spec diff
- Anthropic batch API support for cost-efficient bulk enrichment
- MCP server exposure — allows AI agents to call `openkit.generate()` as a tool

---

## Open questions

| # | Question | Decision needed by |
|---|---|---|
| 1 | Package name: `openkit` vs `apigen` vs `specout`? | Before npm publish |
| 2 | Filter config: glob patterns (`/admin/**`) or regex only? | v0.1 |
| 3 | Zod output: full Zodios client or bare schemas only by default? | v0.2 |
| 4 | AI cache (`.openkit-cache/`): gitignored by default or checked in? | v2 |
| 5 | Should remote URL input support custom auth headers (Bearer, API key)? | v0.2 |
