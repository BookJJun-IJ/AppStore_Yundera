# wger — Rationale

## What deviation / exception is being requested

1. The app authenticates with its own Django login rather than being fronted by the
   AppShield SSO sidecar.
2. Two services run as `user: 0:0` — the nginx front-end and PostgreSQL.
3. The app is shipped without upstream's `powersync` service.

## Why it is necessary

**1. The app's own login.** wger is a per-user training log: every routine, workout
log, body-weight entry and meal belongs to a Django user, and the app has no concept
of an anonymous session that a proxy could stand in for. So a user account is created
either way, and putting AppShield in front adds a *second* credential in front of a
login the user still has to pass. The store's checklist accepts "the app's own
built-in auth (e.g. Jellyfin, Immich onboarding)" as an alternative, which is what
this is: the login gate is on by default, on every route, and the shipped credential
is the deployment's generated `$APP_DEFAULT_PASSWORD`, not a published default.

Two settings that would weaken that gate are turned off explicitly, because upstream's
own defaults enable them:
- `ALLOW_REGISTRATION=False` — the app's default is `True`, which would put an open
  sign-up form on a public URL.
- `ALLOW_GUEST_USERS=False` — the app's default is `True`, which hands an
  unauthenticated visitor a working session and a temporary account.

Brute-force protection (`django-axes`, 10 failures then a 30-minute cool-off) is left
enabled, as upstream ships it.

**2. `user: 0:0` on two services.**
- `wger` (nginx): the master process writes `/var/run/nginx.pid` and `/var/cache/nginx`,
  neither writable by an unprivileged uid in `nginx:alpine`; the workers drop to the
  `nginx` user by themselves. This is the same arrangement as `Apps/Nginx` and
  `Apps/Plex`'s front-end in this store.
- `wger-db` (postgres): the official entrypoint starts as root only to `chown` `PGDATA`
  and then re-executes as the `postgres` user. Same as `Apps/Spliit`.

The application containers (`wger-web`, `wger-worker`, `wger-beat`) run unprivileged as
`$PUID:$PGID`, which is also the uid the image's own `wger` user has.

**3. No `powersync`.** Upstream's compose includes `journeyapps/powersync-service`,
which backs *offline* mode in the phone app. It needs a second PostgreSQL role, logical
replication on the database, its own config tree and roughly another half-gigabyte of
memory. The phone app authenticates and syncs against the REST API without it — which
is why the JWT signing keypair is still generated (see below) — so the cost did not
look proportionate for a single-user server. Upstream's `config/nginx.conf` declares
`powersync` as an nginx `upstream`, and nginx exits at startup on an upstream it cannot
resolve, so the shipped `seed/config/nginx.conf` has that block removed rather than
merely unused.

## Security mitigations in place

- Only the nginx front-end is on the shared `pcs` network. The application, database,
  cache and both celery processes are on an app-private bridge and are not addressable
  from any other app on the box.
- No host port is published by any service.
- Every credential is generated per deployment: `WGER_SECRET_KEY` and
  `WGER_DB_PASSWORD` through `x-compose-app.secrets`, the JWT signing keypair through
  the `pre_install` hook, and the admin password from `$APP_DEFAULT_PASSWORD`.
- wger seeds its first admin from a fixture as `admin` / `adminadmin` — a published
  upstream default — with no environment variable to change it. The
  `rotate-admin-password` init step replaces it once the app is up, and only when the
  password is still that default, so it cannot overwrite a password the owner has
  since chosen. It compares the stored hash directly rather than attempting a login,
  so it never contributes to a `django-axes` lockout.
- Both nginx mounts (`static`, `media`) are read-only, and nothing outside
  `/DATA/AppData/wger` is mounted into any container.
- Resource limits are set on all six services (64M … 1G, ~2.6G in total).

## Alternatives considered and rejected

- **AppShield in front of the whole app.** Rejected: two logins for one person, for
  the reason above. AppShield 3.x no longer collides with an app's own `/login`, so
  this is a usability call rather than a technical block.
- **AppShield plus wger's `AUTH_PROXY_HEADER`**, which exists precisely to accept an
  identity from an authenticating reverse proxy and would give real single sign-on.
  Rejected for now: it is only safe when `AUTH_PROXY_TRUSTED_IPS` pins the proxy, and
  on a Docker bridge the sidecar's address is assigned from the subnet at start, so the
  pin would have to name the whole subnet — which on a shared network means any
  container could assert any identity. Worth revisiting if AppShield gains a signed
  identity assertion the app can verify (3.x has `IDENTITY_ASSERTION_SECRET`, which
  wger cannot currently consume).
- **Dropping nginx and letting gunicorn serve the app alone.** Rejected: Django with
  `DEBUG=False` serves neither `/static/` nor `/media/`, and wger does not bundle
  whitenoise, so the app renders unstyled and every exercise image 404s. Upstream's
  compose carries the same warning.
- **Dropping celery** (`USE_CELERY=False`). Rejected: it is what runs the weekly
  exercise-database refresh, the exercise-image downloads and the barcode scanner's
  on-demand ingredient fetch — the last of which upstream documents as requiring
  celery.
- **Upstream's bulk sync defaults.** `SYNC_INGREDIENTS_CELERY` and
  `SYNC_EXERCISE_VIDEOS_CELERY` are `True` upstream; both are off here. The ingredient
  dump is a multi-hundred-megabyte import and the videos are larger again, which is not
  a reasonable default for a personal server. Both are named in `tips.before_install`
  so the choice is visible, and the barcode scanner still fetches products one at a
  time via `DOWNLOAD_INGREDIENTS_FROM`.
- **`SYNC_EXERCISES_ON_STARTUP=True`**, the app's own way of populating the exercise
  database. Rejected after measuring it: it runs inside the app container's entrypoint,
  ahead of gunicorn, and re-walks every exercise on *every* start -- 5 minutes of
  downtime per restart, not a one-off first-boot cost. With it off, a restart is about
  80 seconds. The same work is queued onto celery once, at install, by the
  `seed-exercise-database` init step, which leaves the app answering while it runs.
- **Letting the exercise refresh arrive only via the weekly celery job.** Rejected as
  the sole mechanism: `add_periodic_task` picks a random day-of-week and time when beat
  starts (`wger/exercises/tasks.py`), so the pictures would be up to a week away on a
  fresh install. The init step queues both jobs immediately; the weekly schedule then
  keeps them current. Measured on a fresh install: the metadata sync finished in ~3.5
  minutes and the images in ~2 more (about 900 exercises, ~210 MB), with the app
  answering throughout. The image download is queued with an explicit
  `time_limit`/`soft_time_limit` as a guard rather than a routine need: upstream's
  fetches set no HTTP timeout, and a hand-run `download-exercise-images` was seen
  wedged on one request for over ten minutes. Unbounded, that would hold the only
  worker process — and every job behind it — until the app was restarted.

## Data protection

All state is under `/DATA/AppData/wger/` — `pgdata` (the database), `media` (uploaded
and downloaded exercise images and videos), `static` (regenerated on every start),
`redis` (cache and queue) and `beat` (the celery schedule) — so it survives uninstall
with "keep user data" and reinstall. The database password and Django secret key are
generated once into the app's `.env` and never regenerated, so an existing database
keeps opening and existing sessions keep working across restarts, updates and restores.
The JWT keypair is likewise generated only when absent: regenerating it would log out
every phone already paired with the server.

Version upgrades are handled by the app's own Django migrations, which run in
`wger-web` on start; the two celery containers are pinned to the same image but have
`DJANGO_PERFORM_MIGRATIONS=False` so that only one process ever applies a migration.
