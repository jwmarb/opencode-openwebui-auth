# AGENTS.md — opencode-openwebui-auth

> An opencode plugin that routes chat completions through an OpenWebUI instance
> using JWT authentication. Built with TypeScript, Bun, and Biome.

## Build / Lint / Test Commands

```bash
# Full build (typecheck + bundle)
bun run build

# Typecheck only (no emit)
bun run build:typecheck

# Bundle only (skip typecheck)
bun run build:bundle

# Watch mode (typecheck)
bun run dev

# Lint (Biome)
bun run lint

# Lint + auto-fix
bun run lint:fix

# Format
bun run format
```

### Testing

There is **no test framework** configured. No test files exist.
If you add tests, use `bun test` (built-in Bun test runner).

```bash
# Run all tests (when added)
bun test

# Run a single test file
bun test src/oauth/jwt.test.ts

# Run tests matching a name pattern
bun test --grep "parseJwtClaims"
```

## Project Structure

```
src/
├── index.ts            # Plugin entry point — exports OpenWebUIAuthPlugin
├── cli.ts              # CLI tool (add/list/remove/use/models/whoami)
├── storage.ts          # Account persistence (~/.config/opencode/openwebui-accounts.json)
├── logger.ts           # File + stderr logging
├── types.ts            # All shared TypeScript interfaces
├── opencode-auth.ts    # Reads/writes opencode's auth.json
├── plugin/
│   └── fetch.ts        # Custom fetch: URL rewrite, header scrub, Bedrock tool compat
└── oauth/
    ├── api.ts          # OpenWebUI REST client (verifyToken, listModels, fetchInstanceConfig)
    └── jwt.ts          # JWT decode + expiry check
```

## Code Style

### Formatting (enforced by Biome)

- **Indent**: 4 spaces
- **Quotes**: double quotes for strings (`"hello"`, not `'hello'`)
- **Semicolons**: omitted (no trailing semicolons — Biome default)
- **Trailing commas**: yes, in multi-line constructs
- **Line length**: not explicitly configured; keep lines reasonable

Run `bun run format` and `bun run lint:fix` before committing.

### Imports

- Use `import type { ... }` for type-only imports — this is enforced by TypeScript strict mode + bundler resolution.
- Group imports in order: (1) Node built-ins (`node:fs`, `node:path`), (2) external packages (`@opencode-ai/plugin`), (3) local modules (`./storage`, `../types`).
- Biome's `organizeImports` assist is enabled — it will auto-sort on format.
- Use `node:` prefix for Node built-ins (e.g., `import { join } from "node:path"`).

```typescript
import { mkdirSync, readFileSync } from "node:fs"
import { join } from "node:path"
import type { PluginInput } from "@opencode-ai/plugin"
import { log } from "./logger"
import type { OpenWebUIAccount } from "./types"
```

### TypeScript Conventions

- **Strict mode** is enabled (`"strict": true` in tsconfig).
- **Target**: ESNext, module: ESNext, moduleResolution: bundler.
- **No runtime dependencies** — only devDependencies. The build bundles everything into `dist/bundle.js`.
- All shared types live in `src/types.ts`. Keep interfaces there, not scattered across files.
- Use `interface` for object shapes (not `type` aliases for objects).
- Prefer `as` type assertions for JSON parsing: `JSON.parse(raw) as MyType`.
- Use `Record<string, T>` over `{ [key: string]: T }`.

### Naming

- **Files**: kebab-case (`opencode-auth.ts`, `fetch.ts`)
- **Interfaces**: PascalCase, prefixed with domain (`OpenWebUIAccount`, `JwtClaims`)
- **Constants**: UPPER_SNAKE_CASE (`PROVIDER_ID`, `DUMMY_KEY`, `STORE_PATH`)
- **Functions**: camelCase (`parseJwtClaims`, `makeOwuiFetch`, `normalizeBaseUrl`)
- **Classes**: PascalCase (`Storage`)
- **Private helpers**: module-scoped plain functions, not class methods

### Error Handling

- Throw `new Error(...)` with descriptive messages including context (status codes, truncated bodies).
- For non-critical I/O (logging, dir creation), use bare `try { ... } catch {}` — swallow silently.
- For critical I/O (storage save), catch, log, then re-throw.
- Validate inputs early and throw; use `undefined` returns for "not found" cases (e.g., `parseJwtClaims`).
- Pattern: `err instanceof Error ? err.message : err` for unknown caught values.

```typescript
// Non-critical — swallow
try { mkdirSync(dir, { recursive: true }) } catch {}

// Critical — log + rethrow
try {
    writeFileSync(tmp, data)
    renameSync(tmp, this.path)
} catch (err) {
    log(`[storage] save failed: ${err instanceof Error ? err.message : err}`)
    throw err
}

// Validation — return undefined
if (parts.length !== 3) return undefined
```

### Patterns in This Codebase

- **Atomic file writes**: write to `.tmp-{pid}-{timestamp}` then `renameSync` to final path. File mode `0o600` for secrets.
- **Factory functions over classes**: `makeOwuiFetch(storage)` returns a closure, not a class instance.
- **URL handling**: always normalize with `normalizeBaseUrl()`, strip trailing slashes, validate `https?://` prefix.
- **Headers**: construct with `new Headers()`, iterate source headers to copy, then delete sensitive ones (`x-api-key`, `anthropic-version`).
- **Logging**: use the `log()` / `logAuth()` / `logRequest()` helpers from `src/logger.ts`. Debug-verbose output gated behind `OPENWEBUI_AUTH_DEBUG=verbose`.
- **CLI structure**: top-level `switch` on `process.argv[2]`, each command is a standalone `async function cmdXxx(args)`.

### What NOT to Do

- Do not add runtime dependencies — everything must bundle into a single ESM file.
- Do not use `require()` — this is ESM-only.
- Do not store secrets in code — tokens go to `~/.config/opencode/` with `0o600` permissions.
- Do not use `console.log` in plugin code — use `log()` from `src/logger.ts`. (`console.log` is fine in `cli.ts`.)
- Do not use `any` — use `unknown` + type narrowing or explicit `as` casts.
