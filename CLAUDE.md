# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

NaviHive is a modern website navigation management system built as a full-stack application deployed on Cloudflare Workers. It combines a React 19 frontend with a Cloudflare Workers backend using D1 (SQLite) database.

## Development Commands

### Essential Commands
```bash
# Install dependencies
pnpm install

# Start development server (runs Vite + Cloudflare Workers locally)
pnpm dev

# Build the project
pnpm build

# Preview production build locally
pnpm preview

# Deploy to Cloudflare Workers
pnpm deploy

# Linting and formatting
pnpm lint
pnpm format
pnpm format:check

# Security: Generate password hash for authentication
pnpm hash-password <your-password>

# Generate Cloudflare Workers types
pnpm cf-typegen
```

### Wrangler CLI Commands
```bash
# Login to Cloudflare
wrangler login

# Create D1 database
wrangler d1 create navigation-db

# Execute SQL on D1 database
wrangler d1 execute navigation-db --file=schema.sql

# Export D1 database
wrangler d1 export navigation-db

# View logs
wrangler tail
```

## Architecture

### Frontend Architecture
- **Framework**: React 19 with TypeScript
- **UI Library**: Material UI 7.0 with emotion styling
- **Styling**: Tailwind CSS 4.1 + CSS-in-JS (emotion)
- **Drag & Drop**: DND Kit for sortable groups and sites
- **Build Tool**: Vite 6 with Cloudflare plugin
- **API Layer**: Client abstraction in `src/API/` with mock support for development

### Backend Architecture
- **Runtime**: Cloudflare Workers (serverless)
- **Database**: Cloudflare D1 (SQLite)
- **Authentication**:
  - JWT tokens signed with Web Crypto API (HMAC-SHA256)
  - Password hashing with bcrypt (10 salt rounds)
  - HttpOnly cookie storage for tokens
- **Entry Point**: `worker/index.ts` handles all API routes
- **Security**: Request validation, size limits, CORS, error handling

### Key Design Patterns

1. **API Route Structure**: All API routes are prefixed with `/api/` and handled in `worker/index.ts`
   - Authentication middleware checks JWT tokens for protected routes
   - Input validation functions prevent malformed data
   - Routes for: groups, sites, configs, login, export/import

2. **Client Architecture**: Two API implementations
   - `NavigationClient` (src/API/client.ts) - Real HTTP client
   - `MockNavigationClient` (src/API/mock.ts) - In-memory mock for dev
   - Selected via environment variables (`VITE_USE_REAL_API`)

3. **State Management**: Component-level state with React hooks
   - No global state library (Redux, Zustand, etc.)
   - API responses cached in component state
   - Drag-and-drop state managed by DND Kit

4. **Database Schema**: Three main tables with guest access support
   - `groups`: Navigation categories with ordering and `is_public` flag
   - `sites`: Website links associated with groups with `is_public` flag
   - `configs`: Key-value store for site settings (title, name, custom CSS)

## v1.1.0 New Features

### Guest Access Mode (Public/Private Content)
- **Feature**: Allow unauthenticated users to view public content
- **Configuration**: `AUTH_REQUIRED_FOR_READ` environment variable
  - `false` (default): Guest mode enabled, public content accessible
  - `true`: All content requires authentication (legacy behavior)
- **Database Fields**: Both `groups` and `sites` tables have `is_public` field (0=private, 1=public)
- **Access Control**:
  - Guests: Only see `is_public=1` groups and sites
  - Authenticated admins: See all content
  - Write operations always require authentication

### Login Rate Limiting
- **Implementation**: `SimpleRateLimiter` class in `worker/index.ts`
- **Default Limit**: 5 attempts per 15 minutes per IP
- **Behavior**: Returns 429 status when limit exceeded
- **Logging**: Records IP address of rate-limited requests

## Configuration

### Environment Variables (wrangler.jsonc)
```jsonc
{
  "vars": {
    "AUTH_ENABLED": "true",    // Enable/disable authentication
    "AUTH_REQUIRED_FOR_READ": "false",  // Guest mode: false=allow public access, true=require auth for all
    "AUTH_USERNAME": "admin",   // Admin username
    "AUTH_PASSWORD": "$2a$10$...", // Admin password bcrypt hash (generate with: pnpm hash-password yourPassword)
    "AUTH_SECRET": "secret-key"  // JWT signing key (use strong random value)
  },
  "d1_databases": [{
    "binding": "DB",
    "database_name": "navigation-db",
    "database_id": "your-database-id"
  }]
}
```

**Password Security**: Use `pnpm hash-password <password>` to generate bcrypt hashes for AUTH_PASSWORD.

### Frontend Environment Variables
- `VITE_USE_REAL_API`: Set to "true" to use real API in development (default: use mock)

## Database Initialization

The database must be initialized after deployment. Run SQL in Cloudflare D1 console:

