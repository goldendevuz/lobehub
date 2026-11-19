# Personal deployment (single user over a private network)

A trimmed copy of `../deploy` for a single-user install that is only reachable
over a private network (Tailscale). Differences from `../deploy`:

- every published port binds to `LOBE_BIND_IP` instead of `0.0.0.0`;
- PostgreSQL and Redis publish no host port at all;
- `restart: unless-stopped` instead of `always`;
- the app image is pinned through `LOBE_IMAGE`;
- the model provider points at an OpenAI-compatible gateway (OmniRoute) through
  `OPENAI_PROXY_URL`, and `OPENAI_MODEL_LIST` exposes only its router models.

## First start

```bash
cp .env.example .env
```

Generate the secrets:

```bash
# base64 secrets (KEY_VAULTS_SECRET, AUTH_SECRET)
openssl rand -base64 32
# gateway service token
openssl rand -hex 32
# RS256 key set: prints private key set, then the public key set
node -e 'const c=require("crypto");const {privateKey,publicKey}=c.generateKeyPairSync("rsa",{modulusLength:2048});const meta={alg:"RS256",kid:c.randomBytes(8).toString("hex"),use:"sig"};console.log(JSON.stringify({keys:[{...privateKey.export({format:"jwk"}),...meta}]}));console.log(JSON.stringify({keys:[{...publicKey.export({format:"jwk"}),...meta}]}))'
```

Put the first line into `JWKS_KEY`, the second into `JWKS_PUBLIC_KEY` (single
line each, no quotes), then fill `POSTGRES_PASSWORD`, `RUSTFS_SECRET_KEY`,
`OPENAI_API_KEY` and the host addresses.

```bash
docker compose up -d --wait
docker compose ps
```

Open `APP_URL`, create the account, then set `AUTH_ALLOWED_EMAILS` to that
address and restart:

```bash
docker compose up -d
```

## Operations

```bash
docker compose ps                    # status
docker compose logs -f lobe          # app logs
docker compose restart lobe          # restart the app only
docker compose pull && docker compose up -d --wait   # update to the pinned image
```

Upgrade by changing `LOBE_IMAGE` in `.env` (and, when the app changes, the
database migrations run on start).

Rollback: set `LOBE_IMAGE` back to the previous tag and `docker compose up -d`.
Database migrations are forward-only, so restore `./data/postgres` from a
backup when a rollback crosses a schema change.

## Backup

```bash
docker compose exec -T postgresql pg_dump -U postgres lobechat | gzip > lobehub-$(date +%F).sql.gz
tar czf lobehub-data-$(date +%F).tar.gz data
```

The bucket lives in the `rustfs-data` volume; copy it with
`docker run --rm -v lobehub-personal_rustfs-data:/data -v "$PWD":/backup alpine tar czf /backup/rustfs-$(date +%F).tar.gz -C /data .`
