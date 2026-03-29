# OpenKit

A composable CLI and Node.js library that transforms OpenAPI specifications into multiple useful outputs — documentation, TypeScript types, Zod schemas, and SDK clients — with first-class support for filtering by tags, paths, or methods.

[![CI](https://github.com/yeasin2002/openkit/actions/workflows/ci.yml/badge.svg)](https://github.com/yeasin2002/openkit/actions/workflows/ci.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

## ⭐ Show Your Support

Give a ⭐️ if this project helped you transform OpenAPI specs efficiently!

## ✨ Features

### Build & Type System

- **📦 TypeScript 5.9+** - Proper TypeScript support
- **⚡ tsdown** - Rust based Lightning-fast builds
- **🔍 Export Validation** - Ensure package exports work correctly with [@arethetypeswrong/cli](https://github.com/arethetypeswrong/arethetypeswrong.github.io)
- **📚 Dual Module Format** - Full CommonJS and ESM support for maximum compatibility

### Testing & Quality

- **🧪 Vitest**
- **🎨 Prettier** - Consistent code formatting with all necessary rules
- **🔧 ESLint** - TypeScript-aware linting with type-checked rules
- **📏 size-limit** - Monitor and control bundle size

### Automation & Workflow

- **🪝 Husky + lint-staged** - Pre-commit hooks for automatic formatting and linting
- **✅ Commitlint** - Enforce conventional commits for better changelogs
- **📝 Changesets** - Automated version management and changelog generation
- **🤖 GitHub Actions** - Complete CI/CD pipeline for testing and releases
- **🔄 Dependabot** - Weekly automated dependency updates with proper grouping

### Documentation & Developer Experience

- **📖 TypeDoc** - Auto-generated API documentation from JSDoc comments
- **🐛 VS Code Integration** - Debug configurations and recommended extensions
- **🔒 Security Audits** - Automated dependency scanning

## 🚀 Quick Start

### Installation

```bash
# Use with npx (recommended)
npx openkit generate ./openapi.json --output markdown --out ./docs/api.md

# Or install globally
npm install -g openkit
openkit generate ./openapi.json --output markdown --out ./docs/api.md

# Or install as dev dependency
npm install -D openkit
```

### Basic Usage

```bash
# Generate markdown documentation
openkit generate ./openapi.json --output markdown --out ./docs/api.md

# Filter by tags and generate TypeScript types
openkit generate ./openapi.json --output ts-types --tags admin --out ./src/types/admin.d.ts

# Multiple outputs from filtered spec
openkit generate ./openapi.json \
  --output markdown,ts-types,zod \
  --tags admin,billing \
  --out-dir ./generated

# From remote URL
openkit generate https://api.example.com/openapi.json \
  --output markdown \
  --paths "/v2/.*" \
  --out ./docs/v2-api.md

# Config-driven (zero flags)
openkit generate  # reads openkit.config.ts
```

### Configuration File

Create `openkit.config.ts` in your project root:

```typescript
import { defineConfig } from 'openkit'

export default defineConfig({
  input: './openapi.json',
  
  filter: {
    tags: ['admin'],
    removeUnusedComponents: true,
  },
  
  outputs: [
    {
      type: 'markdown',
      outFile: './docs/admin-api.md',
      template: './templates/api-docs.eta', // optional custom template
    },
    {
      type: 'ts-types',
      outFile: './src/types/admin.d.ts',
    },
    {
      type: 'zod',
      outFile: './src/schemas/admin.ts',
    },
  ],
})
```

## 📋 Available Commands

### Core Commands

| Command                    | Description                                    |
| -------------------------- | ---------------------------------------------- |
| `openkit generate`         | Transform spec to one or more output types    |
| `openkit filter`           | Extract filtered spec subset (outputs JSON)   |
| `openkit validate`         | Validate spec against OpenAPI 3.x schema      |
| `openkit diff`             | Compare two spec versions (v1.0+)             |
| `openkit enrich`           | AI-powered description generation (v2.0+)     |

### Generate Command Options

| Option                     | Description                                    |
| -------------------------- | ---------------------------------------------- |
| `--output <types>`         | Output types: `markdown`, `ts-types`, `zod`, `sdk` |
| `--tags <tags>`            | Filter by tags (comma-separated)              |
| `--paths <patterns>`       | Filter by path regex patterns                  |
| `--methods <methods>`      | Filter by HTTP methods                         |
| `--flags <flags>`          | Filter by custom extension flags              |
| `--inverse`                | Inverse filtering (exclude matches)           |
| `--out <file>`             | Output file path                               |
| `--out-dir <dir>`          | Output directory for multiple files           |
| `--template <file>`        | Custom Eta template for markdown              |

### Filter Command Options

| Option                     | Description                                    |
| -------------------------- | ---------------------------------------------- |
| `--tags <tags>`            | Filter by tags                                 |
| `--paths <patterns>`       | Filter by path patterns                        |
| `--methods <methods>`      | Filter by HTTP methods                         |
| `--flags <flags>`          | Filter by extension flags                      |
| `--inverse`                | Keep only matching operations                  |
| `--out <file>`             | Output filtered spec file                      |

## 🏗️ Architecture

OpenKit uses a linear pipeline architecture where each stage is independently replaceable:

```
Input → Parser/Deref → Filter Engine → Transformer(s) → Post-processor → Output
```

### Pipeline Layers

| Layer | Responsibility | Key Packages |
|-------|----------------|--------------|
| **Input** | Resolve file path, URL, or stdin to raw JS object | `fs`, `node-fetch` |
| **Parser/Deref** | Validate spec, dereference all `$ref` | `@apidevtools/swagger-parser` |
| **Filter Engine** | Reduce spec to relevant subset | `openapi-filter`, `openapi-format` |
| **Transformer** | Convert filtered spec to target output | per module |
| **Post-processor** | Format and prettify output | `prettier`, `eta` templates |
| **Output** | Write to file, stdout, or return string | `fs`, `process.stdout` |

### Transformer Contract

Every transformer is a pure async function:

```typescript
type Transformer<TOptions = Record<string, unknown>> = (
  spec: OpenAPIObject,  // already parsed + dereferenced
  options?: TOptions
) => Promise<string | Record<string, string>>
// string = single file output
// Record<string, string> = multi-file output (path → content)
```

### Extensibility

Adding new output types requires only implementing one function and registering it:

```typescript
// src/transformers/my-output.ts
export const myOutput: Transformer = async (spec, options) => {
  return generateSomethingFrom(spec)
}

// Register in transformer map - no pipeline changes needed
```

## 🔧 Output Types

### Markdown Documentation

**Package:** `@scalar/openapi-to-markdown`  
**Output:** Single `.md` file with complete API documentation

```bash
openkit generate ./openapi.json --output markdown --out ./docs/api.md
```

Custom templates supported via Eta templating engine:

```typescript
// openkit.config.ts
{
  type: 'markdown',
  outFile: './docs/api.md',
  template: './templates/custom-api-docs.eta'
}
```

### TypeScript Types

**Package:** `openapi-typescript`  
**Output:** Single `.d.ts` file with zero-runtime types

```bash
openkit generate ./openapi.json --output ts-types --out ./src/types/api.d.ts
```

Works seamlessly with `openapi-fetch` for type-safe HTTP clients:

```typescript
import type { paths } from './types/api'
import createClient from 'openapi-fetch'

const client = createClient<paths>({ baseUrl: 'https://api.example.com' })
const { data } = await client.GET('/users/{id}', {
  params: { path: { id: 42 } }
})
```

### Zod Schemas

**Package:** `openapi-zod-client`  
**Output:** `.ts` file with runtime validation schemas

```bash
openkit generate ./openapi.json --output zod --out ./src/schemas/api.ts
```

Generates Zod schemas for runtime validation:

```typescript
import { UserSchema } from './schemas/api'

const user = UserSchema.parse(userData) // Runtime validation
```

### SDK Generation

**Package:** `@hey-api/openapi-ts`  
**Output:** Directory with complete SDK (types, services, client)

```bash
openkit generate ./openapi.json --output sdk --out-dir ./src/generated
```

Generates full-featured SDK with multiple client options (Fetch, Axios, Angular).

## 🎯 Filtering Examples

### Filter by Tags

Extract only admin and billing endpoints:

```bash
openkit generate ./openapi.json \
  --output markdown \
  --tags admin,billing \
  --out ./docs/admin-billing-api.md
```

### Filter by Path Patterns

Include only v2 API endpoints:

```bash
openkit generate ./openapi.json \
  --output ts-types \
  --paths "/v2/.*" \
  --out ./src/types/v2-api.d.ts
```

### Filter by HTTP Methods

Generate docs for read-only operations:

```bash
openkit generate ./openapi.json \
  --output markdown \
  --methods GET \
  --out ./docs/read-only-api.md
```

### Filter by Custom Flags

Remove internal endpoints (requires `x-internal: true` in spec):

```bash
openkit filter ./openapi.json \
  --flags x-internal \
  --inverse \
  --out ./openapi.public.json
```

### Complex Filtering

Combine multiple filters:

```bash
openkit generate ./openapi.json \
  --output markdown,ts-types \
  --tags admin \
  --paths "/admin/.*" \
  --methods GET,POST,PATCH,DELETE \
  --out-dir ./generated/admin
```

## 🛠️ Programmatic API

OpenKit exposes both high-level and low-level APIs for programmatic usage:

### High-level API (Config-driven)

```typescript
import { openkit } from 'openkit'

const results = await openkit({
  input: './openapi.json',
  filter: { tags: ['admin'] },
  outputs: [
    { type: 'markdown', outFile: './docs/admin.md' },
    { type: 'ts-types', outFile: './src/types/admin.d.ts' },
  ],
})
```

### Low-level API (Composable)

```typescript
import { parseSpec, filterSpec, transformers } from 'openkit'

const spec = await parseSpec('./openapi.json')
const filtered = await filterSpec(spec, { tags: ['admin'] })

// Run multiple transformers on the same filtered spec
const [markdown, types] = await Promise.all([
  transformers.markdown(filtered),
  transformers.tsTypes(filtered),
])
```

### Custom Transformers

```typescript
import type { Transformer } from 'openkit'

const myTransformer: Transformer = async (spec, options) => {
  // Transform the OpenAPI spec to your custom format
  return generateCustomOutput(spec)
}

// Register and use
transformers.register('my-format', myTransformer)
```

## 🗺️ Roadmap

### v0.1 — Core CLI *(Current)*
- ✅ `openkit generate` with markdown output
- ✅ Full filter support (tags, paths, methods, flags)
- ✅ `openkit filter` command
- ✅ `openkit validate` command
- ✅ `openkit.config.ts` support
- ✅ TypeScript, ESM + CJS dual build

### v0.2 — Multi-output Types *(Next)*
- 🔄 `--output ts-types` via `openapi-typescript`
- 🔄 `--output zod` via `openapi-zod-client`
- 🔄 `--output sdk` via `@hey-api/openapi-ts`
- 🔄 Multiple output types in single run
- 🔄 `prettier` post-processing
- 🔄 Custom `eta` template support

### v1.0 — DX Polish
- 📋 `--watch` mode with debounced re-generation
- 📋 `openkit diff` command (spec version comparison)
- 📋 `--interactive` wizard mode
- 📋 JSR package alongside npm
- 📋 GitHub Actions workflow examples

### v2.x — AI Enrichment
- 🔮 `openkit enrich` command
- 🔮 AI-powered description generation
- 🔮 Request/response example generation
- 🔮 Changelog generation from spec diff
- 🔮 MCP server exposure for AI agents

## 📄 License

MIT License - see the [LICENSE](LICENSE) file for details.

## 👤 Author

**Md Kawsar Islam Yeasin**

- Email: [mdkawsarislam2002@gmail.com](mailto:mdkawsarislam2002@gmail.com)
- Website: [yeasin2002.vercel.app](https://yeasin2002.vercel.app/)
- GitHub: [@yeasin2002](https://github.com/yeasin2002)

## 🤝 Contributing

Contributions, issues, and feature requests are welcome!

1. Fork the repository
2. Create your feature branch (`git checkout -b feat/amazing-feature`)
3. Commit your changes (`git commit -m 'feat: add amazing feature'`)
4. Push to the branch (`git push origin feat/amazing-feature`)
5. Open a Pull Request

See the [issues page](https://github.com/yeasin2002/openkit/issues) for known issues and feature requests.

## 📚 Resources

- [OpenAPI Specification](https://spec.openapis.org/oas/v3.1.0)
- [OpenAPI TypeScript Generator](https://github.com/drwpow/openapi-typescript)
- [Scalar OpenAPI to Markdown](https://github.com/scalar/scalar/tree/main/packages/openapi-to-markdown)
- [OpenAPI Zod Client](https://github.com/astahmer/openapi-zod-client)
- [Hey API OpenAPI TS](https://github.com/hey-api/openapi-ts)
- [OpenAPI Filter](https://github.com/Mermade/openapi-filter)
- [OpenAPI Format](https://github.com/thim81/openapi-format)