```sql
-- Create groups table
CREATE TABLE IF NOT EXISTS groups (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL,
    order_num INTEGER NOT NULL,
    is_public INTEGER DEFAULT 1,  -- New in v1.1.0: 0=private, 1=public
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Create sites table
CREATE TABLE IF NOT EXISTS sites (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    group_id INTEGER NOT NULL,
    name TEXT NOT NULL,
    url TEXT NOT NULL,
    icon TEXT,
    description TEXT,
    notes TEXT,
    order_num INTEGER NOT NULL,
    is_public INTEGER DEFAULT 1,  -- New in v1.1.0: 0=private, 1=public
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (group_id) REFERENCES groups(id) ON DELETE CASCADE
);

-- Create configs table
CREATE TABLE IF NOT EXISTS configs (
    key TEXT PRIMARY KEY,
    value TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Mark as initialized
INSERT INTO configs (key, value) VALUES ('DB_INITIALIZED', 'true');

-- Create indexes for guest mode performance (v1.1.0)
CREATE INDEX IF NOT EXISTS idx_groups_is_public ON groups(is_public);
CREATE INDEX IF NOT EXISTS idx_sites_is_public ON sites(is_public);
```

**Migration from v1.0.x**: If upgrading from an older version, run `migrations/002_add_is_public.sql`

## Key Code Locations

### Backend (Cloudflare Workers)
- `worker/index.ts` - Main worker entry point with all API routes
- `src/API/http.ts` - NavigationAPI class with database operations

### Frontend Components
- `src/App.tsx` - Main application with layout and state management
- `src/components/GroupCard.tsx` - Display group with sites
- `src/components/SiteCard.tsx` - Individual site display
- `src/components/LoginForm.tsx` - Authentication UI
- `src/components/SortableGroupItem.tsx` - Drag-and-drop wrapper for groups
- `src/components/SiteSettingsModal.tsx` - Global settings dialog
- `src/components/ThemeToggle.tsx` - Dark/light mode toggle

### API Layer
- `src/API/client.ts` - HTTP client implementation
- `src/API/mock.ts` - Mock client for development
- `src/API/http.ts` - Shared types and server-side API class

## Important Implementation Details

### Authentication Flow
1. User submits credentials via LoginForm
2. Worker validates against AUTH_USERNAME/AUTH_PASSWORD (bcrypt verification)
3. Worker checks rate limit (5 attempts/15 min per IP)
4. On success, JWT token generated with configurable expiry (7 days or 30 days if "remember me" checked)
5. Token stored in **HttpOnly Cookie** (primary, prevents XSS) and localStorage (fallback for compatibility)
6. All subsequent API requests automatically include cookie; Authorization header supported as fallback
7. Worker middleware validates token before processing protected routes
8. `/api/auth/status` endpoint verifies authentication state

### Guest Access Flow (v1.1.0)
1. Unauthenticated requests to read-only routes (`/api/groups`, `/api/sites`, `/api/groups-with-sites`, `/api/configs`)
2. If `AUTH_REQUIRED_FOR_READ=false`, middleware checks for token but doesn't require it
3. If token exists and valid, `isAuthenticated=true` (show all content)
4. If no token or invalid, `isAuthenticated=false` (show only `is_public=1` content)
5. Write operations (POST/PUT/DELETE) always require valid authentication

### Drag-and-Drop Ordering
1. User clicks "编辑排序" (Edit Sort) button
2. App enters sort mode (GroupSort or SiteSort)
3. DND Kit provides sortable interface
4. On save, batch updates sent to `/api/group-orders` or `/api/site-orders`
5. Backend updates `order_num` field for all affected items

### Data Export/Import
- Export: Serializes all groups, sites, and configs to JSON with version/timestamp
- Import:
  - Merges groups by name (creates if new, uses existing if found)
  - Sites matched by URL within same group (updates if found, creates if new)
  - Configs completely replaced by imported values

### Custom Styling
- Configs table stores `CUSTOM_CSS` key
- CSS injected into `<style>` tag in document head
- Allows users to override default styles without code changes

## Code Style

### Prettier Configuration
- Print width: 100 characters
- Indentation: 2 spaces
- Single quotes (including JSX)
- Semicolons required
- Trailing commas in ES5 style
- Arrow function parens: always

Format code before committing:
```bash
pnpm format
```

## Testing Considerations

- No test suite currently exists
- When adding tests, structure them by layer (unit tests for API, integration for components)
- Consider testing Worker routes with Miniflare (Cloudflare Workers simulator)
- Mock D1 database for unit tests

## Security Notes

### Implemented Security Measures

1. **Authentication & Authorization**
   - JWT-based authentication using Web Crypto API (HMAC-SHA256)
   - Password hashing with bcrypt (10 salt rounds)
   - HttpOnly cookies for token storage (prevents XSS token theft)
   - Configurable token expiry: 7 days (standard) or 30 days ("remember me")
   - Cookie fallback to Authorization header for backward compatibility

