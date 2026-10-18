# OpenSEO on the LemonScore server

`https://seo.lemon-score.ch`: OpenSEO behind Authelia (password + TOTP), on the
same host as the Lead Engine (`lemonscore-hub-prod`, 2 vCPU / 3.8 GB).

```
internet ─► Caddy (~/lemonscore, ports 80/443)
              └─ seo.lemon-score.ch  (~/lemonscore/sites/openseo.caddy)
                   ├─ /authelia/*  ─► authelia:9091   login page
                   └─ everything else: forward_auth ─► open-seo:3001
            network "edge": caddy, authelia, open-seo (not the Lead Engine app)
```

## Why it is built this way

- **No login in OpenSEO.** Docker mode is `AUTH_MODE=local_noauth`: whoever
  reaches port 3001 is admin and spends DataForSEO credits. Neither container
  publishes a port; only Caddy reaches them, and Caddy lets a request through
  only with an Authelia session that passed password **and** TOTP
  (`default_policy: deny`, rule `two_factor`).
- **Prebuilt image.** Upstream builds the app at every container start with a
  4 GB heap. That would push this host into swap and slow the Lead Engine. The
  `LemonScore image` workflow builds `deploy/lemonscore/Dockerfile` in CI with
  the build baked in, so a start only runs migrations and serves.
- **Limits.** OpenSEO is capped at 1.2 GB RAM and 1 CPU, Authelia at 256 MB;
  both drop all capabilities and run as non-root users.
- **Cookie scope.** The Authelia cookie is set for `seo.lemon-score.ch` only.
- **Brute force.** 5 failed logins in 10 minutes ban the user and IP for an hour.
  Password reset and change in the portal are off.
- **Telemetry** is off (`OPENSEO_TELEMETRY_DISABLED=1`), robots get `noindex`.

## Files on the server

```
~/open-seo/
  compose.yaml                  this folder's compose.yaml
  .env                          OPENSEO_DOMAIN, DATAFORSEO_API_KEY, optional keys (0600)
  authelia/configuration.yml    this folder's authelia/configuration.yml
  authelia/users.yml            the user and its argon2id hash (0600, uid 8000)
  authelia/secrets/             session_secret, storage_encryption_key, jwt_secret (0400, uid 8000)
  authelia/data/                db.sqlite3 (TOTP config, encrypted), notification.txt
~/lemonscore/sites/openseo.caddy  this folder's openseo.caddy
```

Credentials live in 1Password (vault "Lemon Juice", item "OpenSEO (seo.lemon-score.ch)"):
username, password and the one-time password.

## Update OpenSEO

1. Sync the fork with upstream (GitHub "Sync fork", or merge `every-app/main`).
2. The push to `main` runs **LemonScore image**; wait for it to finish.
3. On the server: `cd ~/open-seo && docker compose pull open-seo && docker compose up -d open-seo`

## Google Search Console and Analytics

Google Cloud project `openseo-512007` ("OpenSEO", organisation dariotschannen.ch,
owner dario@lemon-score.ch): Search Console API, Analytics Admin API and Analytics
Data API on; Google Auth Platform audience External; Web OAuth client
"OpenSEO seo.lemon-score.ch" with the redirects
`https://seo.lemon-score.ch/api/gsc/oauth/callback` and `/api/ga4/oauth/callback`.
Client ID, secret and the downloaded JSON are in 1Password ("OpenSEO Google OAuth
client"); `.env` has `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET` and
`BETTER_AUTH_SECRET` (generated on the server, encrypts the stored Google tokens;
changing it disconnects every Google account).

The app must be published ("In production"): in Testing only listed test users
can connect and their connections expire after 7 days. It is not verified by
Google, so each account sees "Google hasn't verified this app" once (Advanced →
continue); at most 100 accounts.

## MCP

MCP clients cannot do the Authelia login, so `https://seo.lemon-score.ch/mcp` takes a
bearer token instead, checked by Caddy (`~/lemonscore/sites/openseo-mcp.secret`, root
0600, from `openseo-mcp.secret.example`). Only `/mcp` opens with it; everything else
still needs password + TOTP. No file, or a missing or wrong token: 401.

The token is in 1Password, item "OpenSEO MCP token (seo.lemon-score.ch)". It is a single
factor with full admin rights (it can start paid DataForSEO jobs), so keep it in as few
client configs as possible. Claude Code:

```bash
claude mcp add --transport http --scope user openseo https://seo.lemon-score.ch/mcp \
  --header "Authorization: Bearer $(op read 'op://Lemon Juice/ejdentqm6ziqax6j7bogmesjfe/credential')"
```

Rotate: new 64-character hex token in the 1Password item, the same in the secret file,
reload Caddy, update the clients.

## Caddy

`~/lemonscore/docker-compose.yml` mounts `~/lemonscore/sites` into Caddy and puts
Caddy on the `edge` network; the Caddyfile ends with `import /etc/caddy/sites/*.caddy`
(LemonScore-Sales repo, so a Lead Engine deploy keeps it). After changing
`openseo.caddy` on the server:
`cd ~/lemonscore && docker compose exec caddy caddy reload --config /etc/caddy/Caddyfile`

DNS: `seo` A record to the server IP in Cloudflare, DNS only (grey cloud), like hub and club.

## Common tasks

- Logs: `docker compose logs -f open-seo authelia`
- Validate the Authelia config: `docker compose run --rm authelia authelia validate-config --config /config/configuration.yml`
- New TOTP secret (e.g. lost device): `docker compose exec authelia authelia storage user totp generate dario --force --config /config/configuration.yml`
  then put the printed `otpauth://` URI into the 1Password item.
- Unban after too many attempts: `docker compose exec authelia authelia storage bans user revoke dario --config /config/configuration.yml`
- New password hash: `docker compose run --rm -T authelia sh -c 'authelia crypto hash generate argon2 --password "$(cat)"'` (password on stdin), replace the `password:` line in users.yml, `docker compose restart authelia`.
