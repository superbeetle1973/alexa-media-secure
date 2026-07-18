# Authentication — how Alexa Media Player Secure logs in

This document explains, end to end, how this integration authenticates with
Amazon, what it stores, what it deliberately does **not** store, and how to
reason about the security properties and limits. It is the piece the original
`alexa_media_player` never spelled out.

If you just want to set it up, read [Quick start](#quick-start). If you want to
understand or audit the model, read the rest.

---

## TL;DR

- You log in **once**, interactively, on **Amazon's real login page** in your own
  browser.
- The integration receives a durable **device refresh token** and stores only
  that (plus `mac_dms` and a device serial).
- **Your Amazon password and TOTP/2FA seed are never stored** — they are typed
  only into amazon.com and never touch this integration's storage.
- Everything at runtime (access tokens, session cookies, push) is re-derived
  from the refresh token on each start. No cookies are persisted.
- There is **no capture proxy** and **no HTML scraping**.

---

## Quick start

1. **Settings → Devices & Services → Add Integration → "Alexa Media Player
   Secure".**
2. **Step 1 — account + region.** Enter your Amazon account email (used only as
   a label/identifier, not for the login itself) and your Amazon region domain
   (e.g. `amazon.com`, `amazon.co.uk`, `amazon.de`).
3. **Step 2 — log in on Amazon.** The form shows an Amazon login URL. Open it in
   your browser, sign in normally — **password, 2FA, captcha, any challenge, all
   handled by Amazon's own page.** When you finish, your browser lands on a blank
   or "not found" page at `https://www.amazon.<region>/ap/maplanding`. **That is
   expected.**
4. **Paste the URL back.** Copy the **full address-bar URL** of that
   `maplanding` page and paste it into the config-flow field. Do this within
   ~5 minutes (the authorization code expires quickly).
5. Done. Your Echo devices, media players, switches, and sensors appear.

If the paste step errors with "registration failed," the code expired — start
the add again and paste faster.

---

## Why this design

The upstream integration historically stored, in plaintext in
`.storage/core.config_entries`:

- your Amazon **password**,
- your **TOTP shared secret** — the 2FA *seed*, not a one-time code, which is
  enough to generate your second factor forever, and
- the durable OAuth tokens.

Storing the TOTP seed next to the password on the same host effectively **clones
your authenticator into Home Assistant**: anyone who can read that file (a leaked
backup, a shared config, an add-on with filesystem access) recovers both factors
from one read. It was done to enable unattended re-login, but it collapses the
factor-independence that 2FA exists to provide — specifically against compromise
of that HA data.

This integration removes that trade. It keeps the one mechanism Amazon's app
actually uses — a **PKCE authorization-code exchange that registers a device and
returns a durable refresh token** — and changes only *how you bootstrap it* and
*what is kept on disk*.

---

## The enrollment flow in detail

```
┌────────────┐   1. open login URL    ┌─────────────────────┐
│  Your      │ ─────────────────────► │  amazon.com          │
│  browser   │   password + 2FA here  │  (real login page)   │
└────────────┘ ◄───────────────────── └─────────────────────┘
      │           2. redirect to
      │              /ap/maplanding?...&openid.oa2.authorization_code=...
      │
      │ 3. you copy the full URL and paste it into HA
      ▼
┌──────────────────────┐   4. POST /auth/register    ┌─────────────────┐
│  This integration    │ ──────────────────────────► │  api.amazon.com  │
│  (extracts the code, │   PKCE code_verifier +      │                  │
│   holds PKCE verifier)│   authorization_code        │                  │
└──────────────────────┘ ◄────────────────────────── └─────────────────┘
      │                     refresh_token + mac_dms
      ▼
   stores refresh_token + mac_dms + serial (0600). Nothing else.
```

Step by step:

1. The integration generates a fresh **PKCE `code_verifier`/`code_challenge`**
   and a device serial, and builds an Amazon `/ap/register` authorize URL that
   embeds the challenge. The `code_verifier` never leaves the integration's
   memory.
2. You open that URL and authenticate on Amazon's page. Amazon redirects you to
   `/ap/maplanding` with an `openid.oa2.authorization_code` in the query string.
3. You paste the full `maplanding` URL back. The integration extracts the
   authorization code (and verifies the URL is a `maplanding` redirect for the
   region you chose).
4. The integration calls `POST /auth/register` with the authorization code and
   the PKCE `code_verifier`. Amazon returns a durable **`refresh_token`** and
   **`mac_dms`** (device authentication material).
5. The integration binds the config entry's `unique_id` to your Amazon account
   id (present in the signed `maplanding` response) and stores only the minimized
   credentials.

**Why paste-URL instead of a proxy?** Amazon's device-registration login only
redirects to its own `maplanding` page — it will not redirect to a local
callback URL. The alternatives are a man-in-the-middle proxy that *sees* your
password in transit, or the paste-URL flow where your credentials only ever
touch amazon.com. This integration uses paste-URL. (The pattern is the same one
`mkb79/Audible` uses in production.)

---

## What is stored — and what is not

Stored, in the config entry (a candidate for a dedicated `Store(private=True)`,
0600):

| Field | What it is |
|---|---|
| `refresh_token` | Durable bearer credential — mints access tokens & cookies |
| `mac_dms` | Device auth material (request signing / push) |
| `serial` | The registered device serial |
| `customer_id` | Your Amazon account id (for `unique_id` binding) |

**Never stored:**

- ❌ Amazon password
- ❌ TOTP / 2FA seed
- ❌ One-time codes
- ❌ Persisted session cookies (no cookie file, no pickle)

Be honest about what *is* stored: `refresh_token` and `mac_dms` are **replayable
credentials** — anyone who reads them can act as your registered device until you
deregister or change your password. This design *minimizes* durable credential
material and removes the password/seed; it does not claim to store nothing
sensitive. Treat your HA backups accordingly.

---

## Runtime

On every start, from the stored refresh token only:

1. `refresh_access_token()` — mint a short-lived access token.
2. `exchange_token_for_cookies()` — re-mint session cookies (`at-main`,
   `session-id`, …). **Cookies are never written to disk.**
3. Normal API polling + **HTTP/2 push** (real-time updates) run off that session
   and `mac_dms`.

Because the session is rebuilt from the refresh token, a restart never depends on
a saved cookie file, and there is no cookie-serialization code to break across
`aiohttp`/Python upgrades.

---

## Reauthentication

Reauth is required only on a **genuine terminal auth failure** — a revoked or
expired refresh token, or an Amazon password change. Transient problems (network
errors, Amazon 5xx, rate limiting) are retried with backoff and do **not** force
reauth.

When reauth is genuinely needed, HA surfaces a "reconfigure/repair" prompt that
runs the **same interactive paste-URL flow** as first-time enrollment. There is
no unattended recovery from a dead refresh token — by design, re-establishing
2FA requires a person. Headless/remote installs should be aware: if the refresh
token dies, the integration stays unavailable until an admin completes the
browser flow.

---

## Removing the integration / deregistering the device

Each enrollment registers a device on your Amazon account. You can see and remove
these under Amazon → **Manage Your Content and Devices**, or in the Alexa app's
device settings. Re-enrolling repeatedly without cleaning up will accumulate
"phantom" devices, which can raise Amazon's risk scoring. A best-effort
deregistration on entry removal is on the roadmap (`/auth/deregister`).

---

## Security properties and honest limits

**What this protects against:**

- Disclosure of your **password** or **2FA seed** via a leaked/ shared HA
  backup, config, or a lower-privilege add-on — those values are never on disk.
- Brittle failures and one class of RCE from the pickle cookie loader (removed).

**What it does *not* protect against:**

- A **root-level compromise of the HA host** — anyone who can read the filesystem
  or process memory can read the refresh token. On-host encryption with an
  on-host key is data hygiene, not protection from host compromise.
- The refresh token and `mac_dms` remaining **replayable** if exfiltrated.

**Operational reality:** this uses Amazon's first-party device-registration
endpoints, which are undocumented for third-party use. It may violate Amazon's
terms, can break on any server-side change, and may trigger account challenges.
That risk is inherent to any integration of this kind; this design reduces the
*secret-handling* blast radius, it does not make the underlying flow supported.

---

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| "Registration failed" on paste | The authorization code expired (~5 min). Restart the add and paste faster. |
| "Not a valid maplanding redirect for this region" | You pasted a URL for a different Amazon region than you selected, or an incomplete URL. Copy the **full** address-bar URL. |
| Entry set up but devices unavailable | Check that the refresh token is still valid; a password change invalidates it → reauth. |
| Repeated reauth prompts | Usually transient Amazon errors; they should self-resolve. Persistent prompts mean the refresh token is genuinely dead. |
| Mobile browser hides the URL | Use a desktop browser for enrollment, or a browser that exposes the full address bar. |

---

## For developers

The auth core lives in the `alexapy_secure.secureauth` module and is independent
of Home Assistant:

- `EnrollmentFlow(domain)` — PKCE + `oauth_url` + `parse_redirect_url()` +
  `async_register()`. `to_state()`/`from_state()` persist an in-progress flow
  across processes (this is what the two-step config flow relies on).
- `TokenManager(credentials)` — `async_refresh_access_token()`,
  `async_exchange_cookies()`, `async_deregister()`, with failures classified as
  `AuthTransientError` (retry) vs `AuthTerminalError` (reauth).
- `DeviceCredentials` — the minimized persisted credential set.

A standalone end-to-end harness is available for verifying the flow against a
real account without touching Home Assistant:

```
python -m alexapy_secure.prove enroll-start          # prints login URL
python -m alexapy_secure.prove enroll-finish "<URL>" # completes registration
python -m alexapy_secure.prove refresh               # mint access token
python -m alexapy_secure.prove cookies               # re-mint session cookies
python -m alexapy_secure.prove probe                 # hit Alexa API endpoints
python -m alexapy_secure.prove deregister            # clean up the test device
```

Design rationale, threat model, and the full validation record:
https://github.com/superbeetle1973/alexa-auth-redesign
