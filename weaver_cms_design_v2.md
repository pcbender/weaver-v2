# Design for Weaver — Static Website Content Management System (v2)

Weaver is a PHP 8.4/MySQL multi-tenant web CMS that generates mostly-static websites. Users manage content (Markdown/HTML) through a responsive web UI with AI assistance, and Weaver builds static HTML sites versioned via Git. The system supports multi-site per user, local authentication (username/password with 2FA) and third-party OAuth, a plugin architecture for extensibility, and exposes both internal and external RESTful APIs. Generated static sites minimize database usage at runtime — the database powers the CMS admin layer only.

## Changes from v1

This revision resolves inconsistencies, fills security/operational gaps, and softens an over-aggressive MVP scope. Major changes:

- **Auth delivery**: Cookie (frontend) vs. Bearer (API) split made explicit; cookie attributes adjusted (`SameSite=Lax`) for OAuth compatibility; CSRF strategy clarified.
- **2FA schema**: Replaced inline TOTP columns in `users` with a `two_factor_methods` table to support WebAuthn/passkeys without migration pain.
- **Plugin trust model**: Honest sandboxing — trusted-only plugins in Phase 3; subprocess isolation introduced in Phase 5.
- **Content versioning**: `content_versions` no longer stores full LONGTEXT bodies; relies on Git for body reconstruction.
- **Templates**: Filesystem-backed (with DB-indexed metadata) rather than DB LONGTEXT.
- **Phase 1 scope reduced**: 2FA and OAuth moved to Phase 2; Phase 1 ships local auth + email verification + lockout only.
- **New sections**: Secret Management, Email Infrastructure, Prompt Injection Defense, GDPR & Data Lifecycle, Operations (worker supervision, Redis HA).
- **Schema additions**: `audit_log.site_id`, `build_jobs.triggered_by_user_id`, `media.content_hash`, `api_keys.revoked_at`, `user_quotas` table.
- **API path fixes**: `{id}` collisions resolved (`{site_id}` / `{content_id}` etc.).

---

## Goals

- **Multi-tenant CMS**: Multi-user system where each user can own/manage multiple static sites
- **Static-first output**: Generate static HTML/CSS/JS sites that require no database at runtime
- **Dual auth**: Local auth (username/password with 2FA + account validation) and third-party OAuth (Google, Apple, Microsoft) via PKCE
- **AI-assisted editing**: Configurable AI integration for content generation, in-page editing, SEO, translation
- **Git versioning**: Every build and content change tracked via per-site Git repositories
- **Dual API surface**: Internal API for the CMS frontend + external API for integrations (API keys)
- **Markdown + HTML**: First-class support for both content formats with live preview
- **Static configuration**: Menus, tag clouds, metadata stored as JSON/YAML files in the generated site
- **Plugin system**: Hook-based extensibility model for community/custom plugins, with a clear trust model
- **Custom templates**: Twig-based template engine with inheritance, blocks, partials, and auto-escaping
- **Performance**: Incremental builds, Redis caching, background job processing, CDN-ready output
- **Security**: Argon2id password hashing, TOTP/WebAuthn 2FA, encrypted token storage, RBAC, input sanitization, rate limiting, plugin sandboxing
- **Compliance**: GDPR-friendly data export, account deletion, configurable retention

## Design

### 1. System Architecture

#### Layered Architecture
```
┌──────────────────────────────────────────────────────┐
│  HTTP Layer (PSR-15 Middleware Pipeline)             │
│  CORS → TenantContext → Auth → CSRF → RateLimit →    │
│  Router                                              │
├──────────────────────────────────────────────────────┤
│  API Layer (RESTful JSON, versioned /api/v1/)        │
│  Controllers: Auth, Sites, Content, Media, AI, Git   │
├──────────────────────────────────────────────────────┤
│  Service Layer (Business Logic)                      │
│  AuthService, SiteService, ContentService,           │
│  BuildService, AIService, PluginService, GitService, │
│  SecretService, MailService                          │
├──────────────────────────────────────────────────────┤
│  Repository Layer (Tenant-Scoped Data Access)        │
│  All queries automatically scoped by tenant context  │
├──────────────────────────────────────────────────────┤
│  Infrastructure Layer                                │
│  MySQL (PDO), Redis (HA), Git CLI, AI Providers,     │
│  File System, Queue (Redis Streams), Vault/KMS,      │
│  Email Transport                                     │
└──────────────────────────────────────────────────────┘
```

#### Directory Structure
```
weaver/
├── public/                  # Web root (index.php single entry point)
├── config/
│   ├── app.yaml             # Core app config
│   ├── ai.yaml              # AI provider config (per-env, no secrets)
│   ├── auth.yaml            # OAuth provider config (no secrets — refs only)
│   └── features.yaml        # Feature flags
├── src/
│   ├── Http/
│   │   ├── Controller/      # API controllers
│   │   ├── Middleware/      # PSR-15 middleware (Auth, Tenant, CSRF, RateLimit, CORS)
│   │   └── Request/         # Request validation objects
│   ├── Service/             # Business logic
│   ├── Repository/          # Tenant-scoped DB repositories
│   ├── Model/               # Entities + Value Objects
│   ├── AI/                  # Provider adapters (OpenAI, Anthropic, Ollama)
│   ├── Build/               # Static site generation engine
│   ├── Git/                 # Git wrapper with locking
│   ├── Plugin/              # Plugin loader, registry, hook system
│   ├── Template/            # Twig-based template layer
│   ├── Auth/                # OAuth + Local Auth + JWT + RBAC + 2FA methods
│   ├── Secret/              # SecretService — vault/KMS abstraction
│   ├── Mail/                # Mail transport + templates + bounce handling
│   ├── Event/               # Event dispatcher (hook system backbone)
│   ├── Queue/               # Job definitions + Redis Streams workers
│   └── Migration/           # Database migration framework
├── plugins/                 # Installed plugins directory
├── templates/               # Filesystem-backed templates (theme-slug/version/...)
├── storage/
│   ├── sites/{site_uuid}/   # Per-site Git repos + built output
│   ├── cache/               # Compiled templates, build cache
│   ├── logs/                # Structured JSON logs
│   ├── uploads/             # Media uploads (outside web root)
│   └── tmp/                 # Temp build directories
├── migrations/              # Versioned SQL migration files
├── openapi/                 # Generated OpenAPI 3.1 spec
├── tests/
│   ├── Unit/
│   ├── Integration/
│   └── E2E/
└── composer.json
```

#### Key Dependencies (Composer)
- `league/commonmark` — Markdown parsing (CommonMark + GFM)
- `league/oauth2-client` + provider packages — OAuth 2.0 PKCE
- `twig/twig` — Template engine
- `firebase/php-jwt` — JWT token handling (RS256)
- `symfony/dependency-injection` — DI container
- `symfony/console` — CLI commands (migrations, builds, cron)
- `monolog/monolog` — Structured logging
- `predis/predis` — Redis client (Sentinel-aware)
- `league/flysystem` — Filesystem abstraction (local, S3)
- `ezyang/htmlpurifier` — HTML sanitization
- `pragmarx/google2fa` — TOTP 2FA generation/validation
- `bacon/bacon-qr-code` — QR code generation for 2FA setup
- `symfony/mailer` — Transactional email
- `web-auth/webauthn-lib` — WebAuthn/passkey support (Phase 4)
- `zircote/swagger-php` — OpenAPI spec generation from annotations

---

### 2. Multi-Tenancy & Isolation Strategy

**Approach: Shared Database, Row-Level Isolation**
- All tables include `user_id` (owner) column; site-specific tables include `site_id`
- **TenantContext middleware**: Extracts authenticated user from JWT → sets TenantContext singleton
- **Repository layer**: All repositories extend `BaseTenantRepository` which automatically appends `WHERE user_id = :tenant_id` (or `site_id`) to every query
- **Cross-tenant prevention**: Repository base class throws `TenantViolationException` if a query attempts to access resources outside the current tenant context
- **Admin bypass**: System admins can set tenant context to any user for support operations (audit-logged with reason)
- **Resource quotas**: `user_quotas` table defines per-user limits (max sites, max storage, max AI tokens/month) keyed by `users.quota_tier`

**Tenant isolation testing**: Integration test suite runs every API endpoint with two distinct tenants and asserts cross-access returns 404 (never leaks existence) or 403 (when access is intentionally checked).

---

### 3. Authentication & Authorization

Weaver supports two authentication methods: **local auth** (username/password with optional 2FA and mandatory email validation) and **OAuth** (Google, Apple, Microsoft). Both flows issue the same JWT tokens for session management. JWTs are delivered via cookies to the browser frontend and via the `Authorization: Bearer` header to programmatic API consumers (with API keys as the more common API-side credential).

#### 3.1 Local Authentication (Username/Password)

##### Registration Flow
1. User submits `email`, `username`, `password`, `password_confirmation` to `POST /api/v1/auth/register`
2. Server validates:
   - Email format
   - Username: 3-30 chars, alphanumeric + underscores
   - Password: minimum 12 characters, checked against breached password list (via k-anonymity API against Have I Been Pwned)
3. **Enumeration-resistant response**: Server always returns `200 OK` with a generic "Check your email" message regardless of whether the email is already registered.
   - If email is new: standard verification email sent.
   - If email exists: a "you already have an account" email is sent to that address with a password-reset link.
   - If username collides (and email is new): user record is created with a temporary suffixed username; verification email asks them to choose a different one on first login. (Alternative: reject username collisions only, since usernames are public; document the choice.)
