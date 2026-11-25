# Varsis with Docker Compose

The app's own Docker setup: PostgreSQL, the backend and the frontend (nginx) as containers, plus Caddy for HTTPS on
a server. Everything for it is in this folder: the compose files, the `Caddyfile`, and your settings in `.env`.
Run every `docker compose` command **in this folder** (`deploy/docker`): Compose reads the `.env` next to the
compose files, both for the `${...}` values in them and for the backend's settings.

## Run on your computer

```bash
cd deploy/docker
cp ../../.env.example .env        # then set SECRET_KEY, POSTGRES_PASSWORD, ADMIN_EMAILS (and SMTP for production)
docker compose --profile dev up --build
```

`--profile dev` adds Mailpit, which catches the emails the app sends (sign-up links). Leave it out in production,
where `.env` has real SMTP values.

All data is stored in folders under `DATA_DIR` (default `/varsis`): `postgres`, `music`, `covers`. Docker creates
them on first start (owned by root).

| Service | URL |
|---|---|
| App | http://localhost:8080 |
| Mailpit (dev emails, with `--profile dev`) | http://localhost:8025 |

## Deploy with HTTPS

On a server with a public address, add `docker-compose.https.yml` to the compose file:

```bash
docker compose -f docker-compose.yml -f docker-compose.https.yml up -d --build
```

It starts Caddy on ports 80 and 443, which gets a Let's Encrypt certificate for `DOMAIN` (set it in `.env`, e.g.
`DOMAIN=music.example.com`) and renews it. The app is then only reachable at `https://DOMAIN`; port 8080 is closed.
Before the first start, the domain's DNS record must point at the server and ports 80 and 443 must be open. Needs
Docker Compose 2.24 or newer.

## Deploying on a server, step by step

From an empty Linux server to `https://music.example.com`. Caddy (from `docker-compose.https.yml`) gets the
certificate and is the only thing reachable from outside. Replace `music.example.com`, `example.com` and
`SERVER_IP` with your own values.

### What you need

- A server with a public IP address where you can open ports 80 and 443. Nothing else may be listening on them
  (check with `sudo ss -tlnp | grep -E ':(80|443) '`).
- A domain whose DNS records you can edit.
- An account at a mail provider that offers SMTP (Resend, Brevo, Mailgun…). The app sends sign-up and
  password-reset mails.
- Enough disk for the music: lossless albums are 300-500 MB each, and every user may upload `DEFAULT_QUOTA_GB`.

### 1. DNS

Add an `A` record for `music.example.com` that points at `SERVER_IP`.

On Cloudflare, set the record to **DNS only** (grey cloud). Proxied (orange) records reject requests over 100 MB,
which breaks uploads: the upload page sends parts of up to 500 MB.

### 2. Email sender

1. At the mail provider, add a sending domain. A subdomain such as `mail.example.com` keeps it apart from other
   mail on the domain.
2. Add the DNS records the provider shows (SPF, DKIM) and wait until the provider marks the domain as verified.
   Without them, sign-up mails land in spam.
3. Create an SMTP password or API key. You need it in step 4.

### 3. Prepare the server

```bash
curl -fsSL https://get.docker.com | sudo sh      # Docker with the compose plugin
docker compose version                           # needs 2.24 or newer
sudo apt install -y git
sudo ufw allow OpenSSH && sudo ufw allow 80,443/tcp && sudo ufw enable
```

Also open ports 80 and 443 in the hosting provider's own firewall, if it has one. The commands below use
`docker compose` and run in `deploy/docker`; put `sudo` in front unless your user is in the `docker` group.

### 4. Configure

```bash
git clone https://github.com/grobolsek/varsis-app.git && cd varsis-app/deploy/docker
cp ../../.env.example .env
openssl rand -hex 32     # run twice: one value for SECRET_KEY, one for POSTGRES_PASSWORD
```

Set these in `.env` (in `deploy/docker`):

