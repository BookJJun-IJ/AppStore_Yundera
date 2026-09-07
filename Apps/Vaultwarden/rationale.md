# Vaultwarden — Rationale

## What deviation / exception is being requested

Three, all documented below:

1. **`SIGNUPS_ALLOWED: "true"`** — until the owner turns signups off, anyone who can
   reach the server's API can create an account.
2. **`DISABLE_ADMIN_TOKEN: "true"`** — Vaultwarden's own `/admin` gate is turned off and
   the app ships with no `ADMIN_TOKEN` at all. AppShield's SSO is the gate instead.
3. **`ALLOWED_PATHS` on the AppShield sidecar** — the Bitwarden client protocol surface
   bypasses the SSO gate. CONTRIBUTING says to prefer `OAUTH_RESOURCE` over
   `ALLOWED_PATHS`; that is not applicable here (see below).

---

## 1. Open signup

### Why it is necessary

- **It is the only bootstrap path.** Vaultwarden has no seeded first user and no
  "create the owner account" flow outside of signup. With `SIGNUPS_ALLOWED: "false"` from
  the start, a freshly installed instance has zero accounts and no way to make one from
  the normal UI — the owner would have to go to `/admin` and invite themselves by email,
  which requires working outbound SMTP.
- **SMTP is not guaranteed to work.** The compose points `SMTP_HOST` at the platform
  `smtp` relay, but invitation mail depends on that relay being reachable and on the
  destination accepting it. Making first-run account creation depend on an email round
  trip would turn a zero-config install into a support case whenever mail is unavailable.
- **It is the upstream default.** `vaultwarden/server` ships `SIGNUPS_ALLOWED=true`;
  this app does not loosen anything relative to upstream.

### Security mitigations in place

- **The exposure is now narrower than it was.** The web vault (`/`) sits behind the SSO
  gate, so a stranger with a browser cannot reach a registration form at all. Registration
  remains reachable over the client API (`identity/`), which must stay open for the
  Bitwarden apps — so this is a reduction, not a fix.
- **Open signup does not expose any existing vault.** Every vault is encrypted client-side
  under its own master password, which the server never holds. A new account created by a
  stranger is an empty vault; it grants no read access to the owner's data, and the admin
  panel cannot decrypt a user vault either — stated in `tips.before_install`.
- **The owner is told to close it, in the install dialog, before they install.**
  `tips.before_install` step 3 is "Secure your instance: Disable new signups in admin
  panel once set up", alongside a direct link to the admin panel.

### Alternatives considered and rejected

1. **`SIGNUPS_ALLOWED: "false"` out of the box** — rejected: it leaves a new install with
   no accounts and no UI path to create one. Recovery requires the admin panel plus
   working SMTP, which is exactly the fragile dependency described above.
2. **`SIGNUP_DOMAINS_WHITELIST`** — rejected: it restricts *which* email domains may
   register, not *whether* an anonymous visitor may. On a personal server the owner's
   address is often at a large public provider, so whitelisting that domain would leave
   registration effectively open anyway.
3. **Auto-disabling signups after the first account** — rejected: Vaultwarden has no such
   setting, and a hook that rewrote the container's environment after first boot would be
   invisible to the user and would fight `restart: unless-stopped`.

---

## 2. No admin token — SSO in front of `/admin`

### Why it is necessary

**The admin token was a shared secret, not a secret.** The app previously set
`ADMIN_TOKEN: $APP_DEFAULT_PASSWORD` and printed it in `tips.before_install`.
`APP_DEFAULT_PASSWORD` is a PCS-wide value injected into *every* app's environment, so the
password manager's admin panel was gated by a credential that every other container on the
box already holds, in plaintext, in its own `docker inspect` output.

Upstream's `.env.template` documents exactly this configuration:

> `## Enable this to bypass the admin panel security. This option is only`
> `## meant to be used with the use of a separate auth layer in front`
> `# DISABLE_ADMIN_TOKEN=false`

AppShield is that layer. `/admin` is not in `ALLOWED_PATHS`, so every request to it is
authenticated against the PCS's Authelia SSO before Vaultwarden sees it, and there is no
longer any admin credential to store, print, or leak.

### Alternatives considered and rejected

