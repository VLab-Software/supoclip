# Coolify + Antigravity Manager (self-hosted)

This Compose configuration uses `gpus: all` on the worker. Docker Compose documents that field starting at v2.30.0; the host also needs a configured NVIDIA Container Toolkit/runtime and compatible GPU driver. Coolify supports GPU device requests, but the installed Coolify/Docker Compose versions on the target server have not been checked.

The Compose-defined `supoclip-network` provides internal service discovery within the stack. Coolify can deploy custom Compose networks, but `supoclip-network` is not Coolify’s cross-resource shared network. Another resource is not connected to it automatically; enable Coolify’s “Connect To Predefined Network” for the `coolify` network on both stacks, then verify the Antigravity alias and reachability.

This Compose configuration is prepared for a manual Coolify deployment from `VLab-Software/supoclip`, branch `main`, Compose build pack, `/docker-compose.yml`. No deployment or server changes are performed by this guide.

## Coolify environment

Set secrets in the Coolify resource's environment panel. Never commit generated values. Required values:

- `POSTGRES_PASSWORD`: generate a strong URL-safe password (or percent-encode reserved URL characters in the two connection URLs).
- `FRONTEND_DATABASE_URL`: `postgresql://supoclip:<password>@postgres:5432/supoclip`.
- `DATABASE_URL`: `postgresql+asyncpg://supoclip:<password>@postgres:5432/supoclip` (backend, worker, cron and AI worker).
- `REDIS_PASSWORD`: a separate strong password. Compose enables Redis `requirepass`, and clients use it.
- `BETTER_AUTH_SECRET`, `BACKEND_AUTH_SECRET`, `APP_SETTINGS_ENCRYPTION_KEY`: three distinct random secrets. Each needs at least 16 non-whitespace characters; use 32 random bytes or more. The encryption key is hashed to an AES-256 key by the application. Keep it stable: changing it prevents decrypting stored admin settings.
- `ANTIGRAVITY_API_KEY`: the private key configured in Antigravity Manager.

Set `SELF_HOST=true`, `FRONTEND_PORT=3107`, `FRONTEND_BIND_IP=192.168.1.109`, `BACKEND_BIND_IP=192.168.1.109`, `BACKEND_PORT=8001`, `NEXT_PUBLIC_API_URL=http://192.168.1.109:8001`, and `BACKEND_PUBLIC_URL` to the same API URL, `NEXT_PUBLIC_APP_URL=http://192.168.1.109:3107`, `BETTER_AUTH_URL` to the same origin, `BETTER_AUTH_TRUSTED_ORIGINS` to that origin, and `CORS_ORIGINS` to that origin. `NEXT_PUBLIC_*` values are baked into the frontend image during build; rebuild after changing them.

Use `ANTIGRAVITY_BASE_URL=http://antigravity-manager:8045/v1` only after connecting both Coolify resources to a common network and confirming the manager alias. Coolify Compose resources get resource-specific networks by default; enable “Connect To Predefined Network” (normally `coolify`) for SupoClip and the manager, and verify the manager’s network alias before using it. Do not assume the alias is `antigravity-manager`. Query `GET <base-url>/models` with the API key on a trusted machine, then set `LLM=antigravity:<exact-model-id>`. The committed example still has a model-ID placeholder; an authenticated model listing is pending. SupoClip sends OpenAI-compatible chat completions to that endpoint. This provider does not have an automatic OpenRouter fallback; no paid external fallback is selected.

For the initial constrained host, use `TRANSCRIPTION_PROVIDER=youtube_captions`, `VIDEO_JOB_USER_CONCURRENCY=1`, `MAX_CLIPS=5`, `FAST_MODE_MAX_CLIPS=5`, `MAX_VIDEO_UPLOAD_BYTES=1073741824`, and `YOUTUBE_DOWNLOAD_MAX_BYTES=1073741824`. YouTube captions only apply to YouTube sources where captions are available; uploads and other source types need AssemblyAI or Whisper. The MCP service is optional and starts only with `COMPOSE_PROFILES=mcp`. Cron jobs are embedded in the main worker unless `EMBED_CRON_WORKER=0`; the separate cron service is opt-in with the `cron` profile. AI editing uses a separate lightweight `ai-worker`.

## Network and firewall

The frontend publishes 3107 and the API publishes host port 8001 bound to `192.168.1.109`; the API listens on container port 8000. PostgreSQL (5432), Redis (6379), and MCP (9100) remain unpublished. The API base URL is used by browser API calls and direct uploads, including multipart and upload completion/abort/parts endpoints. Frontend server-side requests continue to use `backend:8000`. Restrict TCP 8001 to `192.168.1.0/24` in the host Docker firewall (`DOCKER-USER` or equivalent); binding to a host address alone does not restrict client source addresses.

Docker published ports can bypass ordinary UFW filtering. Before deployment, configure the host's `DOCKER-USER` chain or equivalent firewall policy to allow TCP 3107 and 8001 only from `192.168.1.0/24` and reject other sources, while preserving Coolify's own management/proxy rules. Review the effective nftables/iptables rules after any Docker or Coolify network changes. This repository does not change firewall rules.

## Persistence, backup and restore

Compose declares named local volumes `postgres_data`, `redis_data`, `uploads`, and `clips` (plus optional MCP/log data). Coolify must retain these volume names across redeploys; do not switch to ephemeral storage or run `docker compose down -v`. Local media requires both `uploads` and `clips` to persist on the worker host.

Back up PostgreSQL with a consistent `pg_dump -Fc` and separately archive `uploads` and `clips` while writes are stopped or through a consistent filesystem snapshot. Redis is queue/cache state and is not a substitute for the database or media backup. Keep the auth/encryption secrets in the same protected backup inventory. For restore, stop application writers, restore the named volumes and PostgreSQL dump, restore the original secrets, then start services and verify health and sample media access.

## Pre-deploy checklist

- [ ] Configure all required secrets and both driver-specific database URLs in Coolify.
- [ ] Confirm any shared Antigravity network and `/v1/models` model ID; test authenticated access from the SupoClip network.
- [ ] Confirm LAN firewall allows TCP 3107 only from the intended subnet; PostgreSQL and Redis have no host port mappings.
- [ ] Confirm named volumes survive a Coolify redeploy and backups can be restored.
- [ ] Rebuild frontend after setting public URL/build-time variables.
- [ ] Validate healthchecks and signup policy (`DISABLE_SIGN_UP=false` only for initial account creation, then `true` if desired).
- [ ] Test YouTube captions on a captioned YouTube source; test a non-captioned source/upload only after configuring AssemblyAI or Whisper.
- [ ] Review Coolify logs, storage growth and memory usage before raising concurrency or upload limits.
