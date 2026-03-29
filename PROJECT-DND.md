# R&D Specification — `openkit`
### OpenAPI Transformation Toolkit

**Status:** Draft  
**Version:** 0.1  
**Author:** [Your name]  
**Last updated:** 2026-03

---

## Table of contentsJEC

1. [Executive summary](#1-executive-summary)
2. [Problem statement](#2-problem-statement)
3. [Goals and non-goals](#3-goals-and-non-goals)
4. [Architecture overview](#4-architecture-overview)
5. [Transformation pipeline design](#5-transformation-pipeline-design)
6. [Filter engine](#6-filter-engine)
7. [Output modules](#7-output-modules)
8. [CLI design](#8-cli-design)
9. [Programmatic API design](#9-programmatic-api-design)
10. [Configuration file](#10-configuration-file)
11. [Package selection rationale](#11-package-selection-rationale)
12. [AI enrichment layer (future)](#12-ai-enrichment-layer-future)
13. [Roadmap](#13-roadmap)
14. [Technical risks](#14-technical-risks)
15. [Open questions](#15-open-questions)

---

## 1. Executive summary

`openkit` is a composable CLI and Node.js library that transforms OpenAPI specifications into multiple useful outputs — documentation, TypeScript types, Zod schemas, and SDK clients — with first-class support for partial extraction (filtering by tags, paths, or methods). It is designed to be extended incrementally, ship as an npm/JSR package, and eventually support AI-powered enrichment of the generated output.

The project starts from a proven single use case (OpenAPI → Markdown via `@scalar/openapi-to-markdown`) and builds a plugin-style pipeline around it.

---

## 2. Problem statement

**Current state:** A short script converts an entire `openapi.json` to a single markdown file using `@scalar/openapi-to-markdown`. This works, but has four meaningful gaps:

1. **No filtering.** A large spec (e.g. 200+ endpoints across multiple service domains) produces one monolithic document. An "admin" team or an "external partner" only needs a subset.

2. **Single output type.** The same spec needs to produce TypeScript types for the frontend, Zod schemas for runtime validation, and markdown for documentation. These are separate ad-hoc scripts today.

3. **No reuse surface.** Logic is not packaged for use in CI pipelines, other repos, or by external contributors.

4. **No extension points.** Adding future capabilities (custom templates, AI-enriched descriptions, schema diffing) requires rewriting the script from scratch rather than composing on top of a stable core.

---

## 3. Goals and non-goals

### Goals

- Filter any OpenAPI spec to a subset by tags, path patterns, HTTP methods, or custom extension flags before any transformation.
- Produce at least four output types from the same pipeline: markdown, TypeScript interfaces, Zod schemas, SDK/fetch client.
- Ship as both a CLI (`npx openkit`) and a programmatic Node.js API.
- Accept a config file (`openkit.config.ts`) so CI usage requires zero flags.
- Be extensible: new output types are added as modules without touching the core pipeline.
- Support OpenAPI 3.0 and 3.1. (Swagger 2.0 is a nice-to-have, not a requirement.)

### Non-goals

- Generating or hosting interactive API docs (that is Swagger UI / Scalar's job).
- Validating that an API implementation conforms to its spec (that is contract testing).
- Replacing full SDK generation platforms like Speakeasy or Stainless.
- Supporting non-OpenAPI specs (GraphQL, gRPC, AsyncAPI) in v1.

---

## 4. Architecture overview

The system is a linear pipeline with five layers:

```
Input → Parser/Deref → Filter engine → Transformer(s) → Post-processor → Output
```

Each layer is independently replaceable. The CLI and the programmatic API are both thin consumers of the same pipeline.

### Layer responsibilities

| Layer | Responsibility | Key packages |
|---|---|---|
| Input | Resolve file path, URL, or stdin to a raw JS object | `fs`, `node-fetch` |
| Parser / deref | Validate spec, dereference all `$ref` | `@apidevtools/swagger-parser` |
| Filter engine | Reduce spec to a relevant subset | `openapi-filter`, `openapi-format` |
| Transformer | Convert the filtered spec to a target output | (per module — see §7) |
| Post-processor | Format and prettify output | `prettier`, custom templates |
| Output | Write to file, stdout, or return string | `fs`, `process.stdout` |

---

## 5. Transformation pipeline design

### Core contract

Every transformer is a pure async function:

```typescript
type Transformer<TOptions = Record<string, unknown>> = (
  spec: OpenAPIObject,  // already parsed + dereffed
  options?: TOptions
) => Promise<string | Record<string, string>>
// string = single file output
// Record<string, string> = multi-file output (path → content)
```

The pipeline runner resolves the correct transformer by the `--output` flag or `output` config key, passes the filtered spec, and writes the result.

### Why this contract matters

- The filter engine runs once; all transformer modules are downstream of it.
- Adding a new output type means implementing one function and registering it — no changes to the pipeline.
- The same spec object can be piped to multiple transformers in one run (e.g. `output: ['markdown', 'ts-types']`).

### Pipeline execution sketch

```typescript
// Illustrative — not final API
import { parse } from './parser'
import { filter } from './filter'
import { run } from './pipeline'

const spec = await parse('./openapi.json')
const filtered = await filter(spec, { tags: ['admin'] })
const results = await run(filtered, {
  outputs: ['markdown', 'ts-types'],
  outDir: './docs'
})
```

---

## 6. Filter engine

Filtering is the most important differentiator of this tool. The filter layer sits before every transformer so any output type can be produced from any subset of the API.

### Filtering dimensions

| Dimension | Example | Implementation |
|---|---|---|
| By tag | `--tags admin,billing` | `openapi-filter` with `--checkTags` |
| By path pattern | `--paths "/admin/.*"` | `openapi-format` filterOperations regex |
| By HTTP method | `--methods GET,POST` | `openapi-format` filterMethods |
| By custom flag | `--flags x-internal` | `openapi-filter` flag matching |
| By operation ID | `--operations listUsers,createUser` | `openapi-format` filterOperationIds |
| Inverse mode | `--inverse` | `openapi-filter --inverse` (keep only matching) |

### Unused component pruning

After path/operation filtering, `$components/schemas` will contain schemas no longer referenced by any remaining operation. The filter stage must clean these up. `openapi-format` handles this natively via `--removeUnusedComponents`.

### Filter config example

```typescript
// openkit.config.ts
export default {
  input: './openapi.json',
  filter: {
    tags: ['admin'],
    paths: ['/admin/**'],
    methods: ['GET', 'POST', 'PATCH', 'DELETE'],
    removeUnusedComponents: true,
  },
  outputs: [
    { type: 'markdown', outFile: './docs/admin-api.md' },
    { type: 'ts-types', outFile: './src/types/admin.d.ts' },
  ],
}
```

### Reference: openapi-filter flag-based filtering

For organisations that want to annotate the spec directly instead of passing CLI flags, add `x-internal: true` to any path, operation, or schema in the spec and run:

```bash
openapi-filter --flags x-internal -- openapi.json openapi.public.json
```

This removes every annotated element and produces a clean public spec.

---

## 7. Output modules

### 7.1 Markdown

**Package:** `@scalar/openapi-to-markdown`  
**Status:** Already in use  
**Output:** A single `.md` file

This is the existing use case. In the pipeline, it becomes the first registered transformer. No changes to the calling code — the filter layer now wraps it.

Template override: for organisations that need a custom markdown layout (e.g. section ordering, frontmatter, internal branding), an `eta` template can be supplied via `--template ./my-template.eta`. The spec object is passed as template data.

### 7.2 TypeScript types

**Package:** `openapi-typescript`  
**Status:** Planned (v0.2)  
**Output:** A single `.d.ts` file

`openapi-typescript` generates zero-runtime TypeScript types directly from an OpenAPI 3.x spec. It is MIT-licensed, requires no Java, runs in milliseconds, and produces types that work with `openapi-fetch` for fully type-safe HTTP calls.

```typescript
// Generated type usage example
import type { paths } from './admin-types'
import createClient from 'openapi-fetch'

const client = createClient<paths>({ baseUrl: 'https://api.example.com' })

const { data } = await client.GET('/admin/users/{id}', {
  params: { path: { id: 42 } }
})
```

Custom date handling (e.g. `format: date-time` → `Date` instead of `string`) is supported via the `transform` option.

### 7.3 Zod schemas

**Package:** `openapi-zod-client`  
**Status:** Planned (v0.2)  
**Output:** A `.ts` file exporting named Zod schemas

`openapi-zod-client` generates a Zodios client (runtime-validated HTTP client) from any OpenAPI spec. It also supports Handlebars templates if only the bare Zod schemas are needed without the client wrapper.

For teams that use `orval`, the same filter output can be passed to `orval` programmatically via its Node.js API — `orval` supports a `zod` client mode that generates standalone schemas.

```typescript
// openkit.config.ts — Zod output
{
  type: 'zod',
  outFile: './src/schemas/admin.ts',
  options: {
    // only export schemas, not the full Zodios client
    exportSchemas: true,
  }
}
```

### 7.4 SDK / fetch client

**Package:** `@hey-api/openapi-ts`  
**Status:** Planned (v0.2)  
**Output:** A directory of `.ts` files (types, services, client)

`@hey-api/openapi-ts` (used by Vercel, PayPal, OpenCode) is a plugin-based codegen tool. It generates SDK clients, Zod schemas, and TanStack Query hooks from the same spec. It is the heaviest of the output modules and is optional — suited for teams that want a publishable SDK package rather than just types.

The plugin system allows targeting Fetch, Axios, and Angular clients. Output is a directory, not a single file.

### 7.5 AI enrichment (future, v2)

See §12.

---

## 8. CLI design

### Tool name

`openkit` — short, memorable, unambiguous.  
Published as: `@yourscope/openkit` on npm and JSR.  
Invocation: `npx @yourscope/openkit` or globally `npm i -g @yourscope/openkit`.

### Command structure

```
openkit <command> [options]

Commands:
  generate   Transform a spec to one or more output types
  filter     Extract a filtered spec subset (outputs OpenAPI JSON/YAML)
  validate   Validate a spec against the OpenAPI 3.x schema
  diff       Compare two spec versions and summarise changes (v1+)
  enrich     AI-powered description and example generation (v2)
```

### Core command: `generate`

```bash
openkit generate ./openapi.json \
  --output markdown \
  --tags admin \
  --out ./docs/admin-api.md

# Multiple outputs in one pass
openkit generate ./openapi.json \
  --output markdown,ts-types \
  --tags admin \
  --out-dir ./generated

# From URL
openkit generate https://api.example.com/openapi.json \
  --output zod \
  --paths "/v2/.*" \
  --out ./src/schemas.ts

# Config-driven (zero flags)
openkit generate
# reads openkit.config.ts in the project root
```

### Core command: `filter`

The filter command outputs a valid OpenAPI document, not a transformed output. This is useful for piping into other tools (Swagger UI, Postman, etc.) or into `generate` in a two-step pipeline.

```bash
openkit filter ./openapi.json \
  --tags admin \
  --out ./openapi.admin.json

# Inverse: keep only flagged endpoints
openkit filter ./openapi.json \
  --flags x-public \
  --inverse \
  --out ./openapi.public.json
```

### CLI framework choice: commander.js

Commander is chosen over Yargs and Oclif for three reasons:

1. **Size:** ~180KB, zero runtime dependencies. Oclif is 12MB+.
2. **Startup time:** ~18ms overhead vs 85ms for Oclif — important for `npx` invocations.
3. **Simplicity:** A tool of this scope (5-6 subcommands, no plugin system needed) does not benefit from Oclif's overhead.

If the tool later needs a plugin architecture (e.g. third-party transformer plugins), migration to Oclif is straightforward — the command handler logic is framework-agnostic.

---

## 9. Programmatic API design

The library exposes a clean Node.js API so the CLI is just a thin wrapper:

```typescript
import { openkit } from '@yourscope/openkit'

// High-level: config-driven
const results = await openkit({
  input: './openapi.json',
  filter: { tags: ['admin'] },
  outputs: [
    { type: 'markdown', outFile: './docs/admin.md' },
    { type: 'ts-types', outFile: './src/types/admin.d.ts' },
  ],
})

// Low-level: compose manually
import { parseSpec, filterSpec, transformers } from '@yourscope/openkit'

const spec = await parseSpec('./openapi.json')
const filtered = await filterSpec(spec, { tags: ['admin'] })
const markdown = await transformers.markdown(filtered)
const types = await transformers.tsTypes(filtered)
```

The low-level API is intentionally exposed because it lets consumers compose parts of the pipeline without adopting the whole tool.

---

## 10. Configuration file

The config file follows the same pattern as `@hey-api/openapi-ts`, `eslint`, and `vitest` — a TypeScript file in the project root, auto-discovered.

```typescript
// openkit.config.ts
import { defineConfig } from '@yourscope/openkit'

export default defineConfig({
  input: './openapi.json',    // or URL, or function returning OpenAPIObject

  filter: {
    tags: ['admin'],
    removeUnusedComponents: true,
  },

  outputs: [
    {
      type: 'markdown',
      outFile: './docs/admin-api.md',
      // optional: custom eta template
      template: './templates/api-docs.eta',
    },
    {
      type: 'ts-types',
      outFile: './src/generated/admin.d.ts',
      options: {
        // openapi-typescript transform option
        transform: (schema) => {
          if (schema.format === 'date-time') return 'Date'
        },
      },
    },
    {
      type: 'zod',
      outFile: './src/generated/admin-schemas.ts',
    },
  ],

  // Future: watch mode
  watch: process.env.NODE_ENV === 'development',
})
```

---

## 11. Package selection rationale

### Parsing

**`@apidevtools/swagger-parser`** is the de facto standard for OpenAPI validation and dereferencing. It supports OpenAPI 2 / 3.0 / 3.1 and resolves all `$ref` pointers (local, remote, file). This is a non-negotiable dependency — every downstream transformer needs a fully dereferenced spec object.

### Filtering

Two packages cover complementary needs:

- **`openapi-filter`** (by Mermade) handles flag-based filtering (`x-internal`, custom flags) and the `--inverse` mode. It is the right tool when the source spec is annotated with extension flags.
- **`openapi-format`** handles structural filtering (by tag, path regex, method, operationID) and also handles unused component cleanup. It is the right tool when filtering criteria come from outside the spec (e.g. a CLI flag or config file).

These two packages are not competing — they solve different parts of the filtering problem and compose cleanly.

### TypeScript types

**`openapi-typescript`** is chosen over `@hey-api/openapi-ts` for the types-only output mode because it is focused, fast (milliseconds for large specs), and has zero runtime overhead. The generated `.d.ts` types work with `openapi-fetch` or any other HTTP client.

`@hey-api/openapi-ts` is reserved for the SDK output mode because its plugin system is better suited to generating a full client package (services, schemas, hooks) rather than just type declarations.

Note: `openapi-typescript-codegen` (Ferdi Koomen's original) was deprecated in May 2024 — **do not use it**. `@hey-api/openapi-ts` is the maintained fork.

### Zod schemas

**`openapi-zod-client`** is the primary choice. It is programmable, supports Handlebars templates for custom output, and can target schemas-only or the full Zodios client. `orval` is listed as an alternative for teams that also need React Query or SWR hooks generated alongside schemas.

### CLI framework

**`commander`** at ~35M weekly downloads, 180KB, and zero dependencies is the right choice for a CLI of this scope. Startup latency matters for `npx` — commander adds ~18ms, oclif adds ~85-120ms. For future plugin support, oclif can be adopted in v2.

### Template engine

**`eta`** (2KB) is chosen over Handlebars for custom markdown templates. It is TypeScript-native, has no dependencies, and supports async partials — useful for future AI enrichment callbacks inside templates.

### Post-processing

**`prettier`** handles all output formatting. It supports markdown, TypeScript, and JSON natively. A single `.prettierrc` in the consumer's project will govern all openkit output automatically.

---

## 12. AI enrichment layer (future)

Planned for v2. The AI layer is an optional post-processing step — it does not replace any transformer, it augments the output of existing ones.

### Use cases

1. **Description enrichment:** Many OpenAPI specs have sparse or missing `description` fields on operations and schemas. The AI layer can generate natural language descriptions based on the operation name, parameters, and response schema.

2. **Usage examples:** For markdown output, the AI layer can generate realistic request/response examples per endpoint.

3. **Changelog generation:** Given two versions of a spec (old + new), produce a human-readable changelog: "POST /admin/users now requires `role` in the request body. GET /admin/reports was removed."

4. **MCP server exposure:** The toolkit itself can be exposed as an MCP tool — allowing an AI agent to call `openkit.generate({ tags: ['admin'] })` as a tool call inside an agentic workflow.

### Implementation sketch

```typescript
// openkit.config.ts — AI enrichment
import { defineConfig } from '@yourscope/openkit'
import Anthropic from '@anthropic-ai/sdk'

export default defineConfig({
  input: './openapi.json',
  filter: { tags: ['admin'] },
  outputs: [
    {
      type: 'markdown',
      outFile: './docs/admin-api.md',
      enrich: {
        descriptions: true,
        examples: true,
        model: 'claude-sonnet-4-6',
        // batch: true — use Anthropic batch API for cost efficiency
      },
    },
  ],
})
```

The enrichment step calls the Anthropic API per-operation and merges the AI-generated content into the spec object before the markdown transformer runs. Responses are cached to a `.openkit-cache/` directory by a hash of the operation object — re-running with an unchanged spec makes zero API calls.

### Cost management

- Cache responses by operation hash (only re-call on spec changes).
- Use `claude-haiku` for bulk description generation, `claude-sonnet` for examples.
- Support Anthropic batch API for large specs (>50 operations) to reduce cost by ~50%.
- Expose `--dry-run` flag that shows which operations would be enriched and estimated token usage.

---

## 13. Roadmap

### v0.1 — Core CLI (target: 4 weeks)

- `openkit generate` with `--output markdown` and full filter support
- `openkit filter` (outputs filtered OpenAPI JSON/YAML)
- `openkit validate` (wrapper over swagger-parser validation)
- `openkit.config.ts` support
- Published to npm as `@yourscope/openkit`
- TypeScript, ESM + CJS dual build

### v0.2 — Multi-output types (target: +4 weeks)

- `--output ts-types` via `openapi-typescript`
- `--output zod` via `openapi-zod-client`
- `--output sdk` via `@hey-api/openapi-ts`
- Multiple output types in a single run
- `prettier` post-processing
- Custom `eta` template support for markdown

### v1.0 — DX & distribution polish (target: +3 weeks)

- `--watch` mode with debounced re-generation
- `openkit diff` command (schema version comparison)
- `--interactive` mode with `inquirer` prompts
- JSR package alongside npm
- GitHub Actions example workflow
- Full documentation site

### v2.x — AI enrichment (future)

- `openkit enrich` command
- Per-operation AI description generation
- Request/response example generation
- Changelog generation from spec diff
- MCP server exposure

---

## 14. Technical risks

### $ref resolution and circular references

Complex specs with deep circular `$ref` chains can cause issues with `swagger-parser`. The mitigation is to expose a `--dereference: 'safe'` option that uses `json-schema-ref-parser`'s circular reference handling as a fallback.

### Filter producing invalid specs

After aggressive filtering, a spec may reference schemas in `$components` that were removed, or have `allOf`/`oneOf` references that break. `openapi-format --removeUnusedComponents` handles most cases, but an explicit validation step after filtering (using swagger-parser again) is needed before passing to transformers.

### `openapi-typescript` output size with large specs

For specs with hundreds of schemas, the generated `.d.ts` file can be very large. The filter-first approach mitigates this — only filtered schemas are typed. Additionally, `openapi-typescript`'s `--path-params-as-types` option can be considered to reduce redundancy.

### Multiple transformer outputs vs atomic writes

When multiple output types are generated in one run, a failure partway through leaves the output directory in a partially-updated state. Use a write-to-temp-then-rename strategy: write all outputs to a temp directory, then atomically move on success. On failure, report the error and leave the previous outputs untouched.

### Commander vs Oclif migration risk

If the tool grows a plugin system, commander's flat structure becomes limiting. This is not a v1 concern, but the transformer registry (§5) is designed so that third-party transformers can be registered without any CLI changes — only the transformer map needs updating.

---

## 15. Open questions

| # | Question | Owner | Decision needed by |
|---|---|---|---|
| 1 | Package name: `openkit` vs `apigen` vs `specout`? | You | Before npm publish |
| 2 | Should filter config support glob patterns (e.g. `paths: ['/admin/**']`) or only regex? | — | v0.1 |
| 3 | Should the Zod output include a full Zodios client or bare schemas only? | — | v0.2 |
| 4 | Cache AI enrichment results in `.openkit-cache/` — should this be gitignored by default or checked in? | — | v2 |
| 5 | Should the CLI support reading the spec from a remote URL that requires auth headers? | — | v0.2 |

---

## Appendix: directory structure (proposed)

```
packages/
  openkit/
    src/
      cli.ts              ← commander entrypoint
      pipeline.ts         ← core pipeline runner
      parser.ts           ← parse + deref
      filter.ts           ← filter engine
      config.ts           ← config file loader (jiti)
      transformers/
        markdown.ts
        ts-types.ts
        zod.ts
        sdk.ts
        ai-enrich.ts      ← future
      post-process.ts     ← prettier wrapper
      utils/
        write.ts          ← atomic file write
        cache.ts          ← AI response cache
    bin/
      openkit.js          ← shebang entrypoint
    openkit.config.ts     ← example config for this repo's own API
    package.json
    tsconfig.json
    README.md
```

---

*This document is a living R&D spec. Update version and status header when phases are completed.*