4. Password hashed with **Argon2id** (`password_hash()` with `PASSWORD_ARGON2ID`, memory_cost: 65536, time_cost: 4, threads: 3)
5. User record created with `email_verified_at = NULL`. `status` is derived (`status = email_verified_at IS NULL ? 'pending_verification' : 'active'`) rather than stored separately for verification state — see schema notes.
6. Verification token generated: `random_bytes(32)` → hex encoded, stored hashed (SHA-256) in `email_verifications` table with 24-hour expiry
7. Verification email sent with link: `https://app.weaver.io/verify-email?token={token}`

##### Email Verification Flow
1. User clicks verification link → `POST /api/v1/auth/verify-email` with `token`
2. Server hashes submitted token (SHA-256), looks up in `email_verifications` table
3. Validates: token exists, not expired, not already used
4. Updates user: `email_verified_at = NOW()`
5. Marks verification record as used
6. Redirects to login page with success message

##### Login Flow
1. User submits `email` + `password` to `POST /api/v1/auth/login`
2. Server retrieves user by email; returns generic "Invalid credentials" on not found (timing-safe via dummy `password_verify` call against a fixed hash to equalize timing)
3. Verifies password via `password_verify()`; checks `password_needs_rehash()` and re-hashes if algorithm params have changed
4. Checks `email_verified_at IS NOT NULL` — if not verified, returns 403 "Please verify your email first"
5. Checks `users.status = 'active'` (locked/suspended → 403 with appropriate message)
6. Checks account lockout state — see §3.1.7
7. Checks 2FA state via `two_factor_methods` (has any enabled method for this user):
   - **No 2FA enabled**: Issues full JWT pair immediately (2FA is recommended, not mandatory — see §3.1.3)
   - **2FA enabled**: Returns `{ requires_2fa: true, login_ticket: "...", available_methods: ["totp", "webauthn"] }` — a short-lived opaque ticket (5-min TTL, stored in Redis) that the client exchanges with a 2FA challenge response

##### 3.1.3 2FA Policy (Configurable)
- Default: 2FA is **strongly recommended** but not mandatory at first login. Users can dismiss the prompt and enable later.
- Admin-configurable per deployment: `auth.require_2fa = true` forces enrollment on first login (issues a `scope: '2fa_setup'` JWT until enrolled).
- 2FA is always available; only the enrollment requirement is configurable.

##### 2FA Verification (Login Completion)
1. User submits `login_ticket` + method response (e.g., `totp_code` or WebAuthn assertion) to `POST /api/v1/auth/verify-2fa`
2. Server validates login_ticket from Redis (exists, not expired)
3. Dispatches to method-specific validator (`TotpValidator`, `WebAuthnValidator`)
4. For TOTP: validates code against the user's enabled TOTP method secret using `pragmarx/google2fa` (±1 window tolerance); checks code hasn't been used before in the current window (replay prevention via Redis SET with TTL)
5. On success: Issues full JWT pair, clears login_ticket, updates `two_factor_methods.last_used_at`
6. On failure: Increments failed 2FA attempts; after 5 failures, invalidates login_ticket

##### 2FA Enrollment Flow (TOTP)
1. Authenticated user calls `POST /api/v1/auth/2fa/totp/setup`
2. Server generates TOTP secret, stores temporarily in Redis (5-min TTL, keyed by user_id)
3. Returns: `{ secret, qr_code_url, provisioning_uri }` — QR code rendered via `bacon/bacon-qr-code`
4. User scans QR with authenticator app
5. User submits two consecutive TOTP codes to `POST /api/v1/auth/2fa/totp/confirm`
6. Server validates both codes against the temp secret
7. On success: Encrypts secret (AES-256-GCM via SecretService) → inserts row into `two_factor_methods` with `method='totp'`, `enabled_at = NOW()`
8. Generates 8 single-use recovery codes (16 chars each, alphanumeric), stores hashed (SHA-256) in `recovery_codes` table
9. Returns recovery codes to user **once** — user must save them securely

##### 2FA Enrollment Flow (WebAuthn — Phase 4)
1. `POST /api/v1/auth/2fa/webauthn/register-challenge` returns a WebAuthn registration challenge
2. Browser invokes `navigator.credentials.create()` and POSTs the attestation
3. Server validates the attestation, stores the credential ID + public key in `two_factor_methods` with `method='webauthn'`

##### Recovery Code Flow
1. On 2FA prompt, user clicks "Use recovery code"
2. Submits `login_ticket` + `recovery_code` to `POST /api/v1/auth/verify-recovery`
3. Server hashes submitted code, checks against `recovery_codes` table (unused, belonging to user)
4. On match: marks code as used (`used_at = NOW()`), issues JWT pair
5. If remaining recovery codes < 3, response includes `warning: "You have few recovery codes remaining."`

##### Password Reset Flow
1. User submits `email` to `POST /api/v1/auth/forgot-password`
2. Server always returns 200 "If an account exists, a reset email has been sent" (no user enumeration)
3. If user exists: generates reset token (`random_bytes(32)` → hex), stores hashed (SHA-256) in `password_resets` table with 1-hour expiry
4. Sends email with reset link
5. User submits `token` + `new_password` to `POST /api/v1/auth/reset-password`
6. Server validates token, validates password strength
7. Updates password hash, **revokes all existing refresh tokens for the user**, marks reset token as used
8. User must log in again (with 2FA if enabled)

##### 3.1.7 Account Lockout & Brute-Force Protection

Two-tier protection:

**Tier 1 — Redis sliding window (operational, fast)**
- Per-email counter: `login_attempts:{email}` (15-min sliding window)
- Per-IP counter: `login_attempts:ip:{addr}` (15-min sliding window, 20-attempt threshold)
- Used for rate limiting and fast rejection

**Tier 2 — Database durable lockout (persistent)**
- `users.failed_login_attempts` incremented on each failure (atomic SQL UPDATE)
- `users.locked_until` set when threshold crossed:
  - 5 failures → locked 15 minutes
  - 10 cumulative failures → locked 1 hour
  - 20 cumulative failures → admin unlock required
- Reset on successful login (Tier 1 and Tier 2 both cleared)

Redis is consulted first; DB is the source of truth for sustained lockouts that must survive Redis flushes. The relationship: Redis is the fast path for rate decisions; DB carries durable state for cross-restart lockouts.

- **2FA attempts**: 5 failures per login_ticket → ticket invalidated
- **Notification**: Email sent to user on Tier 2 lockout event

#### 3.2 OAuth Authentication (Google, Apple, Microsoft)

##### OAuth 2.0 PKCE Flow
1. User clicks "Sign in with Google/Apple/Microsoft"
2. Server generates `code_verifier` (43-128 chars, base64url of random bytes), computes `code_challenge` (SHA-256, base64url)
3. Stores `code_verifier` + `state` (CSRF) + `nonce` in a short-lived OAuth session (Redis, 10-min TTL, keyed by an opaque session ID set in an `httpOnly Secure SameSite=Lax` cookie)
4. Redirects to provider's authorization endpoint with `code_challenge`, `code_challenge_method=S256`, `state`, `nonce`
5. Provider redirects back to `/api/v1/auth/callback/{provider}` with `code` + `state`
6. Server validates `state` against the OAuth session, exchanges `code` + `code_verifier` for tokens
7. Validates `nonce` in ID token, extracts user profile (email, name, avatar)
8. Upserts `users` + `oauth_accounts` records:
   - If email matches existing local-auth user → links OAuth account to existing user (only if email is verified on both sides)
   - If new user → creates user with `email_verified_at = NOW()` (OAuth emails trusted), `username = NULL` initially
9. **Username collection**: OAuth-only users are prompted to choose a username on first login (a one-time interstitial page); until set, `username IS NULL` is permitted, but most user-facing features that reference a username are gated
10. 2FA: same configurable policy as local auth — provider auth doesn't substitute for app-level 2FA, but it isn't required by default
11. Issues Weaver JWT pair

#### 3.3 Account Linking
- Users who registered locally can link OAuth accounts via `POST /api/v1/auth/link/{provider}` (requires active session + re-auth confirmation)
- Users who registered via OAuth can add a local password via `POST /api/v1/auth/set-password` (requires active session; sends verification email to confirm intent)
- A user can have multiple OAuth providers linked + a local password simultaneously
- Unlinking the last auth method is prevented (must always have at least one way to log in)
- `users.auth_method` is **not stored** — it's computed at read time from the presence of `password_hash` and `oauth_accounts` rows. Avoids denormalization drift.

#### 3.4 JWT Token Management & Delivery

**Token format (same for cookie and Bearer delivery)**
- **Algorithm**: RS256 (asymmetric — private key signs, public key verifies)
- **Access token**: 15-minute expiry; contains `user_id`, `system_role`, `scopes`, `auth_method`, `jti` (for revocation)
- **Refresh token**: 30-day expiry; opaque random string, stored hashed (SHA-256) in `refresh_tokens` table; rotated on each use
- **Key management**: Signing keys stored in SecretService (Vault/KMS); `kid` (key ID) in JWT header; old keys remain valid for verification during a 24h rotation window

