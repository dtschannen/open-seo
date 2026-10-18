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

## Common tasks

- Logs: `docker compose logs -f open-seo authelia`
- Validate the Authelia config: `docker compose run --rm authelia authelia validate-config --config /config/configuration.yml`
- New TOTP secret (e.g. lost device): `docker compose exec authelia authelia storage user totp generate dario --force --config /config/configuration.yml`
  then put the printed `otpauth://` URI into the 1Password item.
- Unban after too many attempts: `docker compose exec authelia authelia storage bans user revoke dario --config /config/configuration.yml`
- New password hash: `docker compose run --rm -T authelia sh -c 'authelia crypto hash generate argon2 --password "$(cat)"'` (password on stdin), replace the `password:` line in users.yml, `docker compose restart authelia`.
