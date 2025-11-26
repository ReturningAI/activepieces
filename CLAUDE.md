# CLAUDE.md - Activepieces AI Assistant Guide

This document provides guidance for AI assistants working with the Activepieces codebase.

## Project Overview

Activepieces is an open-source automation platform (Zapier alternative) built with TypeScript. It features a type-safe pieces framework for creating integrations and supports 280+ community-contributed connectors.

**Version:** 0.63.1
**Node Version:** 18.20.5 (see `.nvmrc`)
**Package Manager:** npm (pnpm also available)

## Repository Structure

```
activepieces/
├── packages/                 # Nx monorepo packages
│   ├── cli/                  # Developer CLI for piece management
│   ├── ee/                   # Enterprise Edition features
│   │   ├── shared/           # EE shared types
│   │   └── ui/embed-sdk/     # Embedding SDK
│   ├── engine/               # Flow execution engine
│   ├── pieces/               # Integration connectors
│   │   ├── community/        # 280+ community pieces
│   │   └── custom/           # Custom user pieces
│   ├── react-ui/             # Frontend SPA (React + Vite)
│   ├── server/               # Backend services
│   │   ├── api/              # Fastify REST API
│   │   ├── worker/           # Job execution worker
│   │   └── shared/           # Server shared code
│   ├── shared/               # Shared types across packages
│   └── tests-e2e/            # End-to-end tests
├── docs/                     # Mintlify documentation
├── tools/                    # Build scripts and utilities
└── deploy/                   # Pulumi IaC deployment
```

## Quick Reference Commands

### Development
```bash
npm run dev              # Start full stack (frontend + backend + engine)
npm run dev:backend      # Start backend API + engine only
npm run dev:frontend     # Start frontend + backend + engine
npm run serve:frontend   # Frontend only (port 4200)
npm run serve:backend    # Backend API only (port 3000)
```

### Piece Development
```bash
npm run create-piece      # Create new piece scaffold
npm run create-action     # Add action to existing piece
npm run create-trigger    # Add trigger to existing piece
npm run sync-pieces       # Sync pieces registry
npm run build-piece       # Build a specific piece
npm run publish-piece     # Publish piece to npm
```

### Testing
```bash
nx test-unit server-api   # Run unit tests
nx test-ce server-api     # Run Community Edition integration tests
nx test-ee server-api     # Run Enterprise Edition integration tests
nx test-cloud server-api  # Run Cloud edition tests
nx lint <package>         # Lint a specific package
```

### Database
```bash
nx db server-api -- migration:generate -d <datasource> <path>  # Generate migration
nx db-migration server-api --name=MigrationName                # Create migration
```

## Tech Stack

### Frontend (`packages/react-ui/`)
- **Framework:** React 18.3
- **Build:** Vite 6.3
- **State:** Zustand 4.5, TanStack React Query 5.51
- **Styling:** Tailwind CSS 3.4
- **UI Components:** Radix UI
- **Forms:** React Hook Form 7.52
- **Routing:** React Router DOM 6.11
- **Flow Builder:** @xyflow/react 12.3

### Backend (`packages/server/api/`)
- **Framework:** Fastify 4.28
- **Database:** TypeORM 0.3.18 with PostgreSQL (SQLite for dev/test)
- **Queue:** BullMQ 5.28 with Redis
- **WebSocket:** Socket.io
- **Validation:** TypeBox (@sinclair/typebox)

### Engine (`packages/engine/`)
- Flow execution runtime
- Communicates via WebSocket with worker
- Sandboxed execution support via isolated-vm

## Code Style and Conventions

### Import Rules
- **No lodash:** Use native JavaScript methods instead (enforced by ESLint)
- **Path aliases:** Use `@activepieces/*` imports (defined in `tsconfig.base.json`)

### Key Path Aliases
```typescript
@activepieces/shared           // packages/shared/src/index.ts
@activepieces/pieces-framework // packages/pieces/community/framework/src
@activepieces/pieces-common    // packages/pieces/community/common/src
@activepieces/engine           // packages/engine/src/main.ts
@activepieces/ee-shared        // packages/ee/shared/src/index.ts
```

### TypeScript
- Target: ES2015
- Module: ESNext
- Strict mode enabled
- Path mappings in `tsconfig.base.json`

### Linting
```bash
nx lint <package-name>    # Lint specific package
```

ESLint enforces:
- Nx module boundaries
- No lodash imports
- TypeScript best practices

### Formatting
- **Prettier** with single quotes
- Run formatting before commits (Husky pre-commit hooks)

## Pieces Framework

Pieces are TypeScript npm packages that provide integrations. Each piece has:

### Structure
```
packages/pieces/community/<piece-name>/
├── src/
│   ├── index.ts              # Main piece export
│   └── lib/
│       ├── actions/          # Action implementations
│       └── triggers/         # Trigger implementations (optional)
├── package.json              # Minimal: name + version only
├── project.json              # Nx project config
└── tsconfig.json
```

### Piece Definition Pattern
```typescript
import { PieceAuth, createPiece } from '@activepieces/pieces-framework';
import { PieceCategory } from '@activepieces/shared';

export const myAuth = PieceAuth.SecretText({
  displayName: 'API Key',
  required: true,
  validate: async (auth) => {
    // Validation logic
    return { valid: true };
  },
});

export const myPiece = createPiece({
  displayName: 'My Piece',
  description: 'Description here',
  minimumSupportedRelease: '0.30.0',
  logoUrl: 'https://cdn.activepieces.com/pieces/my-piece.png',
  authors: ['author-github-username'],
  categories: [PieceCategory.PRODUCTIVITY],
  auth: myAuth,
  actions: [/* action definitions */],
  triggers: [/* trigger definitions */],
});
```