2. **Input Validation**
   - Request body size limit: 1MB (prevents memory exhaustion)
   - Deep validation for all import data (structure, types, URL formats)
   - Field whitelisting in update operations
   - Type validation for all user inputs
   - All inputs validated before database operations

3. **SQL Injection Protection**
   - All database queries use D1 prepared statements
   - Parameterized queries with `.bind()` - never string concatenation
   - Field whitelisting prevents unauthorized column updates

4. **XSS Protection**
   - React escapes all outputs by default
   - Custom CSS sanitization removes dangerous patterns:
     - `javascript:`, `data:text/html`, `vbscript:` protocols
     - `@import`, `expression()`, `-moz-binding`
     - Event handlers and inline scripts
   - URL validation with protocol whitelist (https:, data:image/)

5. **SSRF Protection**
   - URL validation blocks private IP ranges:
     - Localhost (127.0.0.1, ::1)
     - Private IPv4 (10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16)
     - Link-local (169.254.0.0/16, fe80::/10)
   - Only HTTPS and data:image/ URLs allowed

6. **CORS Configuration**
   - Whitelist-based origin validation
   - Automatic same-origin allowance
   - Supports workers.dev subdomains in development
   - Credentials enabled for cookie-based auth

7. **Error Handling**
   - Structured logging with unique error IDs
   - User-friendly error messages (no stack traces to client)
   - Detailed server-side logging for debugging
   - Request context (path, method) included in error logs

8. **TypeScript Strict Mode**
   - Comprehensive type checking enabled
   - No implicit any, strict null checks
   - No unchecked indexed access
   - Prevents type-related runtime errors

### Security Best Practices

- **AUTH_SECRET**: Use cryptographically random value (32+ characters)
- **Password Hashing**: Always use `pnpm hash-password` - never store plaintext
- **HTTPS Only**: Enforce HTTPS in production (automatic on Cloudflare Workers)
- **Token Storage**: Tokens stored in HttpOnly cookies (primary) and localStorage (fallback)
- **Content Security**: Custom CSS limited to 50KB, sanitized before injection

## Deployment Workflow

1. Update code locally
2. Test with `pnpm dev`
3. Build with `pnpm build`
4. Deploy with `pnpm deploy` (runs build automatically)
5. Cloudflare Workers deploys globally within seconds
6. Database migrations must be run manually via Cloudflare dashboard

## Recent Security Improvements

### Phase 1: Critical Security Fixes (Completed)
- ✅ **CR-001**: JWT signing implementation using Web Crypto API
- ✅ **CR-003**: XSS protection with CSS sanitization
- ✅ **CR-004**: SSRF protection with URL validation
- ✅ **CR-002**: SQL injection prevention with parameterized queries

### Phase 2: High Priority Fixes (Completed)
- ✅ **HS-001**: HttpOnly cookies for secure token storage
- ✅ **HS-002**: Login rate limiting (5 attempts/15 min, in-memory SimpleRateLimiter)
- ✅ **HS-003**: bcrypt password hashing
- ✅ **HS-004**: CORS configuration with origin validation
- ✅ **HS-005**: Structured error handling with unique error IDs

### Phase 3: Medium Priority Fixes (Completed)
- ✅ **MS-001**: TypeScript strict mode enabled (65+ type errors fixed)
- ✅ **MS-005**: Request body size limits (1MB max)
- ✅ **MS-007**: Deep data validation for import operations

### v1.1.0 Feature Additions (Completed)
- ✅ Guest access mode with `AUTH_REQUIRED_FOR_READ` configuration
- ✅ Public/private content control via `is_public` database field
- ✅ Database migration script for existing deployments
- ✅ Frontend dual-mode support (guest/authenticated views)
- ✅ Authentication middleware refactoring for conditional access

### Total Security & Feature Commits: 14
All security fixes and v1.1.0 features have been committed to git with detailed commit messages and are deployed.

## Troubleshooting

### Build Issues
- Ensure all dependencies installed: `pnpm install`
- Check TypeScript compilation: `tsc -b`
- Verify wrangler.jsonc is valid JSON with comments

### Database Issues
- Check D1 binding name matches "DB" in wrangler.jsonc
- Verify database_id is correct
- Ensure database initialized (check configs table for DB_INITIALIZED)

### Authentication Issues
- Verify AUTH_ENABLED, AUTH_USERNAME, AUTH_PASSWORD set in wrangler.jsonc
- Check token in localStorage (key: "authToken")
- Tokens expire after 7 days (30 days with remember me)

### Development Mode
- By default, uses MockNavigationClient to avoid needing D1 in dev
- Set `VITE_USE_REAL_API=true` in .env to use real backend locally
- Requires `pnpm dev` to start local Cloudflare Workers server