| Variable | Value |
|---|---|
| `SECRET_KEY`, `POSTGRES_PASSWORD` | the two values from `openssl`. Never reuse the ones from a development `.env` |
| `DOMAIN` | `music.example.com`, without `https://`. `APP_URL` is then ignored |
| `COMPOSE_FILE` | `docker-compose.yml:docker-compose.https.yml` (add this line), so that every `docker compose` command uses both files |
| `ADMIN_EMAILS` | your own address; more than one separated by commas |
| `DATA_DIR` | the folder for all data, on the disk with the most free space (`df -h` lists them). Default `/varsis` |
| `DEFAULT_QUOTA_GB` | upload space per user. Lower it on a small disk |
| `SMTP_HOST`, `SMTP_PORT`, `SMTP_USER`, `SMTP_PASS` | from the mail provider. Port 587 with `SMTP_TLS=true`, or port 465 |
| `SMTP_FROM` | an address on the sending domain from step 2, e.g. `varsis@mail.example.com` |

Example for Resend: `SMTP_HOST=smtp.resend.com`, `SMTP_PORT=587`, `SMTP_USER=resend`, `SMTP_PASS=<API key>`,
`SMTP_TLS=true`.

### 5. Start

```bash
docker compose up -d --build
docker compose ps                         # backend and db turn "healthy" after a few seconds
docker compose logs caddy | grep -i certificate
curl -I https://music.example.com         # HTTP/2 200
```

Caddy asks Let's Encrypt for the certificate on the first start; this takes up to a minute and needs step 1 to be
done. The certificate is kept in `DATA_DIR/caddy` and renewed automatically.

### 6. First login

1. Open `https://music.example.com` and sign up with **every** address in `ADMIN_EMAILS` straight away, then click
   the link in the confirmation mail. An admin address without an account can be registered by anyone.
2. Check that the mail arrived in the inbox, not in spam.
3. Test from a phone on mobile data: log in, upload an album, play a track and seek in it.

Other people sign up themselves; you approve them under account menu → **Admin**.

## Updating

```bash
git pull                     # in deploy/docker
docker compose up -d --build
```

## Backups

All state is in `DATA_DIR` (`postgres`, `music`, `covers`, `caddy`) and in `deploy/docker/.env`.

```bash
docker compose exec -T db pg_dump -U varsis varsis | gzip > varsis-db-$(date +%F).sql.gz
```

Copy that file, `DATA_DIR/music`, `DATA_DIR/covers` and `.env` to another machine (for example with `rsync`), on a
schedule. To restore on an empty `DATA_DIR`: `docker compose up -d db`, then
`gunzip -c varsis-db-<date>.sql.gz | docker compose exec -T db psql -U varsis varsis`, copy `music` and `covers` back,
then `docker compose up -d`.

## Moving to another server

1. Do steps 3 and 4 on the new server, but copy the old `deploy/docker/.env` over instead of writing a new one
   (`POSTGRES_PASSWORD` has to match the copied database).
2. On the old server: `docker compose down`.
3. Copy the whole `DATA_DIR` to the same path on the new server, as root so that file owners are kept:
   `sudo rsync -a /varsis/ root@NEW_SERVER:/varsis/`.
4. Point the DNS record at the new server (step 1), then start it (step 5).

## When something doesn't work

| Symptom | Cause |
|---|---|
| The browser can't find the site, but `dig +short music.example.com @1.1.1.1` prints the server's address | Your own DNS resolver looked the name up before the record existed and remembers that for up to 30 minutes. Wait, or try from a phone on mobile data |
| No certificate; `docker compose logs caddy` shows failed challenges | The DNS record doesn't point at this server (or is proxied), a typo in `DOMAIN`, or port 80/443 is closed in a firewall |
| `caddy` doesn't start: "address already in use" | Another web server holds port 80 or 443: `sudo ss -tlnp \| grep -E ':(80\|443) '`, then stop it |
| The backend keeps restarting | `docker compose logs backend`: usually a `SECRET_KEY` shorter than 32 characters, or `SMTP_USER` set without `SMTP_TLS=true` |
| No sign-up mail | `docker compose logs backend`; check the SMTP values and that the sending domain is verified at the provider |
| Uploads over 100 MB fail | The DNS record is proxied by Cloudflare. Set it to DNS only |
| Changing `POSTGRES_PASSWORD` broke the app | The password is only applied when the database is first created. Change it inside too: `docker compose exec db psql -U varsis -c "ALTER USER varsis PASSWORD '...'"` |