**Delivery — Browser Frontend (first-party)**
- Access and refresh tokens delivered as `httpOnly Secure SameSite=Lax` cookies
- `SameSite=Lax` (not `Strict`) is required so OAuth callbacks and email-link landings can carry the session cookie
- **CSRF protection**: Double-submit cookie pattern — a non-`httpOnly` `csrf_token` cookie is set alongside the auth cookie; every state-changing request (POST/PUT/PATCH/DELETE) must include the token in the `X-CSRF-Token` header. The CSRF middleware compares header to cookie before dispatching to controllers. Synchronizer tokens are used for HTML form submissions (admin pages that aren't AJAX).

**Delivery — Programmatic API**
- API consumers send either an API key (preferred, per §3.6) or a JWT access token in the `Authorization: Bearer ...` header
- No cookie delivery to API consumers; no CSRF token required (header auth is not subject to CSRF)

**Refresh token rotation (race-safe)**
- Refresh uses an atomic update: `UPDATE refresh_tokens SET revoked_at = NOW(), replaced_by = :new_id WHERE id = :id AND revoked_at IS NULL` and checks the affected-row count
- If affected-row count is zero, the token was already used or revoked → treat as **token theft**: revoke the entire token family (all tokens chained through `replaced_by`) and force re-auth

**Logout**
- Refresh token deleted/revoked in DB
- Access token added to a Redis deny-list keyed by `jti` until its natural expiry (≤15 min)
- Cookies cleared with `Set-Cookie: ...; Max-Age=0`

#### 3.5 RBAC Model

| Role | Scope | Source | Permissions |
|------|-------|--------|-------------|
| `admin` | System | `users.system_role` | Full access, user management, feature flags, support impersonation |
| `owner` | Site | `site_members.role` | Full CRUD on owned sites, content, templates, plugins |
| `editor` | Site | `site_members.role` | Content CRUD on assigned sites; no site config, plugins, or members |
| `viewer` | Site | `site_members.role` | Read-only access to assigned sites |

- System role is stored as `users.system_role` (renamed from `role` for clarity)
- Site roles live in `site_members`
- Permissions enforced via `AuthorizationMiddleware` checking system role + site membership + resource ownership
- Owner-only actions on a site: change site config, manage members, install/configure plugins, delete site

#### 3.6 API Key Authentication
- External API uses `Authorization: Bearer wvr_...` header (API key preferred over JWT for machine-to-machine)
- Key generation: `random_bytes(32)` → base64url encode → prefix with `wvr_` for identification
- Storage: Hashed with SHA-256 in `api_keys.key_hash`; constant-time comparison (`hash_equals`)
- SHA-256 (a fast hash) is intentionally chosen: the 32-byte random key has enough entropy that offline brute force is infeasible even with a fast hash; fast hashing is needed because every API request looks the key up
- Scoped: Each key has JSON `scopes` array (e.g., `["sites:read", "content:write"]`)
- Rate limited separately from session auth
- Soft-revocation: `revoked_at TIMESTAMP NULL` rather than hard delete (preserves `last_used_at` and audit trail)

---

### 4. AI Integration Architecture

#### Provider Adapter Pattern
```
AIProviderInterface
├── OpenAIAdapter        (GPT-4o, GPT-4o-mini)
├── AnthropicAdapter     (Claude Sonnet, Haiku)
├── OllamaAdapter        (Local/self-hosted models)
└── CustomAdapter        (Plugin-provided via hook)
```

#### Interface Contract
```php
interface AIProviderInterface {
    public function generate(string $prompt, array $context, AIConfig $config): AIResponse;
    public function edit(string $content, string $instruction, AIConfig $config): AIResponse;
    public function summarize(string $content, AIConfig $config): AIResponse;
    public function generateMetadata(string $content, AIConfig $config): SEOMetadata;
}
```

#### Configuration (per-site, stored in `sites.config` JSON)
```yaml
ai:
  enabled: true
  provider: "openai"
  model: "gpt-4o-mini"
  api_key_ref: "secret://ai-keys/site-123"  # SecretService reference
  features:
    content_generation: true
    in_page_editing: true
    seo_generation: true
    alt_text: true
    translation: false
  limits:
    max_tokens_per_request: 4096
    max_requests_per_day: 100
  content_filters:
    block_explicit: true
    brand_voice_prompt: "Professional, concise, technical."
```

#### AI Capabilities
1. **Content Generation**: Generate draft pages/posts from prompts; outputs Markdown
2. **In-Page Editing**: Select text → choose action (rewrite, expand, simplify, translate) → AI returns suggestion → user accepts/rejects/edits → save
3. **SEO Metadata**: Auto-generate title tags, meta descriptions, OG tags from content
4. **Alt-Text**: Generate image alt-text from media uploads (vision-capable models)
5. **Translation**: Translate content to target language while preserving Markdown structure

#### 4.1 Prompt Injection Defense

Any AI feature that takes user-authored content (or third-party scraped content) as input must treat that input as untrusted data, not as instructions.

- **System prompts are hard-coded** in the adapter layer; users cannot modify them, only supply parameters (e.g., `brand_voice_prompt`) which are inserted into pre-defined positions in the system prompt
- **Content is wrapped in clear delimiters** (e.g., `<user_content>...</user_content>`) with explicit instruction in the system prompt to treat the wrapped block as data
- **Tool use is disabled** for all current AI features (no function/tool calls); if added later, tool calls are reviewed before execution
- **Output is sanitized** via HTMLPurifier before storage and rendering, regardless of the model's claimed format
- **Cross-content contamination**: When summarizing multiple content items, each is wrapped separately and the model is told they are independent
- **Audit**: Inputs and outputs of every AI call are logged (truncated) in `ai_usage_log` for review of injection attempts

#### Safety & Governance
- AI output sanitized through HTMLPurifier before storage
- AI-generated content marked with `ai_generated: true` in content metadata
- "Proposed changes" workflow: AI output presented as a diff; user must explicitly approve
- Usage tracked in `ai_usage_log` for billing/quota enforcement
- Configurable content filters (brand voice prompt, explicit content blocking)

---

### 5. Static Site Generation (Build Pipeline)

#### Build Flow
```
Trigger (manual/content_save/schedule/webhook)
    │
    ▼
Create build_job (status: queued, triggered_by_user_id) ──► Redis Streams queue
    │
    ▼
Worker picks job, acquires site lock (Redis SETNX, 2-min TTL, renewed every 60s)
    │
    ▼
Load site config + all content (batch query, eager-load relations)
    │
    ▼
Build to temp directory (storage/tmp/build-{uuid}/)
    │
    ├── For each content item (batch-processed):
    │   ├── Parse Markdown → HTML (league/commonmark)
    │   ├── Apply Twig template (layout → page → partials)
    │   ├── Execute plugin hooks (build.render_page)
    │   └── Write HTML to temp output path
    │
    ├── Generate static config files:
    │   ├── sitemap.xml
    │   ├── robots.txt
    │   ├── RSS/Atom feed
    │   ├── menus.json / navigation.json
    │   └── tags.json / metadata.json
    │
    ├── Copy static assets (CSS, JS, images, fonts)
    │   └── Asset fingerprinting (content hash in filename for cache busting)
    │
    ▼
Atomic swap: rename temp dir → site output dir
    │
    ▼
Git commit (with file lock, timeout 60s)
    │
    ▼
Deploy (configurable): local serve / rsync / S3 / webhook
    │
    ▼
Update build_job (status: success/failed, log, duration)
Release site lock
```

#### Build Lock Renewal
- Lock TTL is 2 minutes (not 10), aggressively renewed every 60 seconds by the worker via `EXPIRE` on the lock key
- A worker that crashes mid-build leaves the lock for at most ~2 minutes before it auto-expires
- Renewal runs in a separate Fiber (PHP 8.1+) so it isn't blocked by long synchronous build steps

#### Preview Renders (Synchronous)
- Editors need to see template/layout changes immediately without queueing a full build
- `POST /api/v1/sites/{site_id}/content/{content_id}/preview` renders a single content item using the live template kit, in-memory only, returns the HTML
- Preview renders skip Git commit, deploy, and the build queue entirely; they share template-loading code with the full build pipeline
- Preview renders are rate-limited per user

#### Incremental Builds
- Content hash table: `build_cache` stores `content_id → SHA-256(body + template_version + config_hash)`
- On build, compare current hashes to cached hashes; only re-render changed pages
- Full rebuild forced on: template change, config change, plugin change, or manual trigger

#### Concurrency & Failure Handling
- **Site-level lock**: Redis `SETNX` key `build:lock:{site_id}` with renewal pattern above
- **Duplicate suppression**: If build already queued/running, new trigger is coalesced (a "pending re-trigger" flag is set so a follow-up build runs when the current one finishes)
- **Transactional builds**: All output written to temp directory; atomic rename on success
- **Failure rollback**: On error, temp directory deleted; previous build remains live
- **Retry**: Failed builds retried up to 3 times with exponential backoff; then dead-letter state

---

### 6. Git Integration Strategy

#### Architecture
- Each site has its own Git repo at `storage/sites/{site_uuid}/repo/`
- Git operations via `GitService` wrapper class:
  - Validates all paths against site directory (no traversal)
  - Escapes all arguments via `escapeshellarg()`
  - Acquires file lock (`flock()`) per site before any operation
  - Sets timeout (30s) via `proc_open()` with `proc_terminate()` on timeout
  - Returns typed result objects (`GitCommit`, `GitDiff`, `GitLog`)

#### Operations
| Operation | When | Details |
|-----------|------|---------|
| `init` | Site creation | Initialize repo with `.gitignore` |
| `commit` | After successful build, after content save | Built output and content snapshots committed |
| `log` | Version history UI | Paginated commit log with diffs |
| `diff` | Compare versions | Changes between two commits |
| `show` | Restore body | `git show {hash}:content/{slug}.md` recovers historical body |
| `checkout` | Rollback | Checkout specific commit → trigger rebuild |
| `tag` | User-initiated | Named snapshots (e.g., "v1.0") |
| `branch` | Preview/staging | `main` (production) + `preview` (staging) |

#### Source of Truth & Body Reconstruction
- **Database is source of truth** for current content and config
- **Git stores both current and historical content bodies** — every content save commits the Markdown source to the per-site repo before any build
- **`content_versions` stores metadata only** — title, status, metadata JSON, `git_commit_hash`, `author_id`, `change_summary`. The body is reconstructed from Git via `git show {commit_hash}:content/{slug}.md` on demand (diffs, restore). This drops the per-version LONGTEXT storage cost dramatically.
- Periodic `git gc` via scheduled job to manage repository size
- **Future**: `php-git2`/libgit2 binding is on the roadmap (Phase 5+) to eliminate the per-op shell spawn cost — fine for now, but a bottleneck at scale

---

### 7. Plugin System Architecture

#### Trust Model (Honest)

PHP plugins running in-process cannot be meaningfully sandboxed at the language level — a plugin can call `file_get_contents`, `exec`, `eval`, hit the network, etc. Real isolation requires either subprocess execution or a WASM runtime. Weaver takes a phased approach:

- **Phase 3 (plugin launch)**: **Trusted-only plugins**. The plugin marketplace requires manual review + code signing before listing. Self-installed plugins (via the `plugins/` directory) are explicitly the site operator's responsibility, with a UI warning at install time. The "PluginAPI" service is a *convenience* layer for plugins, not a security boundary.
- **Phase 5**: **Subprocess isolation** for community plugins. Hooks execute in a forked worker with `disable_functions`, `open_basedir`, and a restricted environment. The PluginAPI then becomes a real boundary, exposed via stdin/stdout IPC.
- **WASM-based plugins** are evaluated as a longer-term alternative (e.g., Extism PHP SDK) but not committed to.

The current §10 claim of "plugin sandboxing" refers to (a) marketplace gating, (b) capability declaration in the manifest with runtime enforcement of which PluginAPI methods a plugin may call, (c) hook timeouts, and (d) circuit-breaker auto-disable on misbehavior. It does not refer to language-level execution isolation until Phase 5.

#### Hook-Based Event System
```php
$dispatcher->addFilter('content.before_save', function(Content $content): Content {
    return $content;
}, priority: 10);

$dispatcher->addAction('build.after', function(BuildResult $result): void {
    // Post-build notification
});
```

#### Plugin Manifest (`plugin.json`)
```json
{
  "name": "weaver-seo-plugin",
  "version": "1.2.0",
  "description": "Advanced SEO tools",
  "author": "Weaver Community",
  "requires": { "weaver": ">=1.0.0", "php": ">=8.4" },
  "hooks": ["content.before_save", "build.render_page", "template.head"],
  "config_schema": {
    "auto_sitemap": { "type": "boolean", "default": true }
  },
  "permissions": ["content:read", "media:read"],
  "signature": "..."
}
```

#### Lifecycle
`discover` → `verify_signature` (Phase 3+) → `install` (validate manifest, register in DB) → `activate` (load hooks) → `configure` (per-site config) → `deactivate` → `uninstall` (cleanup)

#### Runtime Controls
- **Hook timeout**: 5-second timeout per hook execution via `pcntl_alarm` where available; on platforms without `pcntl` (e.g., default FPM), timeouts are advisory and logged but not strictly enforced — documented limitation
- **Circuit breaker**: 3 failures in 10 minutes → plugin auto-disabled for that site, owner notified
- **Permissions**: Plugin manifest declares required permissions; the `PluginAPI` service checks the calling plugin's manifest before exposing data
- **Plugin context propagation**: Each hook invocation passes a `PluginContext` object identifying the plugin; PluginAPI methods inspect it for permission checks

#### Core Plugin Hooks
| Hook | Type | Description |
|------|------|-------------|
| `content.before_save` | Filter | Modify content before DB write |
| `content.after_save` | Action | React to content save |
| `build.before` | Action | Pre-build setup |
| `build.render_page` | Filter | Modify rendered HTML per page |
| `build.after` | Action | Post-build actions |
| `template.head` | Filter | Inject into `<head>` — output passed through HTMLPurifier |
| `template.footer` | Filter | Inject before `</body>` — output passed through HTMLPurifier |
| `api.response` | Filter | Modify API responses |
| `media.after_upload` | Action | Process uploaded media |
| `ai.after_generate` | Filter | Post-process AI output |
| `auth.after_login` | Action | React to user login |
| `auth.after_register` | Action | React to new registration |

#### Plugin Output Filtering
- Filter hooks that return content destined for the rendered page (`template.head`, `template.footer`, `build.render_page`) have their return values run through HTMLPurifier with a permissive-but-bounded whitelist before injection
- Plugins that need to bypass purification (e.g., a structured-data plugin generating JSON-LD) declare the `unsafe_html` permission in their manifest; this permission is gated behind admin approval

---

### 8. Template Engine (Twig)

**Decision: Use Twig** — mature, compiled, secure (sandbox mode), extensible, PHP 8.4 support. Building a custom engine is high-risk, low-value.

#### Storage: Filesystem, Indexed in DB
- Template files live on the filesystem at `templates/{theme-slug}/{version}/...`
- The `templates` table holds metadata (name, slug, version, type, author, config); the actual files are not stored in MySQL
- Rationale: faster reads, editor-friendly, naturally version-controllable; LONGTEXT storage was the wrong primitive
- For multi-tenancy: user-created templates live in `templates/user-{user_id}/{slug}/{version}/...`; the user_id directory is isolated via the same path-validation logic as the Git wrapper

#### Template Kit Structure
```
templates/
└── theme-slug/
    └── 1.2.0/                  # Version directory
        ├── config.json         # Theme metadata, regions, defaults
        ├── layouts/
        │   ├── default.html.twig
        │   └── blog.html.twig
        ├── partials/
        │   ├── header.html.twig
        │   ├── footer.html.twig
        │   └── sidebar.html.twig
        ├── pages/
        │   ├── page.html.twig
        │   └── post.html.twig
        └── assets/
            ├── css/
            ├── js/
            └── images/
```

#### Theme Versioning per Site
- `sites.theme_id` references `templates.id`; `sites.theme_version` pins the version
- Site builds always use the pinned version; theme updates do not automatically rebuild dependent sites
- A "theme update available" notification surfaces in the site dashboard; site owner triggers the update + rebuild explicitly

#### Template Variables
```twig
{{ site.name }}, {{ site.config.navigation }}
{{ content.title }}, {{ content.body }}
{{ content.metadata.seo_title }}
{{ content.published_at | date('Y-m-d') }}
{% for item in navigation %}...{% endfor %}
{% block content %}...{% endblock %}
```

#### User-Submitted Template Safety
- Twig sandbox mode enabled for user-created templates (`type='user'`)
- Whitelist of allowed tags, filters, and functions
- Template syntax validation on upload
- System templates (`type='system'`) and audited marketplace templates run without sandboxing for full Twig expressiveness

---

### 9. Secret Management

A dedicated `SecretService` abstracts the storage and retrieval of all sensitive material:

- **OAuth client secrets** (per-provider, global)
- **OAuth refresh/access tokens** (per-user, encrypted at rest)
- **TOTP secrets** (per-2FA-method row, encrypted at rest)
- **Per-site AI API keys**
- **JWT signing keys** (RS256 private key, with `kid` versioning)
- **Encryption keys** (master keys for envelope encryption of the above)

#### Backends
- **Production**: HashiCorp Vault or AWS Secrets Manager / KMS. The application holds only a Vault token / IAM role and fetches secrets at startup or on demand (with short-TTL in-memory cache).
- **Development**: Encrypted local file (libsodium sealed box) protected by a single dev-key in the local `.env`. Documented as dev-only.
- **References**: Throughout the codebase, secrets are referenced by URI (`secret://ai-keys/site-123`) and resolved via SecretService — no plaintext secrets ever appear in config files committed to source control.

#### Envelope Encryption
- Data Encryption Keys (DEKs) per secret are AES-256-GCM; DEKs are themselves encrypted by a Key Encryption Key (KEK) held in Vault/KMS
- `encryption_key_version` columns on encrypted records identify which KEK version protects the row, enabling rotation without re-encrypting everything atomically
- A `weaver secrets:rotate` CLI command re-encrypts records in batches

#### Threat Model
- Database compromise alone does not yield plaintext OAuth tokens, TOTP secrets, or AI keys (KEK lives in Vault/KMS)
- Application-server compromise yields whatever is in process memory at the time (short-TTL cache limits blast radius)
- Vault/KMS compromise is treated as game-over; defense is operational (access policies, audit logs)

---

### 10. Email Infrastructure

#### Transport
- Symfony Mailer with a pluggable DSN: SES, Postmark, SendGrid, or SMTP
- **Recommended for production**: Postmark (for transactional reliability) or SES (for cost)
- Per-environment configuration; no provider lock-in

#### Bounce & Complaint Handling
- Webhook endpoint `/api/v1/webhooks/email/{provider}` receives bounce/complaint notifications
- Hard bounces mark the user's email as `email_status='invalid'`; subsequent verification/reset emails fail fast with an in-app notice asking the user to update their email
- Complaints (spam reports) mark the address as `email_status='complained'` and suppress further transactional email until the user acknowledges in-app

#### Email Templates
- Stored in `templates/email/` as Twig templates
- All emails: transactional only, no marketing; one-click "this wasn't me" / report link for security-relevant emails (login from new device, password changed, etc.)

#### Deliverability
- SPF, DKIM, DMARC must be configured for the sending domain (documented in ops runbook)
- Unique unsubscribe header for any non-transactional mail (not currently sent, but headers reserved)

---

### 11. Feature Flags

```yaml
# config/features.yaml
features:
  local_auth: { default: true, phase: 1 }
  two_factor_auth: { default: false, phase: 2 }
  oauth_google: { default: false, phase: 2 }
  oauth_apple: { default: false, phase: 2 }
  oauth_microsoft: { default: false, phase: 2 }
  ai_integration: { default: false, phase: 3 }
  plugin_system: { default: false, phase: 3 }
  multi_site: { default: false, phase: 2 }
  external_api: { default: false, phase: 4 }
  webauthn: { default: false, phase: 4 }
  git_branches: { default: false, phase: 4 }
  plugin_sandbox: { default: false, phase: 5 }
  template_marketplace: { default: false, phase: 5 }
```

- `config/features.yaml` defines all flags with defaults
- `feature_flags` DB table for runtime overrides (per-user or global)
- `FeatureService::isEnabled(string $flag, ?int $userId = null): bool`
- Admin UI to toggle flags

---

### 12. Security

#### Password Security
- **Hashing**: Argon2id via `password_hash()` (memory_cost: 65536, time_cost: 4, threads: 3)
- **Rehashing**: Automatic on login via `password_needs_rehash()`
- **Validation**: Minimum 12 characters; HIBP breached-password check (k-anonymity)
- **Storage**: Only hash stored; plaintext never logged or cached
- **Timing-safe failure**: Login failures for non-existent users dummy-verify against a fixed hash to equalize response time

#### 2FA Security
- TOTP secrets encrypted at rest (AES-256-GCM via SecretService) with versioned KEKs
- TOTP window tolerance: ±1 (30-second window each side)
- Replay prevention: used codes tracked in Redis SET with 90-second TTL
- Recovery codes: 8 single-use codes, SHA-256 hashed in DB, shown once at enrollment
- WebAuthn (Phase 4): credential public keys stored unencrypted (no secret material), per spec

#### Authentication Security
- OAuth 2.0 PKCE with `state` (CSRF) + `nonce` validation
- JWT RS256 with key rotation (24h overlap window)
- Refresh token rotation with theft detection (token family revocation)
- Session fixation prevention: regenerate session on login

#### CSRF Protection
- Browser frontend: double-submit cookie pattern (`csrf_token` cookie + `X-CSRF-Token` header)
- HTML forms: synchronizer token pattern
- API consumers (Bearer header): not vulnerable, no CSRF token required

#### Data Encryption
- All sensitive material managed via SecretService (see §9)
- `encryption_key_version` column on encrypted records for rotation
- Master keys never leave Vault/KMS

#### Input/Output Security
- All user input sanitized; HTML purified via HTMLPurifier
- AI output sanitized identically
- Plugin filter-hook output passed through HTMLPurifier unless plugin holds `unsafe_html` permission
- SQL injection: PDO prepared statements exclusively
- XSS: Twig auto-escaping on all template output; sandbox mode for user templates

#### File Security
- Uploads stored outside web root (`storage/uploads/`)
- MIME type validation (finfo), extension whitelist, max size enforcement
- UUID-prefixed filenames; `realpath()` validation; symlinks disallowed
- Content-hash dedup: identical files reference a single stored blob (`media.content_hash`)

#### Rate Limiting
- Token bucket algorithm via Redis
- Defaults: 60 req/min (authenticated), 20 req/min (unauthenticated), 10 req/min (AI), 5 req/min (login/register), 30 req/min (preview render)
- Response headers: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`

---

### 13. Performance Optimization

- **Redis caching**: Compiled templates, API responses (5-min TTL), site config, sessions
- **Incremental builds**: Content hash diffing → only rebuild changed pages
- **Batch DB loading**: Build pipeline loads all content in 1-2 queries (eager-load)
- **Lazy objects** (PHP 8.4): Defer plugin initialization until first hook call
- **Background jobs**: All builds async via Redis Streams workers
- **CDN-ready output**: Content-hash filenames for infinite caching
- **Database indexes**: Composite indexes on all common query patterns
- **Connection pooling**: PDO persistent connections; Redis connection reuse

---

### 14. Observability & Operations

#### Logging
- Structured JSON logs via Monolog → `storage/logs/` (with rotation to S3 in production)
- Request correlation ID propagated through all entries
- Auth events logged: login (success/failure), logout, 2FA setup/failure, password reset, account lockout, admin impersonation

#### Monitoring
- Prometheus `/metrics` endpoint: build duration, queue depth, API latency, AI usage, auth failure rate, lock contention, secret cache hit rate
- Health check: `GET /health` → DB, Redis, disk space, Git availability, SecretService reachability, mail transport

#### Backups
- **Database**: Daily mysqldump + weekly full backup; 30-day retention; binlog for PITR
- **Git repos**: Daily rsync to backup storage
- **Media files**: Daily sync to secondary storage (S3 or backup volume)
- **Secret backups**: Vault/KMS native snapshot
- **RTO**: 4 hours | **RPO**: 1 hour

#### Database Migrations
- **Tool**: Phinx migration framework
- Zero-downtime strategy: add-only migrations first, backfill, then enforce constraints
- CLI: `weaver migrate:run`, `weaver migrate:rollback`, `weaver migrate:status`

#### Worker Supervision
- Redis Streams consumers run as long-lived processes under `systemd` (or as containers under Docker/Kubernetes in production)
- Each worker has a `weaver-worker@.service` unit; restart on failure with exponential backoff
- Workers expose a `/worker/healthz` endpoint for liveness checks
- Graceful shutdown on SIGTERM: finish in-flight job, release lock, exit

#### Redis High Availability
- Production deployments use Redis Sentinel (3+ sentinels, 1 primary + 2 replicas) or Redis Cluster
- `predis/predis` is Sentinel-aware; failover is transparent
- Critical impact assessment: login_ticket and OAuth-session data live in Redis. If Redis is unreachable, login is blocked. This is an accepted tradeoff for the simplicity of using Redis as the single ephemeral store; HA mitigates the risk.

#### Time Zones & Charset
- All `TIMESTAMP` columns assume UTC; MySQL server `time_zone='+00:00'` set explicitly in config
- All tables use `utf8mb4` charset, `utf8mb4_0900_ai_ci` collation
- Application timezone for display is per-user, stored in `users.preferences.timezone`

---

### 15. GDPR & Data Lifecycle

#### Right to Access (Data Export)
- `POST /api/v1/users/me/export` enqueues an export job
- Worker assembles a ZIP containing: profile, sites, content (Markdown sources + metadata), media references, audit log entries
- Download link delivered via email; link expires in 7 days; ZIP encrypted with a one-time password sent separately

#### Right to Erasure (Account Deletion)
- `POST /api/v1/users/me/delete` initiates a 30-day grace period (`users.deletion_scheduled_at`)
- During grace period, account is suspended but recoverable
- After 30 days: user record anonymized (email/username/displayname replaced with placeholders, password hash cleared, OAuth links removed), owned sites and content either transferred to a designated co-owner or deleted per the user's choice during the request
- Audit log entries are retained but `user_id` is set to NULL with an anonymized reference

#### Data Retention
- Audit log: 1 year by default, configurable
- Build logs: 90 days by default
- AI usage logs: 1 year (for billing); aggregated for longer
- Deleted content: 30-day soft delete via `deleted_at`, then purged

#### Consent & Lawful Basis
- Cookie consent banner for non-essential cookies (analytics if enabled)
- Privacy policy linked from registration; acceptance recorded in `audit_log`
- Data processing agreement (DPA) available for business customers

#### Soft Delete
- All major entities (sites, content, media) have `deleted_at TIMESTAMP NULL`
- Trash bin UI allows recovery within 30 days
- Hard delete is a separate, audit-logged action (or automatic after retention period)

---

### 16. API Documentation

- OpenAPI 3.1 spec generated from controller annotations via `zircote/swagger-php`
- Spec served at `/api/v1/openapi.json` and rendered as Swagger UI at `/api/v1/docs`
- Spec is part of CI: a breaking change requires explicit version bump (`/api/v2/...` for incompatible changes)
- External API consumers receive a generated SDK skeleton from the spec (TypeScript + PHP at launch)

---

### 17. Testing Strategy

#### Unit Tests
- Service-layer business logic; mocked repositories
- Coverage target: ≥80% on services, ≥60% overall

#### Integration Tests
- Real MySQL (Testcontainers or local Docker) + real Redis
- Per-test transaction rollback for isolation
- **Mandatory multi-tenant isolation suite**: every API endpoint tested with two distinct tenants, asserting that A cannot access B's resources

#### E2E Tests
- Playwright against a fully-running stack (Docker Compose for CI)
- Coverage: auth flows (registration, login, 2FA, reset), content CRUD, build trigger, plugin install

#### Security Testing
- Static analysis: Psalm at level 1, PHPStan at level 8
- Dependency vulnerability scanning: Composer Audit + Dependabot
- Annual third-party penetration test (Phase 5+)

---

### 18. Implementation Phases

**Phase 1 — MVP (~8 weeks)**
- Core framework: PSR-15 middleware, DI container, routing, error handling
- Local auth: registration (enumeration-resistant), email verification, login, Argon2id hashing
- Account lockout (Redis + DB two-tier)
- JWT token management (RS256, refresh rotation with theft detection, cookie + Bearer delivery)
- CSRF middleware (double-submit cookie)
- Single-user, single-site content CRUD (Markdown + HTML)
- Twig template engine (filesystem-backed) + 1 default theme
- Basic static build pipeline (full builds)
- Git integration (init, commit, log, show for body reconstruction)
- Preview render path
- Internal API for CMS frontend
- SecretService (with dev backend; Vault integration prepared but not required)
- Mail transport (Postmark/SES) + bounce webhook stub
- Database schema v1 + Phinx migration framework
- Feature flag system
- Structured logging + health check
- OpenAPI spec generation
- Multi-tenant isolation test suite

> **Scope note**: 2FA and OAuth providers are deliberately deferred from Phase 1 to make the timeline realistic. Phase 1 ships a secure, fully-functional local-auth CMS for early adopters.

**Phase 2 — Auth Expansion, Multi-Tenancy & Polish (~6 weeks)**
- 2FA (TOTP) enrollment, verification, recovery codes — opt-in by default; admin-configurable mandatory enrollment
- Google, Apple, Microsoft OAuth providers (PKCE)
- OAuth ↔ local account linking; OAuth-user username collection flow
- Multi-site support + tenant isolation hardening
- RBAC (owner, editor, viewer roles); site_members
- Media management (upload, library, in-content embedding, content-hash dedup)
- Incremental builds
- Site configuration UI (menus, metadata, tags as JSON/YAML)
- API pagination + filtering
- Password reset flow
- Audit log UI

**Phase 3 — AI & Plugins (~6 weeks)**
- AI integration: OpenAI adapter + in-page editing UI
- AI content generation + SEO metadata generation
- Prompt injection defense (system prompt hardening, delimiter wrapping, output sanitization)
- Plugin system: hook framework, loader, manifest validation, capability enforcement
- **Trusted-only plugin model** (no language-level sandboxing yet; marketplace review + signing)
- 3 first-party plugins: SEO, Analytics snippet injection, Contact form
- Anthropic adapter + Ollama (local) adapter
- AI usage tracking + quota enforcement

**Phase 4 — External API, WebAuthn & Advanced Git (~4 weeks)**
- External API + API key management + developer docs
- WebAuthn/passkeys as a 2FA method (and as a primary auth method, opt-in)
- Git branches (preview/staging), tags, rollback UI
- Deploy targets: S3, rsync, Netlify/Vercel/Cloudflare webhooks
- Webhook system (incoming + outgoing) with signed payloads
- Build scheduling (cron-based)
- Custom domain verification (DNS TXT challenge + ACME for TLS)

**Phase 5 — Hardening & Ecosystem (~6 weeks)**
- Security audit (OWASP Top 10, third-party pentest)
- Load testing + performance optimization
- **Plugin subprocess isolation** (real sandboxing for community plugins)
- libgit2 binding evaluation (replace shell_exec for hot paths)
- Plugin SDK + developer documentation
- Template marketplace foundation
- Encryption key rotation tooling (production-ready CLI)
- Monitoring dashboard (Prometheus + Grafana)
- User documentation + onboarding flow
- GDPR data export + erasure flow productionized

---

### 19. Technical Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Build pipeline slow at scale (10k+ pages) | High | Batch DB queries, parallel rendering, aggressive caching, profiling from Phase 1, libgit2 in Phase 5 |
| Git repo size growth | Medium | Media excluded from Git; periodic `git gc`; configurable commit retention |
| AI provider downtime / cost spikes | Medium | Circuit breaker, fallback providers, strict token limits, graceful degradation |
| Prompt injection via user content | High | System prompt hard-coding, input delimiters, tool-use disabled, output sanitization |
| Malicious/buggy plugins (Phase 3) | High | Marketplace review + signing; capability manifest; hook timeouts; circuit breaker; explicit "trusted code" warnings |
| Malicious/buggy plugins (Phase 5+) | High | Subprocess isolation with `disable_functions` + `open_basedir`; capability-based IPC |
| Multi-tenant data leakage | Critical | `BaseTenantRepository` enforces scoping; mandatory isolation test suite; security audit in Phase 5 |
| OAuth provider API changes | Low | `league/oauth2-client` abstraction; provider code isolated in adapters |
| Brute-force attacks on local auth | High | Argon2id, Redis + DB two-tier lockout, IP rate limiting, HIBP password checking |
| 2FA secret compromise | High | AES-256-GCM encryption via SecretService; KEK in Vault/KMS; recovery codes hashed |
| Refresh token theft | Medium | Atomic rotation with family-revocation on detected reuse |
| Redis outage blocks login | Medium | Redis Sentinel/Cluster HA; documented as accepted dependency |
| Secret store (Vault/KMS) outage | High | Short-TTL in-memory cache; degraded mode allows existing sessions but blocks new logins |
| Email deliverability failures | Medium | Bounce/complaint webhooks; per-domain SPF/DKIM/DMARC; transactional-only sending |

**Rollback Strategy**
- **Code**: Git-tagged releases; rollback = deploy previous tag
- **Database**: Phinx `migrate:rollback`; add-then-backfill pattern ensures backward compatibility
- **Content**: Git-based rollback per site; content_versions metadata + Git body recovery for granular recovery
- **Features**: Instant disable via feature flags without deployment

---

### 20. Alternatives Considered

| Decision | Chosen | Alternative | Trade-off |
|----------|--------|-------------|-----------|
| Template Engine | Twig | Custom engine | Custom = full control but 4-6 weeks effort + security risk; Twig is battle-tested |
| Template Storage | Filesystem + DB metadata | LONGTEXT in DB | Filesystem is faster, editor-friendly, naturally version-controllable; DB-LONGTEXT made deploy and editing harder |
| Git Integration | Hardened shell_exec (Phase 1-4), libgit2 evaluation (Phase 5+) | Pure php-git2 from day one | Shell is universally available; libgit2 binding adds compile complexity, deferred until proven bottleneck |
| Queue System | Redis Streams | RabbitMQ / DB queue | Redis already in stack; Streams has consumer groups + dead-letter |
| AI Integration | Direct API adapters | LangChain-like orchestrator | No mature PHP equivalent; direct adapters are simpler and sufficient |
| Deploy Strategy | Pluggable targets | Built-in CDN | Pluggable = maximum flexibility |
| Framework | Custom PSR stack | Laravel / Symfony | PSR stack is leaner; cherry-pick Symfony components |
| Password Hashing | Argon2id | bcrypt | Argon2id is memory-hard; OWASP recommended; PHP 8.4 native support |
| 2FA Method (Phase 2) | TOTP + recovery codes | WebAuthn / SMS | TOTP universal; WebAuthn added in Phase 4; SMS insecure |
| Plugin Sandboxing | Trusted-only (Phase 3) → subprocess (Phase 5) | Day-one WASM sandboxing | WASM in PHP is immature; phased approach delivers value while building real isolation |
| Content Versioning | Metadata + Git body recovery | Full LONGTEXT per version | DB-only versioning balloons; Git already exists in the stack |
| `auth_method` storage | Computed at read time | Denormalized ENUM column | Eliminates drift between `password_hash`/`oauth_accounts` and a stored summary |
| JWT delivery | Cookies (frontend) + Bearer (API) | Cookies only / Bearer only | Cookies are safer for browsers; Bearer is standard for APIs; same token format both ways |
| `SameSite` cookie attribute | `Lax` | `Strict` | `Strict` breaks OAuth callbacks and email-link landings |
| Secret storage | SecretService (Vault/KMS in prod) | Env vars | Env vars leak through process listings, dumps, and error reports |

## DB Changes

### Full Schema (MySQL 8.0+, utf8mb4_0900_ai_ci)

#### users
| Column | Type | Notes |
|--------|------|-------|
| id | BIGINT UNSIGNED PK AUTO_INCREMENT | |
| uuid | CHAR(36) UNIQUE | Public identifier |
| email | VARCHAR(255) UNIQUE | |
| email_status | ENUM('active','invalid','complained') DEFAULT 'active' | Bounce/complaint tracking |
| username | VARCHAR(30) UNIQUE NULL | NULL for OAuth-only users pending selection |
| password_hash | VARCHAR(255) NULL | Argon2id; NULL for OAuth-only users |
| display_name | VARCHAR(255) | |
| avatar_url | VARCHAR(500) NULL | |
| system_role | ENUM('admin','user') DEFAULT 'user' | System-level role (renamed from `role`) |
| status | ENUM('active','locked','suspended') DEFAULT 'active' | Account status; `pending_verification` derived from `email_verified_at IS NULL` |
| email_verified_at | TIMESTAMP NULL | NULL = not yet verified |
| preferences | JSON | UI preferences, locale, timezone |
| quota_tier | VARCHAR(50) DEFAULT 'free' | FK→user_quotas.tier (logical, not enforced) |
| failed_login_attempts | INT DEFAULT 0 | Durable counter (Tier 2 lockout); reset on successful login |
| locked_until | TIMESTAMP NULL | Durable lockout expiry |
| password_changed_at | TIMESTAMP NULL | |
| deletion_scheduled_at | TIMESTAMP NULL | GDPR 30-day grace period |
| deleted_at | TIMESTAMP NULL | Soft delete (post-grace anonymization marker) |
| created_at | TIMESTAMP | |
| updated_at | TIMESTAMP | |

Note: `auth_method` is **not** stored. It is computed at query time as: `oauth' if password_hash IS NULL else ('both' if EXISTS oauth_accounts else 'local')`.

#### two_factor_methods
| Column | Type | Notes |
|--------|------|-------|
| id | BIGINT UNSIGNED PK | |
| user_id | BIGINT FK→users ON DELETE CASCADE | |
| method | ENUM('totp','webauthn') | Extensible enum |
| label | VARCHAR(100) NULL | User-friendly name (e.g., "iPhone Authenticator", "YubiKey 5") |
| secret_enc | TEXT NULL | AES-256-GCM (TOTP only) |
| credential_id | VARBINARY(255) NULL | WebAuthn credential ID |
| public_key | TEXT NULL | WebAuthn public key (COSE encoded) |
| sign_count | INT UNSIGNED NULL | WebAuthn counter |
| encryption_key_version | INT DEFAULT 1 | |
| enabled_at | TIMESTAMP | |
| last_used_at | TIMESTAMP NULL | |
| created_at | TIMESTAMP | |
| INDEX(user_id, method) | | |

#### oauth_accounts
| Column | Type | Notes |
|--------|------|-------|
| id | BIGINT UNSIGNED PK | |
| user_id | BIGINT FK→users ON DELETE CASCADE | |
| provider | ENUM('google','apple','microsoft') | |
| provider_user_id | VARCHAR(255) | |
| access_token_enc | TEXT | AES-256-GCM via SecretService |
| refresh_token_enc | TEXT NULL | |
| encryption_key_version | INT DEFAULT 1 | |
| token_expires_at | TIMESTAMP NULL | |
| created_at | TIMESTAMP | |
| UNIQUE(provider, provider_user_id) | | |

#### email_verifications
| Column | Type | Notes |
|--------|------|-------|
| id | BIGINT UNSIGNED PK | |
| user_id | BIGINT FK→users ON DELETE CASCADE | |
| token_hash | CHAR(64) | SHA-256 hash of token |
| expires_at | TIMESTAMP | 24-hour expiry |
| used_at | TIMESTAMP NULL | |
| created_at | TIMESTAMP | |
| INDEX(token_hash) | | |

#### password_resets
| Column | Type | Notes |
|--------|------|-------|
| id | BIGINT UNSIGNED PK | |
| user_id | BIGINT FK→users ON DELETE CASCADE | |
| token_hash | CHAR(64) | SHA-256 hash |
| expires_at | TIMESTAMP | 1-hour expiry |
| used_at | TIMESTAMP NULL | |
| created_at | TIMESTAMP | |
| INDEX(token_hash) | | |

#### recovery_codes
| Column | Type | Notes |
|--------|------|-------|
| id | BIGINT UNSIGNED PK | |
| user_id | BIGINT FK→users ON DELETE CASCADE | |
| code_hash | CHAR(64) | SHA-256 hash |
| used_at | TIMESTAMP NULL | |
| created_at | TIMESTAMP | |
| INDEX(user_id, used_at) | | |

#### refresh_tokens
| Column | Type | Notes |
|--------|------|-------|
| id | BIGINT UNSIGNED PK | |
| user_id | BIGINT FK→users ON DELETE CASCADE | |
| token_hash | CHAR(64) | SHA-256 hash |
| expires_at | TIMESTAMP | 30-day expiry |
| revoked_at | TIMESTAMP NULL | |
| replaced_by | BIGINT NULL | Self-reference for family tracking |
| created_at | TIMESTAMP | |
| INDEX(token_hash) | | |
| INDEX(user_id, revoked_at) | | |

#### user_quotas
| Column | Type | Notes |
|--------|------|-------|
| tier | VARCHAR(50) PK | e.g., 'free', 'pro', 'enterprise' |
| max_sites | INT UNSIGNED | |
| max_storage_mb | INT UNSIGNED | |
| max_ai_tokens_per_month | INT UNSIGNED | |
| max_api_calls_per_day | INT UNSIGNED | |
| max_build_minutes_per_month | INT UNSIGNED | |
| updated_at | TIMESTAMP | |

#### sites
| Column | Type | Notes |
|--------|------|-------|
| id | BIGINT UNSIGNED PK | |
| uuid | CHAR(36) UNIQUE | Public identifier |
| user_id | BIGINT FK→users | Owner |
| slug | VARCHAR(100) | URL-safe identifier |
| name | VARCHAR(255) | |
| description | TEXT NULL | |
| domain | VARCHAR(255) NULL | Custom domain |
| domain_verified_at | TIMESTAMP NULL | Set when DNS challenge passes |
| config | JSON | Menus, metadata, tag clouds, build settings |
| theme_id | BIGINT FK→templates NULL | |
| theme_version | VARCHAR(20) NULL | Pinned theme version |
| status | ENUM('active','suspended','archived') | |
| git_repo_path | VARCHAR(500) | |
| deleted_at | TIMESTAMP NULL | Soft delete |
| created_at | TIMESTAMP | |
| updated_at | TIMESTAMP | |
| UNIQUE(user_id, slug) | | Slug uniqueness scoped to user |
| INDEX(user_id, status) | | |
| INDEX(domain) | | For domain lookup during request routing |

#### site_members
| Column | Type | Notes |
|--------|------|-------|
| id | BIGINT UNSIGNED PK | |
| site_id | BIGINT FK→sites ON DELETE CASCADE | |
| user_id | BIGINT FK→users ON DELETE CASCADE | |
| role | ENUM('owner','editor','viewer') | |
| created_at | TIMESTAMP | |
| UNIQUE(site_id, user_id) | | |

#### content
| Column | Type | Notes |
|--------|------|-------|
| id | BIGINT UNSIGNED PK | |
| uuid | CHAR(36) UNIQUE | |
| site_id | BIGINT FK→sites ON DELETE CASCADE | |
| slug | VARCHAR(255) | |
| title | VARCHAR(500) | |
| body_markdown | LONGTEXT | Current body (also committed to Git) |
| body_html | LONGTEXT | Rendered from Markdown |
| content_type | ENUM('page','post','partial','layout') | |
| status | ENUM('draft','review','published','archived') | |
| metadata | JSON | SEO, OG tags, custom fields, ai_generated flag |
| author_id | BIGINT FK→users | |
| current_git_commit | CHAR(40) NULL | Latest commit hash for this content |
| published_at | TIMESTAMP NULL | |
| deleted_at | TIMESTAMP NULL | Soft delete |
| created_at | TIMESTAMP | |
| updated_at | TIMESTAMP | |
| UNIQUE(site_id, slug) | | |
| INDEX(site_id, status, published_at) | | |
| FULLTEXT(title, body_markdown) | | |

#### content_versions
| Column | Type | Notes |
|--------|------|-------|
| id | BIGINT UNSIGNED PK | |
| content_id | BIGINT FK→content ON DELETE CASCADE | |
| version_number | INT | Auto-incrementing per content item |
| title | VARCHAR(500) | Snapshot of title at this version |
| status | ENUM('draft','review','published','archived') | Status at this version |
| metadata | JSON | Snapshot of metadata at this version |
| git_commit_hash | CHAR(40) | **Required** — body reconstructed via `git show {hash}:content/{slug}.md` |
| change_summary | VARCHAR(500) NULL | |
| author_id | BIGINT FK→users | |
| created_at | TIMESTAMP | |
| INDEX(content_id, version_number) | | |

Note: Body is no longer stored per-version; it is reconstructed from Git on demand for diffs/restore. Drops storage cost dramatically for high-edit-frequency content.

#### templates
| Column | Type | Notes |
|--------|------|-------|
| id | BIGINT UNSIGNED PK | |
| uuid | CHAR(36) UNIQUE | |
| name | VARCHAR(255) | |
| slug | VARCHAR(100) | Globally unique for `type='system'`/`'marketplace'`; user-scoped for `type='user'` |
| description | TEXT NULL | |
| type | ENUM('system','user','marketplace') | |
| author_id | BIGINT FK→users NULL | |
| version | VARCHAR(20) | |
| config | JSON | Regions, slots, defaults |
| filesystem_path | VARCHAR(500) | Relative path under `templates/` |
| is_active | BOOLEAN DEFAULT TRUE | |
| created_at | TIMESTAMP | |
| updated_at | TIMESTAMP | |
| UNIQUE(slug, type, author_id) | | Composite uniqueness |

Note: Template files live on the filesystem; `template_files` table from v1 is removed.

#### plugins
| Column | Type | Notes |
|--------|------|-------|
| id | BIGINT UNSIGNED PK | |
| slug | VARCHAR(100) UNIQUE | |
| name | VARCHAR(255) | |
| description | TEXT NULL | |
| version | VARCHAR(20) | |
| author | VARCHAR(255) | |
| type | ENUM('system','community') | |
| signature | TEXT NULL | Plugin code signature (Phase 3+) |
| config_schema | JSON | |
| permissions | JSON | Declared capability requirements |
| is_active | BOOLEAN DEFAULT TRUE | |
| created_at | TIMESTAMP | |

#### site_plugins
| Column | Type | Notes |
|--------|------|-------|
| id | BIGINT UNSIGNED PK | |
| site_id | BIGINT FK→sites ON DELETE CASCADE | |
| plugin_id | BIGINT FK→plugins | |
| config | JSON | Per-site plugin config |
| is_enabled | BOOLEAN DEFAULT TRUE | |
| priority | INT DEFAULT 10 | Hook execution order |
| failures_recent | INT DEFAULT 0 | Circuit breaker counter |
| disabled_at | TIMESTAMP NULL | Auto-disable timestamp |
| UNIQUE(site_id, plugin_id) | | |

#### media
| Column | Type | Notes |
|--------|------|-------|
| id | BIGINT UNSIGNED PK | |
| uuid | CHAR(36) UNIQUE | |
| site_id | BIGINT FK→sites ON DELETE CASCADE | |
| user_id | BIGINT FK→users | Uploader |
| filename | VARCHAR(255) | Sanitized + UUID-prefixed |
| original_filename | VARCHAR(255) | |
| mime_type | VARCHAR(100) | |
| file_path | VARCHAR(500) | Relative to storage/uploads/ |
| file_size | BIGINT UNSIGNED | Bytes |
| content_hash | CHAR(64) | SHA-256 of file contents for dedup |
| alt_text | VARCHAR(500) NULL | |
| metadata | JSON | Dimensions, EXIF, etc. |
| deleted_at | TIMESTAMP NULL | Soft delete |
| created_at | TIMESTAMP | |
| INDEX(site_id) | | |
| INDEX(content_hash) | | For dedup lookup |

#### api_keys
| Column | Type | Notes |
|--------|------|-------|
| id | BIGINT UNSIGNED PK | |
| user_id | BIGINT FK→users ON DELETE CASCADE | |
| site_id | BIGINT FK→sites NULL | NULL = all sites |
| key_hash | CHAR(64) | SHA-256 hash |
| key_prefix | CHAR(8) | First 8 chars (after `wvr_`) shown in UI for identification |
| name | VARCHAR(255) | User-friendly label |
| scopes | JSON | ["sites:read","content:write"] |
| last_used_at | TIMESTAMP NULL | |
| expires_at | TIMESTAMP NULL | |
| revoked_at | TIMESTAMP NULL | Soft revocation for audit |
| created_at | TIMESTAMP | |
| INDEX(key_hash) | | |
| INDEX(user_id, revoked_at) | | |

#### build_jobs
| Column | Type | Notes |
|--------|------|-------|
| id | BIGINT UNSIGNED PK | |
| site_id | BIGINT FK→sites ON DELETE CASCADE | |
| triggered_by_user_id | BIGINT FK→users NULL | NULL for system/webhook triggers |
| status | ENUM('queued','building','success','failed','dead_letter') | |
| trigger_type | ENUM('manual','content_update','webhook','schedule') | |
| git_commit_hash | CHAR(40) NULL | |
| build_log | TEXT NULL | |
| retry_count | INT DEFAULT 0 | |
| started_at | TIMESTAMP NULL | |
| completed_at | TIMESTAMP NULL | |
| created_at | TIMESTAMP | |
| INDEX(site_id, status) | | |
| INDEX(triggered_by_user_id, created_at) | | |

#### ai_usage_log
| Column | Type | Notes |
|--------|------|-------|
| id | BIGINT UNSIGNED PK | |
| user_id | BIGINT FK→users | |
| site_id | BIGINT FK→sites NULL | |
| provider | VARCHAR(50) | openai, anthropic, ollama |
| model | VARCHAR(100) | |
| action | ENUM('generate','edit','summarize','seo','translate','alt_text') | |
| input_tokens | INT UNSIGNED | |
| output_tokens | INT UNSIGNED | |
| cost_amount | DECIMAL(10,6) DEFAULT 0 | Cost in `cost_currency` |
| cost_currency | CHAR(3) DEFAULT 'USD' | ISO 4217 |
| input_preview | TEXT NULL | Truncated input for audit/injection review |
| output_preview | TEXT NULL | Truncated output |
| created_at | TIMESTAMP | |
| INDEX(user_id, created_at) | | |
| INDEX(site_id, created_at) | | |

#### feature_flags
| Column | Type | Notes |
|--------|------|-------|
| id | BIGINT UNSIGNED PK | |
| flag_name | VARCHAR(100) UNIQUE | |
| is_enabled | BOOLEAN | Global override |
| user_scope | JSON NULL | {"include": [1,2], "exclude": [3]} |
| updated_at | TIMESTAMP | |

#### audit_log
| Column | Type | Notes |
|--------|------|-------|
| id | BIGINT UNSIGNED PK | |
| user_id | BIGINT FK→users NULL | NULL for system events / anonymized after GDPR deletion |
| site_id | BIGINT FK→sites NULL | NULL for non-site-scoped events |
| action | VARCHAR(100) | login, logout, login_failed, 2fa_setup, password_reset, content.create, plugin.install, admin.impersonate, etc. |
| entity_type | VARCHAR(50) NULL | user, site, content, plugin |
| entity_id | BIGINT NULL | |
| ip_address | VARCHAR(45) | IPv4/IPv6 |
| user_agent | VARCHAR(500) NULL | |
| details | JSON NULL | Additional context |
| created_at | TIMESTAMP | |
| INDEX(user_id, created_at) | | |
| INDEX(site_id, created_at) | | |
| INDEX(action, created_at) | | |

## API Changes

### Authentication Endpoints
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v1/auth/register` | Register with email/username/password (enumeration-resistant) |
| POST | `/api/v1/auth/verify-email` | Verify email with token |
| POST | `/api/v1/auth/resend-verification` | Resend verification email |
| POST | `/api/v1/auth/login` | Login with email + password |
| POST | `/api/v1/auth/verify-2fa` | Complete login with TOTP/WebAuthn |
| POST | `/api/v1/auth/verify-recovery` | Complete login with recovery code |
| POST | `/api/v1/auth/2fa/totp/setup` | Begin TOTP enrollment (returns QR) |
| POST | `/api/v1/auth/2fa/totp/confirm` | Confirm TOTP setup with two codes |
| POST | `/api/v1/auth/2fa/webauthn/register-challenge` | Begin WebAuthn registration (Phase 4) |
| POST | `/api/v1/auth/2fa/webauthn/register` | Complete WebAuthn registration |
| DELETE | `/api/v1/auth/2fa/methods/{method_id}` | Remove a 2FA method |
| POST | `/api/v1/auth/2fa/regenerate-recovery` | Generate new recovery codes |
| POST | `/api/v1/auth/forgot-password` | Request password reset email |
| POST | `/api/v1/auth/reset-password` | Reset password with token |
| POST | `/api/v1/auth/login/{provider}` | Initiate OAuth flow (google/apple/microsoft) |
| GET | `/api/v1/auth/callback/{provider}` | OAuth callback handler |
| POST | `/api/v1/auth/link/{provider}` | Link OAuth provider to account |
| POST | `/api/v1/auth/set-password` | Add local password to OAuth account |
| POST | `/api/v1/auth/choose-username` | OAuth user picks username on first login |
| POST | `/api/v1/auth/refresh` | Refresh JWT access token |
| POST | `/api/v1/auth/logout` | Revoke refresh token + add access token to deny-list |

### Resource Endpoints
| Method | Endpoint | Description |
|--------|----------|-------------|
| GET/POST | `/api/v1/sites` | List/create sites |
| GET/PUT/DELETE | `/api/v1/sites/{site_id}` | Read/update/soft-delete site |
| POST | `/api/v1/sites/{site_id}/build` | Trigger site build |
| GET | `/api/v1/sites/{site_id}/builds` | List build history |
| POST | `/api/v1/sites/{site_id}/domain/verify` | Initiate custom domain DNS verification |
| GET/POST | `/api/v1/sites/{site_id}/content` | List/create content |
| GET/PUT/DELETE | `/api/v1/sites/{site_id}/content/{content_id}` | Read/update/soft-delete content |
| POST | `/api/v1/sites/{site_id}/content/{content_id}/publish` | Publish content |
| POST | `/api/v1/sites/{site_id}/content/{content_id}/preview` | Synchronous preview render |
| GET | `/api/v1/sites/{site_id}/content/{content_id}/versions` | List versions |
| GET | `/api/v1/sites/{site_id}/content/{content_id}/versions/{version_id}` | Get version (body reconstructed from Git) |
| POST | `/api/v1/sites/{site_id}/content/{content_id}/versions/{version_id}/restore` | Restore version |
| GET/POST | `/api/v1/sites/{site_id}/members` | List/add site members |
| DELETE | `/api/v1/sites/{site_id}/members/{user_id}` | Remove site member |
| GET/POST | `/api/v1/templates` | List/create templates |
| GET/PUT/DELETE | `/api/v1/templates/{template_id}` | Read/update/delete template |
| GET | `/api/v1/plugins` | List available plugins |
| POST/DELETE | `/api/v1/sites/{site_id}/plugins/{plugin_id}` | Install/remove plugin for site |
| GET/POST | `/api/v1/sites/{site_id}/media` | List/upload media |
| DELETE | `/api/v1/sites/{site_id}/media/{media_id}` | Delete media |
| POST | `/api/v1/sites/{site_id}/ai/generate` | AI content generation |
| POST | `/api/v1/sites/{site_id}/ai/edit` | AI in-page editing |
| POST | `/api/v1/sites/{site_id}/ai/seo` | AI SEO metadata generation |
| GET | `/api/v1/sites/{site_id}/versions` | Git version history (built output) |
| POST | `/api/v1/sites/{site_id}/versions/{hash}/rollback` | Rollback site to commit |
| GET/POST | `/api/v1/api-keys` | List/create API keys |
| DELETE | `/api/v1/api-keys/{key_id}` | Revoke API key (soft) |
| POST | `/api/v1/users/me/export` | Initiate GDPR data export |
| POST | `/api/v1/users/me/delete` | Schedule account deletion (30-day grace) |
| POST | `/api/v1/users/me/delete/cancel` | Cancel pending deletion |
| POST | `/api/v1/webhooks/email/{provider}` | Inbound email bounce/complaint webhook |
| GET | `/api/v1/openapi.json` | OpenAPI 3.1 spec |
| GET | `/api/v1/docs` | Swagger UI |
| GET | `/health` | Liveness probe |
| GET | `/metrics` | Prometheus metrics |

All endpoints return JSON. List endpoints support `?page=`, `?per_page=` (max 100), `?sort=`, `?filter[field]=value`. Browser frontend authenticates via cookies + CSRF token; external API uses `Authorization: Bearer wvr_...` (API key) or `Authorization: Bearer <JWT>`.

## UI Changes

- **Registration page**: Email, username, password fields with strength indicator; link to OAuth sign-in (Phase 2); generic success message regardless of email existence
- **Login page**: Email + password form; "Sign in with Google/Apple/Microsoft" buttons (Phase 2); "Forgot password?" link
- **2FA management page (Phase 2)**: List of enrolled methods (TOTP, WebAuthn — Phase 4); add/remove method; view recovery codes count; regenerate recovery codes
- **2FA enrollment pages**: TOTP QR + two-code confirmation; WebAuthn device registration (Phase 4)
- **2FA verification page**: Method picker (TOTP, WebAuthn, recovery code); per-method input
- **OAuth username chooser**: One-time interstitial for OAuth users without a username
- **Password reset page**: Email input → token link → new password form
- **Account settings**: Change password, manage 2FA methods, link/unlink OAuth providers, manage API keys, request data export, schedule account deletion
- **Account lockout notice**: Displayed on login when account is temporarily locked with countdown
- **Custom domain page**: DNS verification instructions (TXT challenge); status indicator; TLS certificate state (Phase 4)
- **Trash bin / Recovery**: View soft-deleted sites and content, restore within 30 days
- **Plugin install page**: Trust warning for non-marketplace plugins (Phase 3); capability requirements displayed before install
- **AI proposed changes UI**: Diff view of AI suggestions; explicit Accept/Reject per change
