# MetaWatch — Rationale

## What deviation / exception is being requested

1. **Three services run as `user: 0:0` (root):** `metawatch-core`,
   `metawatch-search` and `metawatch-share`. The user-facing service
   (`metawatch-app`) and the AppShield gate do **not**.
2. **Two non-HTTP host ports are published:** TCP `4003` and `4004`.
3. **Two services sit on the shared `pcs` network without Caddy labels:**
   `metawatch-search` and `metawatch-share`.
4. **The app fetches and re-shares third-party content** from a public
   peer-to-peer network.
5. **`ALLOWED_PATHS` on the AppShield gate** exempts three static JS files from
   the SSO gate: `sw.js`, `js/downloads/cache.js` and `js/downloads/opfs.js`.

No volume reaches outside `/DATA/AppData/$AppID/`. The app mounts none of
`/DATA/Documents`, `/DATA/Downloads`, `/DATA/Media` or `/DATA/Gallery`, and
reads no other app's data.

## Why it is necessary

**Root is about one shared volume, not privilege.** The three services share a
single `/meta-core` volume: `metawatch-core` runs the Redis instance and owns
the lock and service-registry files in it, and the two peers read the leader
info and write their own registration files beside it. They are separate
containers writing one directory as one identity, so they must agree on a uid.
meta-core creates that tree on first boot before any of them could be chowned
into place. `metawatch-app` — the only service reached by a browser, and the
only one parsing user input — is unprivileged (`$PUID:$PGID`) with its own
private data dir.

Note this core is deliberately **not `privileged`**, unlike the standalone
MetaCore app. That app needs it for rclone remote mounts; MetaWatch's core runs
with `ENABLE_FILE_WATCHER=false`, mounts no media, and performs no mounts.

**The published ports are the network.** MetaWatch is a peer, not a client of a
server: `4003` (content transport) and `4004` (discovery) are raw libp2p TCP.
libp2p speaks its own multiplexed, Noise-encrypted protocol — it cannot be
carried through an HTTP reverse proxy, so there is no way to fold it behind
AppShield the way the web UI is. Without inbound reachability the app still
installs, starts and works for browsing and playback; it simply cannot serve
bytes to anyone else. That is what the `needs-public-ip` tag declares, and
`tips.before_install` says it in plain words.

**`pcs` without Caddy labels is for app-to-app reach, not browser reach.** The
two peers join the shared network so they can find and query a co-located
MetaGateway — a companion app on the same box — including by mDNS. Neither has
Caddy labels and neither is published on a hostname, so no browser can reach
either through the perimeter. Cross-host discovery rides the public DHT and
needs no shared network at all.

**The three exempted paths are the service-worker script and its imports.**
MetaWatch registers a **module** service worker
(`navigator.serviceWorker.register("/sw.js", { type: "module" })`) — it owns the
offline app shell, the OPFS byte-range shim that lets the player read a
downloaded title with no network, and the offline downloads/range-seek
subsystem the app advertises. The
script fetch that `register()` performs does **not** present the session cookie,
so behind an unqualified gate it is 302'd to `/nhl-auth/oidc/login`, and by spec
a redirected script resource is a fatal, unrecoverable `SecurityError`: no
worker ever installs, `getRegistrations()` stays empty, and the whole offline
tier is silently dead while browsing and server-proxied playback still work. A
module worker's **static import graph** must be fetchable the same way, which is
why the exemption is three entries and not one: `sw.js` imports
`js/downloads/cache.js` and `js/downloads/opfs.js`, and those two import nothing
further. Nothing else needs it — once installed, the worker's own `fetch()`
calls do carry the cookie, so the shell precache and the range shim go through
the gate normally.

**What is exempted is inert static app code, and nothing more.** The three files
are shipped, versioned JS assets served off the image — the same class as
`Apps/FileBrowser`'s `static/` exemption. They read no user data, hold no
secret, and are byte-identical for every installation, so an unauthenticated
fetch of them discloses only code the browser would hand any signed-in user
anyway. AppShield matches per URL segment, so `sw.js` exempts exactly `/sw.js`
(not `/sw.js.map`) and each `js/downloads/…` entry exempts exactly that one
file: the other nine modules in `js/downloads/` (`engine.js`, `store.js`,
`db.js`, …), the whole rest of `js/`, `index.html`, `/` and — critically —
**the app's own `/api/` second gate all stay behind SSO**. The pre-auth 401s the
app returns on `/api/prefs/languages`, `/api/avatars` and `/api/account` are
unchanged.

**`OAUTH_RESOURCE` cannot do this job.** The store's rule is to prefer
`OAUTH_RESOURCE` over `ALLOWED_PATHS` because the latter leaves a path reachable
with no credential at all. That rule is aimed at *machine/API* clients, which
can present a Bearer token. The client here is the browser's own service-worker
installer: it fetches the script with no `Authorization` header and no
opportunity to run an OAuth 2.1 flow, and a 401 is as fatal to registration as a
302. There is no token-bearing form of this request, so `OAUTH_RESOURCE` would
leave the finding exactly where it is.

**The content behaviour is the product.** MetaWatch discovers what other peers
publish and, having fetched bytes, seeds them back — that reciprocity is what
makes the network work. It ships no indexer, no tracker and no content of its
own, and it is disclosed twice before install: in the app `description` and in
`tips.before_install`.

## Security mitigations in place

- **Two independent gates.** AppShield (Authelia SSO) fronts the whole app, and
  MetaWatch keeps its own profile sign-in behind it. A profile is a secp256k1
  keypair: signing in means signing a challenge in the browser, and the secret
  key never reaches the server. Sign-out is real (tokens are revoked
  server-side), not just a cleared cookie.
- **The `ALLOWED_PATHS` exemption is three exact files, not a prefix.** It is
  enumerated file-by-file rather than opening `js/downloads/`, so it covers the
  worker's static import graph and stops there; it was verified against
  AppShield 3.0.2 (`/sw.js`, `/js/downloads/cache.js`, `/js/downloads/opfs.js`
  → 200; `/js/downloads/engine.js`, `/index.html`, `/api/account`, `/` → 302 to
  the SSO login). Routing the three around the gate with `caddy_N.M_handle`
  labels instead was rejected: `metawatch-app` is deliberately **not** on the
  `pcs` network, so Caddy cannot reach it, and putting it there to make the
  labels work would expose the protected back-end to every other app on the box
  — a strictly worse trade than serving three static JS files anonymously.
- **Profile creation is invite-gated**, so a publicly reachable box does not
  offer "create a profile" as a button to anyone who finds the URL.
- **The attack surface facing the browser is the unprivileged service.** The
  three root services are reachable only from inside the app's own network; the
  two on `pcs` expose an HTTP API to sibling apps, not to the perimeter.
- **Memory is capped on every service** (`deploy.resources.limits.memory`), so a
  swarm peer that misbehaves cannot exhaust the host — 3 G on the transport
  peer, 1 G on discovery, 768 M on the core, 512 M on the app, 128 M on the
  gate.
- **All images are version-pinned**; none uses `:latest`.
- **The cache is bounded to one directory** (`share-data`), disclosed in
  `tips.before_install` so the owner knows what grows and where to clear it.
