# Zerbook — Backend

REST API that records a user's physical gold holdings, values them against live
market prices, and shares that value inside family groups.

## Overview

People in Turkey hold a meaningful share of their savings as physical gold —
bracelets, quarter coins, full coins, bullion — kept at home or split across
family members. There is no ledger for it. Nobody knows what the household
actually owns, what it is worth today, or who is holding what.

Zerbook's backend is the system of record for that problem. It stores holdings
as exact integer quantities, polls a live price feed so every holding has a
current value, and models a **group** (a family, a couple, a business partner
set) where members can pool holdings, transfer them to each other, and see a
combined total. It deliberately does **not** buy, sell, or move money — it is a
ledger and a valuation engine, nothing more.

## Features

- Firebase-backed identity: e-mail/password, phone, Google and Apple sign-in all
  arrive as a single Firebase ID token that the API exchanges for its own JWT
- Access/refresh token pair with rotation, plus SMTP-based password reset codes
- Portfolio CRUD — holdings per gold category, with purchase price so
  profit/loss since acquisition can be computed
- Groups with 6-digit invite codes (1-hour TTL), member roles, per-member
  holdings, member-to-member transfers, and admin-side "record on behalf of"
- Live market prices polled from the Truncgil v4 feed, with price history
  written only on actual change and pruned by a retention job
- Push notifications through Firebase Cloud Messaging, delivered off the request
  path by an in-process queue plus a background worker
- Contact/feedback mail endpoint over SMTP
- Uniform error surface: every exception, validation failure and bodiless status
  code leaves the API as the same `{ message, code }` shape in user-facing text
- Unauthenticated `/health` endpoint for container and Kubernetes probes

## Architecture

**Controllers → Services → Repositories → EF Core `DataContext`.** Controllers do
HTTP only; business rules live in services; all database access goes through
repository interfaces registered in `Program.cs`.

- `Controllers/` — `AuthController`, `UserController`, `PortfolioController`,
  `GroupsController`, `PriceController`, `FcmController`, `MailController`
- `Services/` — token issuing and refresh, registration, Firebase/Google identity
  verification, password reset, notification dispatch, price fetching, SMTP send
- `Repositories/` — portfolio, group, price, mail, FCM token persistence
- `Models/` — `Portfolio`/`PortfolioItem`, `Group`/`GroupMember`/`GroupHolding`,
  `MarketPrice`/`MarketPriceHistory`, `AppUser`/`RefreshToken`/`PasswordResetCode`,
  `FcmToken`, `Mail`
- `Middleware/` + `Extensions/` — the friendly-error pipeline
  (`AddFriendlyApiErrors` / `UseFriendlyApiErrors`)

**Background workers** (all registered as `IHostedService`):

| Worker | Job |
|---|---|
| `PriceUpdateBackgroundService` | polls the Truncgil feed every minute |
| `PriceHistoryRetentionService` | prunes old price history rows |
| `NotificationBackgroundService` | drains the FCM notification queue |
| `InviteCodeCleanupService` | deletes expired group invite codes |

**Quantities are integers end to end.** Gram-denominated categories travel as
micrograms, piece-denominated categories as whole pieces, and prices as kuruş.
No floating point ever touches a balance. The client mirrors the same rule in
`core/utils/gold_format.dart`.

## Tech Stack

Backend:
- ASP.NET Core 10 (`net10.0`) Web API
- ASP.NET Core Identity for user store and password hashing
- JWT bearer auth (`Microsoft.AspNetCore.Authentication.JwtBearer`), 30-second
  clock skew
- Firebase Admin SDK 3.5 — identity verification and FCM delivery
- `Google.Apis.Auth` for Google ID token validation
- AutoMapper 15, DotNetEnv, Scalar (`Scalar.AspNetCore`) for API reference UI
- Truncgil v4 price feed over a named `HttpClient` (20s timeout)

Database:
- PostgreSQL 16
- Entity Framework Core 10 with the Npgsql provider
- EF migrations in `Migrations/`; optional auto-migrate on boot via
  `Database__AutoMigrate`

Infrastructure:
- Docker (`Dockerfile`) + `docker-compose.yml` for the local stack
- Kubernetes manifests in `k8s/` (namespace, deployment, service, nginx ingress)
- Public host: `zerbook.enesorhan.com`
- SMTP relay for password reset and contact mail
- ASP.NET Data Protection keys persisted to a volume so password-reset tokens
  survive container restarts

## Running Locally

**Prerequisites:** .NET 10 SDK, Docker Desktop.

1. **Clone and enter the project**

   ```bash
   git clone <repo-url> zerbook
   cd zerbook
   ```

2. **Create `.env` in the project root** (git-ignored):

   ```env
   PG_PASSWORD=<postgres-password>
   JWT__SigningKey=<at-least-32-random-characters>
   GOOGLE_CLIENT_ID_ANDROID=<...>
   GOOGLE_CLIENT_ID_IOS=<...>
   SMTP_HOST=<...>
   SMTP_PORT=587
   SMTP_USER=<...>
   SMTP_PASSWORD=<...>
   ```