### Authentication Types
```typescript
PieceAuth.SecretText()    // API key, token
PieceAuth.OAuth2()        // OAuth2 flow
PieceAuth.BasicAuth()     // Username/password
PieceAuth.CustomAuth()    // Custom auth fields
```

### Common Utilities
```typescript
import { httpClient, HttpMethod, AuthenticationType } from '@activepieces/pieces-common';

// Make HTTP requests
await httpClient.sendRequest({
  method: HttpMethod.GET,
  url: 'https://api.example.com/endpoint',
  authentication: {
    type: AuthenticationType.BEARER_TOKEN,
    token: auth.token,
  },
});
```

## Database

### Supported Databases
- **PostgreSQL** (production)
- **SQLite** (development/testing)

### Migrations
Location: `packages/server/api/src/app/database/migration/`
- `common/` - Shared migrations
- `postgres/` - PostgreSQL-specific
- `sqlite/` - SQLite-specific

Naming: `[timestamp]-[DescriptiveName].ts`

### Key Entities
- `FlowEntity`, `FlowVersionEntity` - Workflow definitions
- `FlowRunEntity` - Execution history
- `AppConnectionEntity` - Third-party connections
- `PieceMetadataEntity` - Installed pieces
- `UserEntity`, `ProjectEntity` - Users and workspaces

## Environment Configuration

Key environment variables (see `.env.example`):

```bash
# Core
AP_ENVIRONMENT=prod|dev
AP_FRONTEND_URL=http://localhost:8080
AP_ENGINE_EXECUTABLE_PATH=dist/packages/engine/main.js

# Security
AP_ENCRYPTION_KEY=           # 32 hex characters (256-bit)
AP_JWT_SECRET=               # JWT signing secret
AP_API_KEY=                  # API authentication (optional for CE)

# Database (PostgreSQL)
AP_POSTGRES_HOST=postgres
AP_POSTGRES_PORT=5432
AP_POSTGRES_DATABASE=activepieces
AP_POSTGRES_USERNAME=postgres
AP_POSTGRES_PASSWORD=

# Cache (Redis)
AP_REDIS_HOST=redis
AP_REDIS_PORT=6379

# Execution
AP_EXECUTION_MODE=UNSANDBOXED|SANDBOXED
AP_FLOW_TIMEOUT_SECONDS=600
AP_WEBHOOK_TIMEOUT_SECONDS=30
AP_TRIGGER_DEFAULT_POLL_INTERVAL=5
```

## Testing Strategy

### Unit Tests
- Location: `packages/server/api/test/unit/`
- Run: `nx test-unit server-api`

### Integration Tests
- Location: `packages/server/api/test/integration/`
- Organized by edition: `ce/`, `ee/`, `cloud/`
- Uses `.env.tests` configuration

### E2E Tests
- Location: `packages/tests-e2e/`
- Framework: Playwright

### Test Commands
```bash
nx test-unit server-api      # Unit tests only
nx test-ce server-api        # Community Edition tests
nx test-ee server-api        # Enterprise Edition tests
nx test-cloud server-api     # Cloud tests
nx test server-api           # All tests (parallel)
```

## Build System

### Nx Monorepo
- Caching enabled for build, lint, test targets
- 3 parallel workers by default
- Daemon disabled (`useDaemonProcess: false`)

### Build Outputs
- Backend: `dist/packages/server/api`
- Frontend: `dist/packages/react-ui`
- Engine: `dist/packages/engine`

### Build Commands
```bash
nx build server-api          # Build backend
nx build react-ui            # Build frontend
nx build engine              # Build engine
nx run-many -t build         # Build all
```

## Docker

### Development
```bash
docker-compose -f docker-compose.dev.yml up  # Start dev databases
```

### Production
```bash
docker-compose up            # Full stack with PostgreSQL + Redis
```

### Dockerfile
Multi-stage build:
1. **base** - Node 18.20.5 with system dependencies
2. **build** - Compile TypeScript, bundle assets
3. **run** - Production runtime with Nginx

## CI/CD Workflows

Located in `.github/workflows/`:
- `release.yml` - Production releases
- `release-rc.yml` - Release candidates
- `release-pieces.yml` - Piece publishing
- `validate-pr-title.yml` - PR title validation
- `validate-pr-labels.yml` - PR label validation

## Common Development Tasks

### Adding a New Action to Existing Piece
```bash
npm run create-action
# Follow prompts to select piece and define action
```

### Creating a New Piece
```bash
npm run create-piece
# Follow prompts to scaffold new piece
```

### Running Local Development
```bash
# Start dev databases
docker-compose -f docker-compose.dev.yml up -d

# Start application
npm run dev
```

### Debugging
- Backend: Port 3000 (with source maps)
- Frontend: Port 4200 (HMR enabled)
- Engine: WebSocket on localhost:12345

## Architecture Notes

### Request Flow
1. Frontend → Fastify API (port 3000)
2. API → BullMQ job queue (Redis)
3. Worker picks up job
4. Worker → Engine (WebSocket)
5. Engine executes flow steps
6. Results stored in PostgreSQL

### Editions
- **Community Edition (CE):** MIT licensed, core features
- **Enterprise Edition (EE):** Commercial license, advanced features
- **Cloud:** Hosted version with additional features

EE code in `packages/ee/` and `packages/server/api/src/app/ee/`

## Helpful Resources

- [Documentation](https://www.activepieces.com/docs)
- [Contributing Guide](https://www.activepieces.com/docs/contributing/overview)
- [Create a Piece](https://www.activepieces.com/docs/developers/overview)
- [Discord Community](https://discord.gg/2jUXBKDdP8)
- [Pieces List](https://www.activepieces.com/pieces)
