# Notes for coding agents

Rehike is a PHP server application (not a browser extension) that renders the
classic YouTube (Hitchhiker) UI from InnerTube data. Every request enters
through `index.php`; `router.php` maps paths to controllers.

## Running it here

```sh
docker compose -f docker-compose.base44.yml up -d --build
```

- Single service `web`: `php:8.3-apache` (see `Dockerfile.base44`) serving the
  bind-mounted checkout at `/var/www/html` on host port 3000.
- PHP is interpreted per request, so source edits are live — no rebuild and no
  asset build step is needed. Only `Dockerfile.base44` / compose changes need
  `up -d --build`.
- `vendor/` (Composer deps: Twig, protobuf, BlueLibraries DNS) is committed, so
  no `composer install` is required.
- No external credentials or `env_file` are needed: the app only talks to
  public YouTube endpoints.

## Non-obvious setup facts

- **PHP >= 8.1 is required** (`Fiber` is used by the network layer).
- **ext-sockets is mandatory for boot.** `Rehike\Boot\Tasks::initNetwork()`
  resolves `www.youtube.com` itself through `Util\Nameserver`. Its bundled-DNS
  strategy needs ext-sockets; without it the only fallback demands a CNAME
  record, and resolvers that answer `www.youtube.com` with A records directly
  (as this sandbox's do) make boot fail with `DnsLookupException`.
- **Writable paths in the web root.** On first run Rehike writes `config.json`,
  `cache/` and `logs/` next to the source (all gitignored). The bind mount is
  root-owned while Apache runs as `www-data`, so the compose command seeds
  `config.json` with `{}`, `chmod 777`s `cache`/`logs` and adds `o+x` on the
  document root. An *empty* `config.json` is rejected by
  `ConfigManager\Config::_loadConfig()`, which is why it is seeded with `{}`
  rather than `touch`ed.
- **Apache needs `AllowOverride All`** because routing relies on the repo's
  `.htaccess` (`RewriteRule . index.php [L]`), not just `mod_rewrite`.
- Health endpoint used by compose: `/generate_204` — it runs the full boot
  (config, i18n, DNS bootstrap) and exits 204.

## Verifying

```sh
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:3000/            # 200, YouTube masthead + guide
curl -s -o /dev/null -w '%{http_code}\n' 'http://localhost:3000/results?search_query=music'
curl -s -o /dev/null -w '%{http_code}\n' 'http://localhost:3000/watch?v=jNQXAC9IVRw'
```

A fatal error renders an HTML page (HTTP 200) titled `Rehike fatal error` with a
`<pre class="exception-log">` stack trace — grep for `exception-log` when a page
looks broken. There is no test suite or linter configured; `php -l <file>` runs
inside the container (`docker compose -f docker-compose.base44.yml exec -T web php -l <file>`).

## Known limits of this environment (not app bugs)

YouTube bot-gates datacenter IPs, which this sandbox is:

- `/youtubei/v1/player` answers `LOGIN_REQUIRED` ("Sign in to confirm you're not
  a bot"), so the watch page shows that message instead of a video.
- Because of that, a signed-out homepage has no recommendations (YouTube stopped
  serving them without watch history in 2024, and Rehike's visitor-data
  workaround depends on playback tracking that the player response no longer
  includes here).
- `/youtubei/v1/att/get` returns 403, so the archived YouTube client JS cannot
  finish initialising: the masthead's JavaScript search button does nothing.
  Typing a query and pressing Enter submits the plain `/results` form and works.

## App fixes carried for YouTube API drift

This branch predates recent YouTube changes; boot and page rendering rely on:

- `modules/Rehike/Player/PlayerUpdater.php` — the embed page now advertises a
  `player_embed` build; rewriting it to `player_ias` (instead of `player`) yields
  a build that actually exists and contains the signature timestamp.
- `modules/Rehike/Boot/Tasks.php` — `populateVisitorDataRecommendations()` skips
  its playback-stats ping when the player response carries no
  `playbackTracking`.
- `models/Guide/Converter.php` — the watch-history guide entry is looked up in
  any guide section instead of assuming `items[1]` (InnerTube now interleaves a
  sign-in promo renderer there).
