# Candidate Search — Backend

ASP.NET Core 8 Web API + PostgreSQL (EF Core / Npgsql) with JWT authentication.
Provides registration, login (with refresh tokens) and a filterable, sortable, paginated candidate search.


## Prerequisites

| Tool | Version |
| --- | --- |
| .NET SDK | 8.0 or newer (project targets `net8.0`; the .NET 8 runtime must be installed) |
| PostgreSQL | Any recent version, running locally (default port 5432) |

## Setup

All commands below are run from the repo root unless stated otherwise. The project itself is in `Candiate_search_assesment/`.

### 1. PostgreSQL connection string

The connection string is read from `Candiate_search_assesment/appsettings.json`:

```json
"ConnectionStrings": {
  "CandidateSearchDb": "Host=localhost;Port=5432;Database=CandidateSearchDb;Username=postgres;Password=postgres"
}
```

Edit the host / port / username / password to match your local PostgreSQL. You do **not** need to create the database first — the migration step below creates `CandidateSearchDb` if it doesn't exist.

To avoid keeping credentials or secrets in a file, any setting can be overridden with environment variables (double underscore = nesting), e.g.:

```bash
# bash
export ConnectionStrings__CandidateSearchDb="Host=localhost;Port=5432;Database=CandidateSearchDb;Username=me;Password=secret"
export Jwt__Key="a-long-random-string-of-at-least-32-characters"
```

```powershell
# PowerShell
$env:ConnectionStrings__CandidateSearchDb = "Host=localhost;Port=5432;Database=CandidateSearchDb;Username=me;Password=secret"
$env:Jwt__Key = "a-long-random-string-of-at-least-32-characters"
```

### 2. Other configuration (`appsettings.json`)

| Key | Purpose | Default |
| --- | --- | --- |
| `Jwt:Key` | HMAC signing key (min. 32 chars). **Replace for anything beyond local dev.** | dev placeholder |
| `Jwt:Issuer` / `Jwt:Audience` | Validated on every request | `CandidateSearchAssessmentApi` / `CandidateSearchAssessmentClient` |
| `Jwt:ExpiresInMinutes` | Access-token lifetime | `60` |
| `Jwt:RefreshTokenExpiresInDays` | Refresh-token lifetime | `7` |
| `Cors:AllowedOrigins` | Frontend origins allowed to call the API | `http://localhost:5173`, `5174`, `5175` |

If your frontend runs on a different origin, add it to `Cors:AllowedOrigins` (or set `Cors__AllowedOrigins__0=...`).

### 3. Run the migrations

The EF Core CLI is pinned as a local tool (`dotnet-tools.json`, version 8.0.10). From the repo root:

```bash
dotnet tool restore
cd Candiate_search_assesment
dotnet ef database update
```

This creates the database (if needed) and applies both migrations:

1. `InitialCreate` — the `candidates` table, with a unique index on `email`.
2. `AddRefreshTokenToCandidates` — adds `refresh_token_hash` and `refresh_token_expires_at`.

`candidates` columns: `id`, `name`, `email` (unique), `password_hash`, `age`, `gender`, `location`, `education`, `created_at`, `last_login_at` (nullable), `refresh_token_hash`, `refresh_token_expires_at`.

### 4. Seed data

**There is no seed script in this repo.** Candidates are created through the API. Registration only requires `email` + `password`, but the search filters need profile data, so also pass the optional profile fields. Example (bash):

```bash
API=http://localhost:5074/api/auth/register
reg() { curl -s -X POST "$API" -H "Content-Type: application/json" -d "$1" -w " -> %{http_code}\n"; }

reg '{"email":"anjali@example.com","password":"Password@123","name":"Anjali","age":28,"gender":"Female","location":"Kochi","education":"M.Tech"}'
reg '{"email":"rahul@example.com","password":"Password@123","name":"Rahul","age":30,"gender":"Male","location":"Kochi","education":"MBA"}'
reg '{"email":"meera@example.com","password":"Password@123","name":"Meera","age":26,"gender":"Female","location":"Bangalore","education":"B.Tech"}'
reg '{"email":"arjun@example.com","password":"Password@123","name":"Arjun","age":32,"gender":"Male","location":"Mumbai","education":"MCA"}'
reg '{"email":"diya@example.com","password":"Password@123","name":"Diya","age":29,"gender":"Female","location":"Chennai","education":"PhD"}'
```

The backend must be running (next section) when you do this. Use education values exactly as the frontend checkboxes send them: `B.Tech`, `M.Tech`, `MBA`, `MCA`, `PhD` (see the education filter assumption below).

Rows inserted directly with SQL need a real bcrypt hash in `password_hash` to be able to log in. A malformed hash simply makes that login return `401`.

## Running the backend

```bash
cd Candiate_search_assesment
dotnet run
```

The API listens on **http://localhost:5074**. In the `Development` environment Swagger UI is at **http://localhost:5074/swagger** (use the *Authorize* button and paste the token returned by `/api/auth/login`).

## Running the frontend

The frontend is a separate folder/repo (`Frontend/candidate-search`, React 19 + TypeScript + Vite + Tailwind).

```bash
cd ../Frontend/candidate-search      # adjust to wherever you cloned it
npm install
```

Create a `.env` file in that folder (see `.env.example` if present):

```
VITE_API_BASE_URL=http://localhost:5074/api
```

Then:

```bash
npm run dev
```

Open **http://localhost:5173** and sign in with one of the candidates you registered. If port 5173 is taken, Vite falls back to 5174/5175 — those are already in the backend's CORS allow-list.

