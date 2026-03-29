# OpenKit Agent Guidelines

This document provides comprehensive guidance for AI agents working on the OpenKit project - a composable CLI and Node.js library for transforming OpenAPI specifications into multiple output formats.

## Project Overview

OpenKit is an OpenAPI transformation toolkit that converts OpenAPI specifications into documentation, TypeScript types, Zod schemas, and SDK clients with advanced filtering capabilities.

### Core Architecture

```
Input → Parser/Deref → Filter Engine → Transformer(s) → Post-processor → Output
```

**Key Principles:**

- Linear pipeline where each stage is independently replaceable
- Filter-first approach - filtering happens once before all transformers
- Extensible transformer system for adding new output types
- Both CLI and programmatic API support

## Development Guidelines

### Code Organization

**Source Structure:**

```
src/
├── cli.ts              # Commander.js CLI entrypoint
├── pipeline.ts         # Core pipeline runner
├── parser.ts           # OpenAPI parsing and dereferencing
├── filter.ts           # Filtering engine
├── config.ts           # Configuration file loader
├── transformers/       # Output format transformers
│   ├── markdown.ts     # Markdown documentation
│   ├── ts-types.ts     # TypeScript types
│   ├── zod.ts          # Zod schemas
│   └── sdk.ts          # SDK generation
└── utils/              # Shared utilities
```

**Key Files:**

- `src/index.ts` - Main export file, re-exports all public APIs
- `openkit.config.ts` - Configuration file example
- `bin/openkit.js` - CLI executable shebang entrypoint

### Transformer Contract

Every transformer must fs = Record<string, unknown>> = (
spec: OpenAPIObject, // already parsed + dereferenced
options?: TOptions
) => Promise<string | Record<string, string>>
// string = single file output
// Record<string, string> = multi-file output (path → content)

````

**Adding New Transformers:**
1. Create transformer function in `src/transformers/`
2. Register in transformer map - no pipeline changes needed
3. Transformer receives pre-filtered spec object

### Technology Stack

**Core Dependencies:**
- `@apidevtools/swagger-parser` - OpenAPI parsing and dereferencing (non-negotiable)
- `openapi-filter` - Flag-based filtering (`x-internal`, custom flags)
- `openapi-format` - Structural filtering (tags, paths, methods)
- `commander` - CLI framework (fast startup for npx usage)
- `prettier` - Output formatting

**Output Packages:**
- `@scalar/openapi-to-markdown` - Markdown generation
- `openapi-typescript` - TypeScript types (zero-runtime)
- `openapi-zod-client` - Zod schemas and Zodios client
- `@hey-api/openapi-ts` - Full SDK generation

**Template Engine:**
- `eta` - 2KB TypeScript-native template engine for custom layouts

### Filtering System

**Filtering Dimensions:**
- **Tags**: `--tags admin,billing`
- **Path patterns**: `--paths "/admin/.*"`
- **HTTP methods**: `--methods GET,POST`
- **Custom flags**: `--flags x-internal`
- **Operation IDs**: `--operations listUsers,createUser`
- **Inverse mode**: `--inverse` (exclude matches)

**Implementation:**
- Use `openapi-filter` for flag-based filtering
- Use `openapi-format` for structural filtering
- Always run `--removeUnusedComponents` after filtering
- Re-validate spec after aggressive filtering

## CLI Design

### Command Structure

```bash
openkit <command> [options]

Commands:
  generate   Transform spec to output types
  filter     Extract filtered spec subset
  validate   Validate OpenAPI spec
  diff       Compare spec versions (v1.0+)
  enrich     AI-powered enrichment (v2.0+)
````

### Core Examples

```bash
# Basic markdown generation
openkit generate ./openapi.json --output markdown --out ./docs/api.md

# Filtered multi-output
openkit generate ./openapi.json \
  --output markdown,ts-types,zod \
  --tags admin \
  --out-dir ./generated

