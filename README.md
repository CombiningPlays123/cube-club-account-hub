# Cube Club Account Hub (Repository #1)

This is the separate account website in the Cube Club two-repository system. Members create and manage their account here, import exported CSTimer solve files, see verified stats and Cube Points, and generate/revoke authentication tokens for the distinct [Cube Club main site](../cube-club/).

The Account Hub is a static GitHub Pages frontend. It does not host the database, validate solves itself, or keep account data in browser storage. Those actions require the secure API described below. The main Cube Club frontend is in its own `cube-club` folder/repository. Copy the contents of this folder into the Account Hub repository root; deploy the two folders as two different sites.

## Local development

Edit `config.js` and serve this folder over HTTP:

```sh
cd account-hub
npx serve .
```

No frontend package install or build is needed. Keep `DEMO_MODE: false` unless serving locally and intentionally previewing sample flows. Local demo accepts arbitrary account form input, keeps all state in memory, and returns an obvious non-usable demo credential. Demo mode is limited to localhost and must never be enabled in production.

## Configure URLs

Set public URLs in `config.js` before deploying:

```js
window.ACCOUNT_HUB_CONFIG = {
  CUBE_CLUB_API_URL: "https://api.example.org/cube-club",
  CUBE_CLUB_SITE_URL: "https://club.example.org",
  DEMO_MODE: false
};
```

`CUBE_CLUB_API_URL` is the shared backend base URL. `CUBE_CLUB_SITE_URL` links to the separately deployed main Cube Club site. Both are public values. Never place passwords, private API keys, session signing secrets, database credentials, service-role keys, or raw reward codes in this repository. GitHub Pages does not read `.env`; `.env.example` is a reference only.

## What the website does

- Registers and signs members in through the backend.
- Shows backend-owned Cube Points, rank, solve stats, goals, and transactions.
- Uploads CSTimer exported solve files for server-side validation. It does not claim direct CSTimer account access.
- Creates tokens with selectable 24-hour, 7-day, or 30-day lifetimes and displays the new raw credential once.
- Lists token created/expiry/last-used status and lets members revoke credentials.
- Provides password and public-stat privacy settings through authenticated API endpoints.

The generated token is a long random authentication credential only. It does not encode points, username, email, solves, rewards, or expiry. The backend stores a secure hash and metadata, uses server time to check expiration, associates the credential with the account, and creates a separate secure session when the main site exchanges the token. The account token is not the permanent session.

## API contract

All endpoints use `CUBE_CLUB_API_URL` and require HTTPS. API requests include credentials so the backend can use a `Secure; HttpOnly; SameSite` session cookie. A short-lived opaque session token is supported as a fallback; prefer HttpOnly cookies with CSRF defenses. Enable strict CORS for both deployed site origins. Use rate limits and backend authorization for every endpoint.

| Method | Endpoint | Purpose |
|---|---|---|
| `POST` | `/auth/register` | Accept email, display name, password; create account through backend validation. |
| `POST` | `/auth/login` | Authenticate credentials and establish secure session. |
| `POST` | `/auth/logout` | Revoke current session. |
| `GET` | `/account` | Restore session and return safe account summary / visibility preferences. |
| `GET` | `/account/stats` | Return authoritative points, rank, stats, goals and achievement counts. |
| `GET` | `/account/solves` | Return this member's validated solve records. |
| `POST` | `/account/imports/cstimer` | Multipart form field `file`; parse, validate, deduplicate, flag, and accept imported records. |
| `GET` | `/account/goals` | Return backend-calculated goal state and progress. |
| `GET` | `/account/transactions` | Return immutable points ledger history. |
| `GET` | `/account/tokens` | Return token metadata only (never token hash or raw secret). |
| `POST` | `/account/tokens` | Accept `{ "expires_in_days": 1, 7, or 30 }`; create hash/metadata and return the raw token once. |
| `DELETE` | `/account/tokens/{id}` | Revoke an account-owned token. |
| `POST` | `/account/password` | Change password after backend verification of current password. |
| `PATCH` | `/account/profile-visibility` | Save the account's public-stat visibility preferences. |

Expected one-time token response shape:

```json
{
  "token": {
    "id": "opaque-token-id",
    "secret": "CC-…",
    "created_at": "2026-10-04T19:00:00Z",
    "expires_at": "2026-10-11T19:00:00Z"
  }
}
```

Only `secret` is shown once. Persist its cryptographic hash, never the raw secret. Keep token metadata (`id`, account ID, hash, created/expiry/revoked/last-used times). A token exchange at the main site's `POST /auth/token` validates server-side, resolves the account, then creates a separate session. Return stable invalid/expired/revoked error codes without leaking account existence.

## CSTimer import and anti-cheat

Members export their solves from CSTimer and upload a supported text, CSV, or JSON export. The backend must parse actual supported formats, validate fields and times, deduplicate by account/source/solve identity, and flag repeated payloads, impossible solve times, and abnormal activity for review. Do not award points directly from a file upload or browser-reported goal. After validation, the backend can recalculate best single/average, total solves, goals, and eligible points transactions. Suspicious solves should be flagged for administrator review; one anomalous time is not an automatic ban.

## Database and security

Use the shared schema proposal at [`../cube-club/schema.sql`](../cube-club/schema.sql) after adapting it to the existing account model. It includes token hashes and metadata, sessions, solves, goals, user goals, points transactions, redemptions, achievements, audit logs, and club settings. Maintain points from trusted transactions; browser-supplied points are ignored. Database writes run under a least-privilege backend role. Do not ship Supabase service-role credentials or database keys to the static site.

Also enforce server-side password hashing, account ownership, token revocation/expiry, CSRF protections where cookies are used, and admin role checks. Raw token values should not appear in application logs or analytics. A token's expiration is checked against server time, never the member's computer clock.

## GitHub Pages

Copy the contents of this folder into the **Cube Club Account Hub** repository root. In **Settings → Pages**, select the default branch and `/ (root)`. Set the URL values in `config.js`, leave `DEMO_MODE: false`, and allow this site's HTTPS origin in API CORS. Repeat independently for the main Cube Club repository. The parent deliverable README explains both deployments and link configuration.

Before real member use, the backend must be deployed and integrated. This frontend alone cannot persist registrations, account data, stats, tokens, or solve imports. When API configuration is blank, real operations report that configuration is missing rather than pretending to succeed.