Start the backend **before** signing in.

## API reference

| Method | Route | Auth | Description |
| --- | --- | --- | --- |
| POST | `/api/auth/register` | none | `{ email, password, name?, age?, gender?, location?, education? }` → `201`; `409` if email exists; `400` with a structured validation body |
| POST | `/api/auth/login` | none | `{ email, password }` → `{ token, expiresIn, refreshToken }`; `401` on any bad credentials |
| POST | `/api/auth/refresh` | none | `{ refreshToken }` → new `{ token, expiresIn, refreshToken }` (rotates the refresh token); `401` if invalid/expired |
| POST | `/api/auth/logout` | Bearer | Revokes the caller's refresh token → `204` |
| GET | `/api/candidates/search` | Bearer | Search (below) |

Every endpoint except register / login / refresh requires `Authorization: Bearer <token>`; a missing, malformed or expired token returns `401`.

### `GET /api/candidates/search`

All parameters are optional and applied only when supplied.

| Param | Example | Notes |
| --- | --- | --- |
| `minAge`, `maxAge` | `minAge=25&maxAge=32` | Inclusive |
| `gender` | `gender=Female` | Case-insensitive exact match |
| `location` | `location=Kochi` | Case-insensitive "contains" |
| `education` | `education=B.Tech,M.Tech,MBA` | Comma-separated; one `IN` query |
| `createdFrom`, `createdTo` | `createdFrom=2026-09-01&createdTo=2026-09-22` | Date range, inclusive both ends |
| `lastLoginFrom`, `lastLoginTo` | `lastLoginFrom=2026-09-01` | Same; never-logged-in candidates never match |
| `sortBy`, `sortOrder` | `sortBy=lastLoginAt&sortOrder=desc` | `sortBy`: `createdAt` (default) \| `lastLoginAt` \| `age`; `sortOrder`: `asc` \| `desc` (default) |
| `page`, `pageSize` | `page=2&pageSize=20` | Defaults `1` / `20`; `pageSize` capped at 100 |

Response:

```json
{
  "page": 2,
  "pageSize": 20,
  "totalCount": 135,
  "items": [
    { "id": 101, "name": "Anjali", "age": 28, "education": "M.Tech", "location": "Kochi",
      "createdAt": "2026-09-10T08:30:00Z", "lastLoginAt": "2026-09-21T10:15:00Z" }
  ]
}
```

The logged-in candidate (taken from the JWT `sub`/`NameIdentifier` claim) is excluded in the SQL `WHERE` clause. All filtering, counting, sorting and paging (`COUNT`, `ORDER BY`, `LIMIT/OFFSET`) run in PostgreSQL — nothing is loaded into memory first.

## Assumptions

Places where the brief was ambiguous, and what I chose:

**Data model and auth**
- **A candidate is also the user.** One `candidates` table backs both login and search; there is no separate users table.
- **Registration takes optional profile fields** (`name`, `age`, `gender`, `location`, `education`) beyond the `{email, password}` in the brief, because search needs them and there's no profile-edit endpoint. Omitted fields are stored as empty string / `0`.
- **Emails are trimmed and lower-cased** before storing and looking up, so `User@x.com` and `user@x.com` are the same account.
- **Password strength:** at least 8 characters with an uppercase letter, a lowercase letter, a digit and a special character. Hashed with BCrypt (work factor 11).
- **Registration success** returns `201` with the new candidate's `id`, `email`, `name` (the brief didn't specify a body). A duplicate email returns `409`; the unique index also backstops concurrent duplicate registrations.
- **Login** returns the same generic `401` for an unknown email, a wrong password, or a malformed stored hash. `last_login_at` is set (UTC) on every successful login.
- **JWT:** HS256, 60-minute lifetime, 30-second clock skew. `expiresIn` is in seconds and is derived from the token's actual `exp`. The candidate id is carried in both `sub` and `NameIdentifier`.
- **Everything is protected by default.** A global fallback authorization policy requires authentication; only register / login / refresh opt out with `[AllowAnonymous]`.
- **Refresh tokens (an extra beyond the brief):** opaque random values, stored only as a SHA-256 hash on the candidate's row, so there is **one active refresh token per candidate** (logging in on a second device invalidates the first). Tokens rotate on every refresh (the old one stops working) and are revoked on logout.

**Search**
- **Education** is an exact, case-sensitive match against the stored string, translated by EF Core/Npgsql to `education = ANY(@values)` (the `IN` equivalent). Values are split on commas, trimmed and de-duplicated. So `B.Tech` will **not** match `BTech` or `B.Tech Computer Science` — the stored data has to use the same canonical strings the UI sends.
- **Location** is a case-insensitive *contains* match (`ILIKE '%…%'`), on the assumption that "search" should be forgiving. **Gender** is a case-insensitive exact match.
- **Dates** are treated as calendar dates in UTC. `From` is the start of that day and `To` includes that entire day (compared as `< next day`), so both ends are inclusive.
- **Null `last_login_at`:** candidates who have never logged in are excluded whenever `lastLoginFrom` or `lastLoginTo` is supplied (`last_login_at IS NOT NULL AND …`).
- **Bad query input doesn't error:** an unknown `sortBy` falls back to `createdAt`, a non-`asc` `sortOrder` falls back to `desc`, `page < 1` becomes `1`, and an invalid `pageSize` becomes `20` (max `100`). `minAge > maxAge` simply returns no results.