# Config-driven
openkit generate  # reads openkit.config.ts
```

## Configuration

### Config File Pattern

```typescript
// openkit.config.ts
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
      template: './templates/custom.eta', // optional
    },
    {
      type: 'ts-types',
      outFile: './src/types/admin.d.ts',
      options: {
        transform: (schema) => {
          if (schema.format === 'date-time') return 'Date'
        },
      },
    },
  ],
})
```

## Development Workflow

### Adding Features

1. **New Output Types:**
   - Implement transformer function
   - Register in transformer map
   - No pipeline changes needed

2. **Filter Enhancements:**
   - Extend filter engine in `src/filter.ts`
   - Update CLI options in `src/cli.ts`
   - Add config schema support

3. **CLI Commands:**
   - Add command handler in `src/cli.ts`
   - Follow commander.js patterns
   - Keep startup time minimal

### Testing Strategy

**Test Structure:**

- `test/` directory with `*.test.ts` files
- Import from `../src/module` (not dist)
- Vitest with globals enabled
- Focus on transformer outputs and filtering logic

**Key Test Areas:**

- Parser handles various OpenAPI formats
- Filter engine produces correct subsets
- Transformers generate expected output
- CLI commands work with various options

### Build Process

**Build Tools:**

- `tsdown` for fast TypeScript bundling
- Dual CJS/ESM output in `dist/`
- TypeScript declarations with source maps

**Output Structure:**

```
dist/
├── index.cjs           # CommonJS bundle
├── index.js            # ESM bundle
├── index.d.ts          # TypeScript declarations
└── *.map               # Source maps
```

## Code Quality Standards

### TypeScript Configuration

- Strict mode enabled
- ES2022 target
- Module preservation for dual format support
- Type-checked ESLint rules

### Formatting Rules

- Prettier with single quotes, no semicolons
- 100 character line width
- Consistent formatting across all files

### Git Workflow

- Conventional commits for changelog generation
- Lefthook pre-commit hooks for formatting/linting
- Changesets for version management

## Roadmap Awareness

### Current Phase (v0.1)

- Core CLI with markdown output ✅
- Full filtering support ✅
- Config file support ✅
- Validation command ✅

### Next Phase (v0.2)

- TypeScript types output 🔄
- Zod schemas output 🔄
- SDK generation 🔄
- Multiple outputs in single run 🔄

### Future Phases

- Watch mode (v1.0)
- Interactive wizard (v1.0)
- AI enrichment (v2.0)
- MCP server exposure (v2.0)

## Common Patterns

### Error Handling

```typescript
// Always validate after filtering
const filtered = await filterSpec(spec, options)
await validateSpec(filtered) // Re-validate after aggressive filtering
```

### Atomic Writes

```typescript
// For multiple outputs, write to temp then rename
const tempDir = await createTempDir()
await writeAllOutputs(tempDir, results)
await atomicMove(tempDir, outputDir)
```

### Extensibility

```typescript
// Transformer registration pattern
export const transformers = {
  markdown: () => import('./markdown'),
  'ts-types': () => import('./ts-types'),
  zod: () => import('./zod'),
  'my-custom': () => import('./my-custom'), // Easy to add
}
```

## Performance Considerations

- Commander.js chosen for fast startup (~18ms vs 85ms for Oclif)
- Filter once, transform many times
- Lazy-load transformers to reduce initial bundle size
- Cache parsed specs for watch mode (future)

## Security Guidelines

- Validate all OpenAPI inputs
- Sanitize file paths for output
- Handle remote URL inputs safely
- Consider auth headers for private specs (v0.2)

## Documentation Standards

- JSDoc comments for all public APIs
- README examples for common use cases
- Architecture documentation for contributors
- Migration guides between versions

---

*This document should be updated as the project evolves through its roadmap phases.*ollow this contract:

```typescript
type Transformer<TOption
```
