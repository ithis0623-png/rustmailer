# Threat Model: RustMailer (Self-Hosted Email Middleware)

RustMailer is a self-hosted email middleware service written in Rust that provides IMAP synchronization, SMTP sending, and Gmail/Graph API handling over REST (Poem OpenAPI) and gRPC interfaces. It manages account credentials, token authorization, webhooks, and email tracking.

## 1. What This Project Does and Where Untrusted Input Enters
- **Network Boundaries:**
  - REST API endpoints (`/api/v1/*`) and public authentication endpoints (`/api/login`, `/metrics`, `/api/status`).
  - Email tracking link redirection and pixel endpoint (`/email-track/:id`).
  - gRPC service endpoints (`src/modules/grpc/*`) listening on port 16630.
  - Inbound IMAP server responses and MIME email parsing via `mail-parser`.
  - Webhook delivery and Vector Remap Language (VRL) execution scripts.
- **Data & Configuration:**
  - Client-supplied access tokens, IP whitelist checks, and rate-limiting rules.
  - Encrypted account credentials stored in native_db / redb.

## 2. Threat Actors & Boundaries
- **Remote Anonymous Attacker:** Senders of arbitrary network traffic to HTTP/gRPC ports. Target: unauthenticated API execution, root login brute-force, open redirect via `/email-track/:id`, unauthenticated scraping via `/metrics`.
- **IP Spoofing Attacker:** Network clients utilizing `X-Forwarded-For` headers to bypass `access_token` IP whitelist access controls.
- **Compromised Account / Untrusted Email Sender:** Inbound malicious email messages containing specially crafted MIME structures aimed at crashing parsers or causing denial of service.

## 3. Critical Components vs Out of Scope
### High Priority Focus
- **Authentication & Authorization (`src/modules/common/auth.rs`, `src/modules/token/`):**
  - Validation of `ClientContext` and ensuring endpoints cannot be invoked when `rustmailer_enable_access_token` is enabled.
  - Resistance of `POST /api/login` against brute force and default credentials (`DEFAULT_ROOT_PASSWORD`).
  - Secure IP parsing ensuring `X-Forwarded-For` cannot be spoofed unless behind an explicitly configured trusted proxy.
- **Cryptography & Key Storage (`src/modules/utils/encrypt.rs`):**
  - PBKDF2 iteration count, salt generation, and AES-256-GCM authenticated encryption.
  - Storage of root tokens on the filesystem (`src/modules/token/root.rs`).
- **SMTP & Redirection Safety (`src/modules/smtp/`):**
  - Input validation on tracking redirects to prevent open redirection to malicious domains or arbitrary protocols (`javascript:`).

### Out of Scope (Do Not Report)
- Third-party Rust dependencies in `~/.cargo/registry`.
- Development UI test fixtures.
- Memory consumption during synchronization of multi-gigabyte IMAP mailboxes when configured by the administrator.

## 4. Severity Calibration
- **Critical:** Remote Code Execution (RCE), unauthenticated admin access / root token retrieval, credential decryption key bypass.
- **High:** Authentication bypass in REST/gRPC APIs, IP whitelist bypass via spoofed headers, SSRF in webhook dispatcher.
- **Medium:** Open redirect in email tracking endpoint, unauthenticated sensitive metrics leakage (`/metrics`), panic-induced denial of service (`.expect()` on missing configurations).
- **Low:** Missing rate limiting on public endpoints, minor CORS configuration laxity.

## 5. How to Exercise and Test
- Run tests: `cargo test`
- Run specific module tests: `cargo test modules::token`
- Check code with clippy: `cargo clippy --all-targets`