3. **Drop the Firebase service account** at `secrets/firebase-adminsdk.json`.
   The app still boots without it — but `/api/Auth/firebase` returns 503 and
   push notifications are disabled, and a warning is printed at startup.

4. **Start the stack**

   ```bash
   docker compose up -d --build
   docker compose logs -f api
   ```

   The API listens on **http://localhost:5156**, Postgres on **5433**. The
   schema is applied at boot (`Database__AutoMigrate=true`), so no manual
   `dotnet ef database update` is needed.

5. **Or run on the host** against the composed database:

   ```bash
   dotnet restore
   dotnet ef database update
   dotnet run
   ```

6. **Verify**

   ```bash
   curl http://localhost:5156/health
   ```

   In Development the OpenAPI document is at `/openapi/v1.json` and the Scalar
   reference UI at `/scalar/v1`. `zerbook.http` and `docs/smoke-backend.http`
   hold ready-made requests.

7. **Tests**

   ```bash
   dotnet test tests/
   ```

## Production

Built as a container image and deployed to Kubernetes from `k8s/`:

- `namespace.yaml` — the `zerbook` namespace
- `deployment.yaml` — the API pod; connection string, JWT signing key and
  Firebase credentials come from Kubernetes secrets, not the image
- `service.yaml` — ClusterIP on port 8080
- `ingress.yaml` — nginx ingress for `zerbook.enesorhan.com`

TLS terminates at the ingress; the container speaks plain HTTP and only enables
`UseHttpsRedirection` when an HTTPS port is actually configured. `/health`
serves the liveness and readiness probes. Migrations run on pod start, so a
rollout applies schema changes with no shell step. The `secrets/` directory is
mounted read-only, and Data Protection keys live on a persistent volume so
password-reset tokens issued before a restart still validate afterwards.

## Technical Decisions

**Why PostgreSQL?**
Money and quantities need exact arithmetic and real transactional guarantees —
a group transfer debits one member and credits another and must not half-apply.
Postgres also handles the price-history table (an append-only time series) well
under index without a second store, and Npgsql + EF Core 10 is a first-class
combination on .NET.

**Why not Redis?**
There is no Redis here, and that is deliberate. The only hot cache candidate is
the market price table, which is a handful of rows refreshed once a minute —
Postgres answers that from shared buffers. Notification fan-out uses an
in-process `INotificationQueue` rather than an external broker. Adding Redis
would buy nothing at current scale and would add a component to operate. The
moment the deployment goes multi-replica with cross-instance state, this is the
first decision to revisit.

**Why background hosted services instead of a job framework?**
The four recurring jobs are simple, idempotent, and tolerate a missed tick.
`IHostedService` ships in the box, runs in the same process, and needs no job
store. Hangfire or Quartz would mean another schema and another dashboard to
secure, for jobs that are a loop and a timer.

**Why Firebase as the identity source?**
The mobile app needs e-mail/password, phone OTP, Google and Apple sign-in.
Implementing phone verification and Apple's token dance server-side is
significant work with real security surface. Firebase does all four and hands
back one ID token; the API verifies that token and issues its own JWT, so the
rest of the system knows only about Zerbook tokens. Firebase Admin was already a
dependency for FCM, so this added no new vendor.

**Why integers everywhere instead of `decimal`?**
Gold quantities get divided (a transfer of a third of a bracelet's worth) and
multiplied by prices constantly. Fixing the unit at microgram/piece/kuruş means
rounding happens once, at the display boundary, in a single function on each
side — rather than accumulating silently across a chain of operations.

**Why a custom error middleware?**
The app is consumed by one Flutter client that shows server messages directly to
users. Letting ASP.NET emit its default `ProblemDetails` for some failures,
model-binding errors for others, and empty 401/403 bodies for the rest would
push that normalization into the client. One middleware at the outermost layer
guarantees a single shape and human-readable text.

## Screenshots

API reference (Scalar) in Development:

```
http://localhost:5156/scalar/v1
```

| | |
|---|---|
| Scalar API reference | `docs/screenshots/scalar-reference.png` |
| Health endpoint | `docs/screenshots/health.png` |

Client-side screens are in the [Zerbook mobile README](./zerbook-frontend.md).

## Roadmap

- [ ] Redis backplane + distributed cache, as a prerequisite to running more
      than one replica
- [ ] Second price source with cross-checking, so a bad upstream tick cannot
      move everyone's portfolio value
- [ ] Per-user price alerts ("notify me when gram gold passes X")
- [ ] Group activity log surfaced through the API (who added, transferred,
      removed what and when)
- [ ] CSV / PDF portfolio export
- [ ] Rate limiting on auth endpoints
- [ ] Structured logging and metrics export
- [ ] Broader integration test coverage over group transfer edge cases

## License

See the `LICENSE` file in the project repository.