- **Argon2 PHC hash for `ADMIN_TOKEN`** (upstream's `vaultwarden hash`). Keeps the
  plaintext out of `docker inspect`, but the plaintext *is* `APP_DEFAULT_PASSWORD`, which
  stays readable in every other app's environment. Hides the copy, not the secret.
- **A separate SSO-gated `vaultwardenadmin-` hostname, with a Caddy matcher blocking
  `/admin` on the main host.** Leaves the web vault untouched, but it fails *open*: the
  entire protection of `/admin` would rest on one Caddy label continuing to exist. It also
  needs a second AppShield container whose name is not the app name, which
  `auth-registrar` attests by PTR lookup.
- **Dropping the admin panel entirely** (leaving `ADMIN_TOKEN` unset disables it). Loses
  signup control, user management and diagnostics.

---

## 3. `ALLOWED_PATHS` instead of `OAUTH_RESOURCE`

### Why it is necessary

**Bitwarden clients cannot do an OIDC browser flow.** The mobile, desktop and
browser-extension clients authenticate with their own bearer tokens against a fixed
protocol surface. Putting that surface behind SSO breaks every client. `OAUTH_RESOURCE` is
not an alternative: it gates a path on tokens *AppShield* issues, and no Bitwarden client
will ever present one — the app's own auth is the credential here, and it is the same auth
model Bitwarden ships to the public internet.

The exempted set is the complete client-reachable route list for Vaultwarden 1.37.0, taken
from the mounts in `src/main.rs` (`/api`, `/admin`, `/events`, `/identity`, `/icons`,
`/notifications`) and the root routes in `src/api/web.rs` (`/attachments/…`, `/alive`,
`/app-id.json`, `/.well-known/apple-app-site-association`, plus the web-vault static
files):

| Prefix | Who needs it |
|---|---|
| `api/` | all clients — sync, ciphers, config, attachment metadata |
| `identity/` | prelogin, registration, `connect/token` |
| `notifications/` | SignalR push websocket |
| `events/` | client event collection |
| `icons/` | website favicons in the vault UI |
| `attachments/` | attachment download (authorised by a signed token in the URL) |
| `.well-known/` | iOS app-site-association |
| `app-id.json` | FIDO/U2F app id |
| `alive` | liveness |

`admin` is deliberately absent, and so is the web vault (`/`) and its static files.

### Security mitigations in place

- **The gate fails closed.** Everything not in `ALLOWED_PATHS` requires SSO. A missing
  entry breaks a client loudly and visibly; no mistake in this list can expose `/admin`.
- **The web vault is behind SSO too.** Opening the vault in a browser needs a PCS login
  *and then* the vault master password. Clients are unaffected.
- Backend carries no Caddy labels and publishes no host port — from outside the box it is
  reachable only through AppShield.
- Resource limits on both services; AppShield runs with Docker's default capability set.
- **`DOMAIN` is pinned** to `https://vaultwarden-$APP_DOMAIN`, so Vaultwarden issues its
  own links and WebAuthn origins against the gateway hostname rather than an
  attacker-supplied `Host` header.

---

## Data protection

- All state lives in `${DATA_ROOT:-/DATA}/AppData/vaultwarden/data/` and survives uninstall
  / reinstall and image upgrades. The front-door change needs no migration and does not
  touch the SQLite database or the RSA key; existing installs keep the same hostname and
  URLs, and every configured client keeps its connection.
- Vault contents are end-to-end encrypted under each user's master password. The server
  stores ciphertext only; neither the admin panel nor filesystem access to the PCS yields
  plaintext.
- `tips.before_install` opens by warning that a lost master password makes the vault
  permanently unrecoverable, so the user is told before install that recovery is not
  something the server operator can perform.

**Accepted residual risk, deliberate:** any PCS SSO account can open the admin panel, not
only the owner; and a container on the shared `pcs` network can reach
`http://vaultwarden-backend/admin` without credentials (the backend stays on `pcs` because
it has to resolve the `smtp` mail gateway). The second is not a regression — the shipped
`ADMIN_TOKEN` was `APP_DEFAULT_PASSWORD`, which those same containers already have. The
first is one setting away from being closed: `OIDC_REQUIRED_GROUPS: "admins"` on the
sidecar restricts the panel to the `admins` group (verified working — a non-admin PCS user
is bounced to the dashboard).

## Verification

Tested on wisera (`vaultwarden-wisera.inojob.com`) on 2026-09-07 against
`appshield:3.0.2` + `vaultwarden/server:1.37.0`. 24/24 checks: `/`, `/admin`,
`/admin/users`, `POST /admin` and `/vw_static/*` all 302 to `/nhl-auth/oidc/login`; the
full client flow (register → `connect/token` → `/api/sync` → create cipher → attachment
upload → unauthenticated attachment download by URL token → `/events/collect`) all reached
Vaultwarden; the push websocket upgraded through the gateway and the sidecar
(`101 Switching Protocols`). After SSO the admin panel renders, its assets load, and a
write action (`POST /admin/users/<id>/deauth`) returns 200. Vaultwarden's own Diagnostics
page reports `Domain configuration: Match`, `Websocket enabled: Yes`,
`IP header: Match (X-Real-IP)` and `HTTP Response validation: Ok` behind the sidecar.
