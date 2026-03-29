# Product Overview

OpenKit is a composable CLI and Node.js library that transforms OpenAPI specifications into multiple useful outputs — documentation, TypeScript types, Zod schemas, and SDK clients — with first-class support for filtering by tags, paths, or methods.

## Purpose

- Transform OpenAPI specs into multiple output formats from a single source
- Provide advanced filtering capabilities to extract relevant API subsets
- Enable both CLI and programmatic usage for maximum flexibility
- Support extensible transformer system for custom output types

## Key Characteristics

- Linear pipeline architecture with independently replaceable stages
- Filter-first approach - filtering happens once before all transformers
- Multiple output formats: markdown, TypeScript types, Zod schemas, SDK clients
- Extensible transformer system for adding new output types
- Both CLI tool and programmatic Node.js library
- Configuration-driven with `openkit.config.ts` support

## Core Value Proposition

- **Single Source, Multiple Outputs**: Generate docs, types, schemas, and SDKs from one OpenAPI spec
- **Smart Filtering**: Extract only relevant API sections (admin endpoints, public APIs, etc.)
- **Developer Experience**: Config-driven usage with sensible defaults
- **Extensibility**: Add new output types without touching core pipeline
