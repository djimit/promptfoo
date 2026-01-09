# Promptfoo Build Guide

Complete documentation for building, testing, and developing promptfoo - an open-source LLM evaluation framework.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Quick Start](#quick-start)
- [Project Structure](#project-structure)
- [Build System](#build-system)
- [Development](#development)
- [Testing](#testing)
- [Linting & Formatting](#linting--formatting)
- [Database](#database)
- [Docker](#docker)
- [CI/CD](#cicd)
- [Troubleshooting](#troubleshooting)
- [Missing Features & Known Limitations](#missing-features--known-limitations)

---

## Prerequisites

### Required

| Tool | Version | Notes |
|------|---------|-------|
| Node.js | >=20.0.0 | Use `nvm use` to match `.nvmrc` (v24.7.0) |
| npm | >=11.6.4 | Node 24 ships with older npm; upgrade via `npm install -g npm@latest` |
| Python | >=3.9 | Required for Python providers, prompts, assertions |
| Git | Latest | Required for version control |

### Optional

| Tool | Purpose |
|------|---------|
| Ruby 3.0+ | Ruby provider support |
| Go 1.23+ | Go provider support |
| Docker | Container builds |
| Playwright | Browser automation tests |

### Environment Setup

```bash
# Clone the repository
git clone https://github.com/promptfoo/promptfoo.git
cd promptfoo

# Use correct Node version
nvm use

# Upgrade npm if using Node 24.x
npm install -g npm@latest

# Install all dependencies
npm ci
```

---

## Quick Start

```bash
# Full build (type check + transpile + frontend)
npm run build

# Run development server (hot reload)
npm run dev

# Run tests
npm test

# Lint and format
npm run l && npm run f
```

---

## Project Structure

```
promptfoo/
├── src/                    # Core library source code
│   ├── app/               # React 19 web UI (Vite workspace)
│   ├── commands/          # CLI command implementations
│   ├── providers/         # LLM provider integrations (50+)
│   ├── server/            # Express backend server
│   ├── redteam/           # Security testing module
│   ├── database/          # Drizzle ORM setup
│   ├── util/              # Shared utilities
│   ├── types/             # TypeScript type definitions
│   ├── assertions/        # Test assertions
│   ├── python/            # Python helper scripts
│   ├── ruby/              # Ruby provider wrappers
│   ├── golang/            # Go provider wrappers
│   ├── index.ts           # Library entry point
│   └── main.ts            # CLI entry point
├── test/                  # Test suite (Vitest)
├── site/                  # Docusaurus documentation site
├── examples/              # 230+ example configurations
├── drizzle/               # Database migrations
├── scripts/               # Build and utility scripts
├── docs/                  # Agent/development documentation
└── .github/workflows/     # CI/CD pipelines
```

### Workspaces

The project uses npm workspaces:

- `src/app` - React frontend
- `site` - Documentation site (Docusaurus)
- `code-scan-action` - GitHub Action for code scanning

---

## Build System

### Overview

Promptfoo uses **tsdown** (modern esbuild-based bundler) for fast, efficient builds.

### Build Commands

| Command | Description |
|---------|-------------|
| `npm run build` | Full build: type check + transpile + frontend |
| `npm run build:clean` | Remove `dist/` directory |
| `npm run build:watch` | Watch mode with hot rebuilding |
| `npm run build:app` | Build frontend only |
| `npm run tsc` | TypeScript type checking only |

### Build Configuration

**File:** `tsdown.config.ts`

The build produces four outputs:

1. **Server Build (ESM)** - `dist/src/server/index.js`
   - Entry: `src/server/index.ts`
   - Target: Node 20

2. **CLI Binary (ESM)** - `dist/src/main.js`
   - Entry: `src/main.ts`
   - Includes shebang for direct execution

3. **Library ESM** - `dist/src/index.js`
   - Entry: `src/index.ts`
   - Tree-shaking enabled

4. **Library CJS** - `dist/src/index.cjs`
   - Entry: `src/index.ts`
   - For CommonJS compatibility

### Build-Time Constants

Injected via `tsdown.config.ts`:

```typescript
__PROMPTFOO_VERSION__        // Package version
__PROMPTFOO_POSTHOG_KEY__    // Analytics key (from env)
__PROMPTFOO_ENGINES_NODE__   // Node.js requirement
BUILD_FORMAT                 // 'esm' or 'cjs'
```

### Frontend Build

**File:** `src/app/vite.config.ts`

- **Framework:** Vite + React 19
- **Output:** `dist/src/app/`
- **Features:**
  - Manual chunk splitting for optimal loading
  - Browser-compatible module replacements
  - React Compiler (babel-plugin-react-compiler)
  - Source maps (hidden in production)

### Post-Build

**File:** `scripts/postbuild.ts`

Copies non-TypeScript assets:
- HTML templates
- Python/Ruby/Go wrapper scripts
- Drizzle migration files
- Sets executable permissions on CLI

---

## Development

### Running Locally

```bash
# Start both server and frontend (recommended)
npm run dev

# Or run separately:
npm run dev:server    # API server at localhost:3000
npm run dev:app       # Frontend at localhost:3000

# Test CLI with local build
npm run local -- eval -c path/to/config.yaml --no-cache

# Link for global CLI access
npm link promptfoo
promptfoo eval -c config.yaml
```

### DevContainer

The repository includes a DevContainer configuration (`.devcontainer/devcontainer.json`):

```bash
# Post-create automatically runs:
npm ci && npm run build && npm link
```

### Environment Variables

Create a `.env` file for API keys (never commit this file):

```bash
# Required for most providers
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-...

# Optional
LOG_LEVEL=debug                              # Verbose logging
PROMPTFOO_DISABLE_REMOTE_GENERATION=true     # Skip remote generation
PROMPTFOO_POSTHOG_KEY=...                    # Analytics (production only)
```

Load environment variables:

```bash
npm run local -- eval -c config.yaml --env-file .env --no-cache
```

### Important Development Notes

1. **Always run from repository root**, not subdirectories
2. **Always use `--no-cache`** during development
3. **Use `--` before flags** with npm scripts:
   ```bash
   npm run local -- eval --max-concurrency 1  # Correct
   npm run local eval --max-concurrency 1     # Wrong
   ```
4. **Don't run `npm run local -- view`** unless explicitly needed (use `npm run dev` instead)

---

## Testing

### Framework

All tests use **Vitest** with different configurations per test type.

### Test Commands

| Command | Description |
|---------|-------------|
| `npm test` | Run all unit tests |
| `npm run test:watch` | Watch mode |
| `npm run test:integration` | Integration tests |
| `npm run test:smoke` | Smoke tests (requires build) |
| `npm run test:app` | Frontend component tests |
| `npm run test:site` | Documentation site tests |
| `npm run test:redteam:integration` | Red team integration tests |
| `npx vitest path/to/test` | Run specific test file |

### Test Configuration

**Unit Tests** (`vitest.config.ts`):
- Pool: Forks (child processes for memory isolation)
- Workers: CPU count - 2 (min 4)
- Memory: 3GB per worker
- Timeout: 30s per test
- Shuffle: Enabled (catches isolation issues)

**Integration Tests** (`vitest.integration.config.ts`):
- Timeout: 60s per test
- Memory: 4GB per worker
- Globals: Enabled (`describe`, `it`, `expect`)

**Frontend Tests** (`src/app/vite.config.ts`):
- Environment: jsdom
- Memory: 2GB per worker
- CSS processing enabled

### Test Setup

**File:** `vitest.setup.ts`

Configures test environment:
- Sets `NODE_ENV=test`
- Uses memory cache
- Mocks API keys
- Cleans up mocks/timers after each test

### Writing Tests

```typescript
// Backend tests (test/) - globals disabled
import { describe, it, expect, vi } from 'vitest';

// Frontend tests (src/app/) - globals disabled
import { describe, it, expect } from 'vitest';
import { render, screen } from '@testing-library/react';
```

---

## Linting & Formatting

### Tools

- **Biome** - Linting & formatting for JS/TS/JSON
- **Prettier** - Formatting for CSS/HTML/MD/YAML

### Commands

| Command | Description |
|---------|-------------|
| `npm run lint` | Lint src/ directory |
| `npm run lint:src` | Lint src/ directory |
| `npm run lint:tests` | Lint test/ directory |
| `npm run lint:site` | Lint site/ directory |
| `npm run lint:ci` | Stricter CI mode |
| `npm run format` | Format all files |
| `npm run format:check` | Check formatting |
| `npm run l` | Lint changed files only |
| `npm run f` | Format changed files only |

### Before Committing

```bash
npm run l && npm run f
```

### Biome Configuration

**File:** `biome.json`

Key settings:
- Indent: 2 spaces
- Line width: 100 characters
- Single quotes (double for JSX)
- Trailing commas: All
- Semicolons: Always
- **Custom rule:** Enforce `fetchWithProxy()` over native `fetch`

### Prettier Configuration

**File:** `.prettierrc.yaml`

```yaml
trailingComma: 'all'
singleQuote: true
printWidth: 100
bracketSpacing: true
```

---

## Database

### Overview

Promptfoo uses **Drizzle ORM** with SQLite (file-based at `~/.promptfoo/promptfoo.db`).

### Commands

| Command | Description |
|---------|-------------|
| `npm run db:generate` | Generate new migrations |
| `npm run db:migrate` | Apply pending migrations |
| `npm run db:studio` | Open Drizzle Studio UI |

### Configuration

**File:** `drizzle.config.ts`

```typescript
{
  dialect: 'turso',
  schema: './src/database/tables.ts',
  out: './drizzle',
  dbCredentials: {
    url: `file:${getDbPath()}`,
  },
}
```

### Important Notes

- **Never delete** `~/.promptfoo/promptfoo.db` without explicit permission
- Migrations are in `drizzle/` directory
- Schema defined in `src/database/tables.ts`

---

## Docker

### Building

```bash
# Standard build
docker build -t promptfoo .

# With build arguments
docker build \
  --build-arg PYTHON_VERSION=3.12 \
  --build-arg VITE_PUBLIC_BASENAME=/app \
  -t promptfoo .
```

### Dockerfile Overview

**File:** `Dockerfile`

Multi-stage build:
1. **base** - Node 24.7.0 Alpine + Python 3.12
2. **builder** - Install deps, build frontend & backend
3. **server** - Runtime image with built assets

### Running

```bash
docker run -p 3000:3000 promptfoo
```

### Health Check

The container includes a health check endpoint at `/health`.

---

## CI/CD

### Main Workflow

**File:** `.github/workflows/main.yml`

Runs on: Pull requests, pushes to main, manual dispatch

#### Jobs

| Job | Description |
|-----|-------------|
| `test` | Run tests (Node 20/22/24 × Ubuntu/Windows/macOS) |
| `build` | Build project (Node 20/22/24) |
| `style-check` | Biome, Prettier, dependency checks |
| `shell-format` | Shell script formatting |
| `assets` | JSON schema generation |
| `python` | Python linting and tests (3.9, 3.14) |
| `ruby` | Ruby linting and tests (3.0, 3.3) |
| `golang` | Go tests (1.23) |
| `docs` | Documentation site build |
| `site-tests` | Documentation site tests |
| `webui` | Frontend component tests |
| `integration-tests` | Integration tests |
| `smoke-tests` | Smoke tests |
| `share-test` | Share functionality test |
| `redteam` | Red team tests |
| `actionlint` | GitHub Actions linting |

### Other Workflows

| Workflow | Purpose |
|----------|---------|
| `docker.yml` | Docker image builds & publishing |
| `release-please.yml` | Automated versioning & releases |
| `validate-pr-title.yml` | Conventional commit validation |
| `claude-code-review.yml` | AI-powered code review |

### Branch Protection

- Never commit/push directly to main
- All changes go through pull requests
- Windows tests are sharded (3 shards) for speed

---

## Troubleshooting

### Common Issues

#### Build Fails with Memory Error

```bash
# Increase memory limit
NODE_OPTIONS='--max-old-space-size=8192' npm run build
```

#### Tests Fail Intermittently

```bash
# Disable test shuffling to debug
npx vitest --sequence.shuffle=false
```

#### Cache Issues

```bash
# Use --no-cache flag (don't delete cache directory)
npm run local -- eval -c config.yaml --no-cache
```

#### npm Install Fails on Node 24

```bash
# Upgrade npm first
npm install -g npm@latest
npm ci
```

#### Lockfile Compatibility Error

```bash
# Node 24 requires npm >=11.6.4
npm install -g npm@latest
rm -rf node_modules package-lock.json
npm install
```

### Debug Logging

```bash
# Enable verbose logging
LOG_LEVEL=debug npm run local -- eval -c config.yaml

# Or via environment
npm run local -- eval -c config.yaml --verbose
```

### Cache & Database Locations

| Path | Purpose |
|------|---------|
| `~/.cache/promptfoo` | Evaluation cache |
| `~/.promptfoo/promptfoo.db` | SQLite database |
| `~/.promptfoo/config` | User configuration |

---

## Missing Features & Known Limitations

### Build System

1. **No incremental builds** - Full rebuild required on changes; watch mode helps but doesn't do true incremental compilation
2. **No parallel workspace builds** - Workspaces build sequentially; could benefit from parallel execution
3. **No build caching** - No persistent caching like Turborepo or Nx; each build starts fresh
4. **No bundle analysis tool** - No built-in way to analyze bundle size/composition

### Testing

1. **No visual regression testing** - No screenshot comparison for UI components
2. **No E2E testing** - Missing Playwright/Cypress E2E tests for full user flows
3. **No performance benchmarking** - No automated performance regression tests
4. **No mutation testing** - No Stryker or similar mutation testing setup
5. **Limited Windows CI coverage** - macOS Node 24 excluded due to flakiness

### Development Experience

1. **No hot module replacement for backend** - Uses tsx --watch but full restart required
2. **No unified dev command** - Must run separate terminals for full stack development
3. **No API documentation generation** - No TypeDoc or similar for API docs
4. **No changelog auto-generation from commits** - Manual changelog updates required

### Code Quality

1. **No type coverage reporting** - No tooling to track TypeScript strictness over time
2. **No dead code detection** - Knip exists but not integrated into CI
3. **No complexity metrics** - No automated cyclomatic complexity checks
4. **No license compliance checking** - No automated FOSS license auditing

### CI/CD

1. **No automatic dependency updates** - No Dependabot or Renovate configured
2. **No preview deployments** - No Vercel/Netlify preview for PRs
3. **No canary releases** - No automated canary deployment pipeline
4. **No semantic release** - Manual version bumping required

### Infrastructure

1. **No Kubernetes manifests** - Docker-only deployment
2. **No Terraform/Pulumi** - No infrastructure-as-code
3. **No OpenTelemetry integration** - Limited observability in self-hosted deployments
4. **No rate limiting** - No built-in API rate limiting for server

### Documentation

1. **No architecture diagrams** - Missing visual system architecture docs
2. **No API changelog** - No versioned API documentation
3. **No migration guides** - Limited upgrade documentation between versions
4. **No troubleshooting runbook** - No structured debugging guide

### Security

1. **No SAST in CI** - No automated security scanning (CodeQL, Snyk)
2. **No container scanning** - No Trivy or similar for Docker images
3. **No secrets scanning** - No git-secrets or truffleHog integration
4. **No SBOM generation** - No Software Bill of Materials

### Suggested Improvements

#### High Priority

```bash
# Add E2E testing with Playwright
npm install --save-dev @playwright/test
npx playwright install

# Add dependency update automation
# .github/dependabot.yml configuration

# Add bundle analysis
npm install --save-dev rollup-plugin-visualizer
```

#### Medium Priority

```bash
# Add TypeDoc for API documentation
npm install --save-dev typedoc

# Add mutation testing
npm install --save-dev @stryker-mutator/core

# Add visual regression testing
npm install --save-dev @percy/cli
```

#### Low Priority

- Kubernetes deployment manifests
- Infrastructure-as-code templates
- OpenTelemetry tracing integration
- SBOM generation pipeline

---

## Additional Resources

- [AGENTS.md](./AGENTS.md) - Guidance for AI agents
- [docs/agents/](./docs/agents/) - Developer documentation
- [site/](./site/) - Documentation website source
- [examples/](./examples/) - 230+ example configurations
- [CONTRIBUTING.md](./CONTRIBUTING.md) - Contribution guidelines

---

## Version Information

- **Current Version:** 0.120.10
- **Module Type:** ESM with CJS compatibility
- **License:** MIT
- **Repository:** https://github.com/promptfoo/promptfoo
