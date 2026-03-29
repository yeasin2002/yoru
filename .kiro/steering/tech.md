# Technology Stack

## Core Dependencies

- **@apidevtools/swagger-parser**: OpenAPI parsing and dereferencing (non-negotiable)
- **openapi-filter**: Flag-based filtering (`x-internal`, custom flags)
- **openapi-format**: Structural filtering (tags, paths, methods, operationIDs)
- **commander**: CLI framework (~180KB, fast startup for npx usage)
- **prettier**: Output formatting for all generated files

## Build System

- **tsdown**: Fast TypeScript bundler for dual CJS/ESM output
- **TypeScript 5.9+**: Strict mode enabled with modern ES2022 target
- **Module System**: Preserve mode supporting both CommonJS and ESM

## Output Format Packages

### Markdown Documentation
- **@scalar/openapi-to-markdown**: Primary markdown transformer (already in use)

### TypeScript Types
- **openapi-typescript**: Zero-runtime TypeScript types, MIT licensed, millisecond execution
- Pairs with `openapi-fetch` for type-safe HTTP clients

### Zod Schemas
- **openapi-zod-client**: OpenAPI → Zod schemas + Zodios client
- Supports Handlebars templates for custom output shapes
- **orval**: Alternative for React Query/SWR hooks alongside schemas

### SDK Generation
- **@hey-api/openapi-ts**: Plugin-based codegen (used by Vercel, PayPal)
- Generates SDK clients, Zod schemas, TanStack Query hooks

## Template Engine

- **eta**: 2KB TypeScript-native template engine for custom markdown layouts
- Zero dependencies, async-capable, supports partials

## Testing

- **Vitest**: Fast unit testing framework with globals enabled
- **Coverage**: v8 provider with 80% threshold for lines, functions, branches, and statements
- **Environment**: Node.js

## Code Quality

- **ESLint**: TypeScript ESLint with recommended type-checked rules
- **Prettier**: Code formatting with single quotes, no semicolons, 100 char line width
- **Husky + lint-staged**: Pre-commit hooks for automatic formatting and linting

## Package Management

- **pnpm**: Primary package manager (also supports npm, yarn, bun)
- **Changesets**: Version management and changelog generation

## Common Commands

### Development

```bash
pnpm run dev          # Run tests in watch mode
pnpm run build        # Build the package (CJS + ESM + types)
pnpm run lint         # Run ESLint and TypeScript type checking
pnpm run lint:fix     # Auto-fix linting issues
```

### OpenKit CLI Commands

```bash
# Core transformation commands
openkit generate ./openapi.json --output markdown --out ./docs/api.md
openkit filter ./openapi.json --tags admin --out ./openapi.admin.json
openkit validate ./openapi.json

# Multi-output generation
openkit generate ./openapi.json --output markdown,ts-types,zod --tags admin --out-dir ./generated
```

### Testing

```bash
pnpm run test              # Run all tests once
pnpm run test:coverage     # Run tests with coverage report
```

### Formatting

```bash
pnpm run format:write      # Format all files with Prettier
pnpm run format:check      # Check formatting without changes
```

### Quality Checks

```bash
pnpm run check-exports     # Validate package exports with ATTW
pnpm run ci                # Run full CI pipeline (build + format + lint + test)
```

### Publishing

```bash
npx changeset              # Create a changeset for version bump
pnpm run local-release     # Version and publish with Changesets
```

## Build Configuration

- **Entry**: `src/index.ts` (library) + `src/cli.ts` (CLI)
- **Output**: `dist/` directory with CJS, ESM, and TypeScript declarations
- **CLI**: `bin/openkit.js` shebang executable
- **Formats**: Both `.cjs` and `.js` (ESM) files generated
- **Type Definitions**: `.d.ts` files with source maps

## Architecture Decisions

### Commander over Oclif
- Startup time: ~18ms vs 85ms (important for npx usage)
- Size: ~180KB vs 12MB+
- Simplicity: Perfect for 5-6 subcommands without plugin system

### Two Filtering Packages
- `openapi-filter`: Flag-based filtering (annotations in spec)
- `openapi-format`: Structural filtering (external criteria)
- They compose cleanly without overlap

### Filter-First Pipeline
- Filter runs once before all transformers
- No duplication of filtering logic across output modules
- All transformers receive the same pre-filtered spec

### Atomic Writes
- Multiple outputs write to temp directory first
- Atomic rename on success prevents partial updates
- Failure leaves previous outputs untouched
