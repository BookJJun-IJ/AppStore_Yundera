# qBittorrent — Rationale

## What deviation / exception is being requested

The three Caddy site blocks do not proxy with a single `reverse_proxy`. Each one
defines a named matcher and two `handle` branches:

```
@tilenav {
  method GET
  path /
  header Sec-Fetch-Mode navigate
}
handle @tilenav {
  reverse_proxy qbittorrent:80 {
    header_up -Referer
    header_up -Origin
  }
}
handle {
  reverse_proxy qbittorrent:80
}
```

So the `Referer` and `Origin` request headers are removed — **only** on a
top-level browser navigation to `/`. Every other request reaches qBittorrent with
its headers untouched.

## Why it is necessary

qBittorrent's WebUI has a built-in cross-site check
(`WebApplication::doProcessRequest`). With `WebUI\CSRFProtection=true` it answers
`401 Unauthorized` — `text/plain`, 12 bytes, no HTML — to any request that still
needs authentication and whose `Referer`/`Origin` host differs from the host it
was served on.

The Maison dashboard is a different host from the app (`https://<user>.<domain>/`
vs `https://qbittorrent-<user>.<domain>/`), so the tile's **Open** action is
exactly such a request: a cross-site navigation by an unauthenticated visitor.
The user clicks the tile and gets the bare string `Unauthorized` instead of the
login page. Reloading does not help — Chrome replays the original referrer — so
the only way in is to retype the URL by hand. The tile is the app's primary entry
path, and it was broken (Touchstone `works-immediately`, MAJOR).

The request that has to be unpicked is fully identified:

| | tile navigation | everything else |
|---|---|---|
| method | `GET` | `POST`/`GET`/… |
| path | `/` | `/api/v2/…`, assets |
| `Sec-Fetch-Mode` | `navigate` | `cors` (XHR), `no-cors`/`same-origin` (assets) |

Matching all three and stripping the two headers there makes the tile land on the
login page, and changes nothing else.

## Security mitigations in place — why CSRF protection is still intact

The attack the `enable-csrf-protection` init step exists to stop is a cross-site
`POST /api/v2/app/setPreferences` setting `autorun_program`, i.e. arbitrary
command execution. That request is **not** affected by this exception:

- **The matcher requires `method GET`.** No mutating request can match it. Every
  qBittorrent API call that changes state is a `POST` and falls to the second
  `handle`, which forwards `Origin`/`Referer` verbatim; qBittorrent's own check
  sees them and rejects the cross-site ones exactly as before.
- **The matcher requires `path /`** — an exact match, not a prefix. Nothing under
  `/api/v2/` can reach the stripping branch even as a `GET`.
- **The matcher requires `Sec-Fetch-Mode: navigate`.** The WebUI drives the API
  over `fetch`/XHR, which browsers label `cors` or `same-origin`; only a
  top-level document navigation is labelled `navigate`, and a browser sets this
  header itself — page JavaScript cannot forge it (it is a forbidden header
  name). An attacker's page therefore cannot make its API call look like a tile
  click.
- **What the exception actually grants an attacker is nothing.** The only thing
  it lets a cross-site context do is fetch `/` — the login page — with the
  referrer hidden. `GET /` has no side effect, and the response is opaque to the
  attacker under the same-origin policy; qBittorrent already serves that same
  page to a request with no `Referer` at all (an address-bar visit), which is why
  typing the URL worked while the tile did not.
- Authentication itself is unchanged: the WebUI still requires the `admin`
  password seeded from `$APP_DEFAULT_PASSWORD`, and the login `POST` is
  same-origin once the page has loaded on the app's own host.

Net effect: the pre-authentication cross-site *read* of `/` is allowed (it always
effectively was), and the cross-site *write* is still blocked by qBittorrent
itself.

## Alternatives considered and rejected

- **Strip `Referer`/`Origin` unconditionally on the whole site** (a plain
  `header_up -Referer; header_up -Origin` on the single `reverse_proxy`). This is
  the obvious fix and it is wrong: qBittorrent's check passes a request that
  carries no `Origin`/`Referer` at all, so stripping them everywhere is
  functionally identical to setting `WebUI\CSRFProtection=false` — it reopens the
  `setPreferences`/`autorun_program` RCE, with only the browser's SameSite cookie
  default left in front of it. Rejected.
- **Revert `enable-csrf-protection` and police cross-site writes in Caddy
  instead** (deny when `Sec-Fetch-Site` is not `same-origin` and the method is
  not `GET`/`HEAD`). It is a coherent design, but it is worse here on three
  counts: (1) it fails open for any client that sends no `Sec-Fetch-Site` — which
  is every non-browser client, so the rule must either allow header-less writes
  (and the defence evaporates for an attacker who can suppress the header) or
  break the *arr integrations and mobile clients that drive `/api/v2` with no
  fetch-metadata at all; (2) it moves the defence off the app and onto the
  perimeter, so anything reaching the container directly on the `pcs` network is
  unprotected, whereas qBittorrent's own check is always on; (3) it needs a third
  init step to write `CSRFProtection=false` back, touching a config file the user
  may since have changed. Rejected in favour of keeping the app's own defence and
  making one precisely-scoped hole in the proxy in front of it.
- **Point the tile somewhere else** (a `webui-path` that qBittorrent treats
  differently). There is no such path: the check is on authentication state, not
  on the URL, so every unauthenticated entry point behaves identically.
- **Leave it and tell users to retype the URL.** That is the finding, not a fix.

## Data protection

Unchanged by this exception. All state stays in
`/DATA/AppData/qbittorrent/config/` (declared under `x-compose-app.folders`),
downloads in `/DATA/Downloads/` (disclosed in `tips.before_install`), the
container runs as `$PUID:$PGID`, memory is capped at 512M and `cpu_shares: 50`.
Both init steps are guarded on files inside the config volume, not on `once`, so a
reinstall never resets the WebUI password or the user's preferences.
