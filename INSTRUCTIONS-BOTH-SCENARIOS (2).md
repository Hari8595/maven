# Instructions — Two-Application Deployment (Both Scenarios)

How to run App 1 + App 2 in Docker, with the two SQL dumps, the 2 missing
tables handled via Sequelize sync, logs persisted, heap tuned, and 24/7 self-healing.

---

## Step 1 — Lay out the folders and code

```
stack/
├── docker-compose.yml          (paste Scenario 1 OR 2 from the files document)
├── .env                        (shared infra values — step 3)
├── db/
│   ├── init/00-create-databases.sql   (Scenario 2 only)
│   └── dumps/
│       ├── app1-dump.sql       ← put dump 1 here
│       └── app2-dump.sql    ← put dump 2 here
├── app1/{backend,frontend}/    (Dockerfile, .dockerignore, .env, your code)
└── app2/{backend,frontend}/ (same)
```

Put your real code in each app folder; keep the Dockerfile + .dockerignore. Create an empty
`logs/` folder in each app folder so the volume mount has somewhere to write:
```bash
mkdir -p app1/backend/logs app1/frontend/logs \
         app2/backend/logs app2/frontend/logs
```

Confirm each backend's `package.json` has a start script (their entry is `app.js`):
```json
"scripts": { "start": "node app.js" }
```

---

## Step 2 — Fill the four `.env` files

Edit each app folder's `.env` (templates in the files document). The ONLY values that change
between scenarios are the DB host and Redis host:

| Variable | Scenario 1 (host) | Scenario 2 (docker) |
|---|---|---|
| `APP1_DB_HOST` / `APP2_DB_HOST` | `host.docker.internal` | `mysql` |
| `REDIS_HOST` | `host.docker.internal` | `redis` |

Keep every OTHER variable exactly as their app expects (CORS `ALLOWED_ORIGINS`, JWT, SendGrid,
Twilio, etc.). Lock them down:
```bash
chmod 600 app1/backend/.env app1/frontend/.env \
          app2/backend/.env app2/frontend/.env
```

> Use `openssl rand -hex 24` for passwords (no special characters — avoids MySQL/Redis errors).

---

## Step 3 — Shared infra `.env` (beside docker-compose.yml)

```bash
cat > .env << 'EOF'
MYSQL_ROOT_PASSWORD=change_me_hex
MYSQL_APP_PASSWORD=change_me_hex
REDIS_PASSWORD=change_me_hex
APP1_PUBLIC_API_BASE_URL=http://localhost:5001
APP2_PUBLIC_API_BASE_URL=http://localhost:5002
EOF
chmod 600 .env
```
(The app passwords here must match the DB/Redis passwords inside the four app `.env` files.)

---

## Step 4 — Docker starts on boot (reboot survival)
```bash
sudo systemctl enable docker
```

---

## Step 5 — Build and start
```bash
docker compose up -d --build
docker compose ps          # wait until all healthy
```

---

## Step 6 — Create databases + load the two dumps

### Scenario 2 (MySQL in Docker)
Databases are auto-created by the init SQL. Load each dump:
```bash
docker compose exec -T mysql mysql -u root -p"$MYSQL_ROOT_PASSWORD" app1_db    < db/dumps/app1-dump.sql
docker compose exec -T mysql mysql -u root -p"$MYSQL_ROOT_PASSWORD" app2_db < db/dumps/app2-dump.sql
```

### Scenario 1 (MySQL on host)
Create databases + user on the host MySQL, then load:
```bash
sudo mysql -e "CREATE DATABASE IF NOT EXISTS app1_db; CREATE DATABASE IF NOT EXISTS app2_db;"
sudo mysql -e "CREATE USER IF NOT EXISTS 'app_user'@'%' IDENTIFIED BY 'your_password';
  GRANT ALL PRIVILEGES ON app1_db.* TO 'app_user'@'%';
  GRANT ALL PRIVILEGES ON app2_db.* TO 'app_user'@'%'; FLUSH PRIVILEGES;"
mysql -u root -p app1_db    < db/dumps/app1-dump.sql
mysql -u root -p app2_db < db/dumps/app2-dump.sql
```
Also set `bind-address = 0.0.0.0` in the host MySQL config and restart it, so containers can reach it.

---

## Step 7 — Create the 2 missing tables (Sequelize sync, ONCE)

The dumps build most tables; 2 are missing. Turn sync on **once** to create them safely
(plain sync only creates missing tables — it never wipes data):

```bash
# 1) set DB_SYNC=true in BOTH backend .env files
sed -i 's/DB_SYNC=false/DB_SYNC=true/' app1/backend/.env app2/backend/.env

# 2) restart the backends so they run sync once
docker compose up -d app1-backend app2-backend

# 3) confirm the 2 tables now exist
docker compose exec mysql mysql -u root -p"$MYSQL_ROOT_PASSWORD" -e "USE app1_db; SHOW TABLES;"
# (Scenario 1: run the SHOW TABLES on the host MySQL instead)

# 4) turn sync back OFF and restart (so it doesn't sync on every boot)
sed -i 's/DB_SYNC=true/DB_SYNC=false/' app1/backend/.env app2/backend/.env
docker compose up -d app1-backend app2-backend
```

> This assumes the backend code runs `sequelize.sync()` only when `DB_SYNC=true`. If their
> code doesn't read such a flag, either add the small guard shown in the files document, OR
> run sync manually once via a small script, then remove it. NEVER use `force: true`.

---

## Step 8 — Verify

```bash
curl http://localhost:5001/health
curl http://localhost:5002/health
curl -I http://localhost:3001
curl -I http://localhost:3002
```
View in a browser via SSH tunnel (secure, no firewall change):
```
ssh -i "key.pem" -L 3001:localhost:3001 -L 3002:localhost:3002 -L 5001:localhost:5001 -L 5002:localhost:5002 <user>@<server-ip>
```
Then open `http://localhost:3001` and `http://localhost:3002`.

---

## Step 9 — Check logs and cron are working

```bash
# persisted log files on the host
ls -la app1/backend/logs/
tail -f app1/backend/logs/out.log

# docker-captured logs (look for the scheduler/cron starting)
docker compose logs -f app1-backend
```

---

## Step 10 — Prove 24/7 self-healing
```bash
docker compose kill app1-backend
docker compose ps        # down → ~10s → restarted itself (other apps keep running)
sudo reboot              # reconnect → docker compose ps → all back up
```

---

## Troubleshooting (everything seen before + new items)

| Symptom | Cause | Fix |
|---|---|---|
| `no space left on device` | disk full from images/cache | `docker system prune -af`; grow EBS if small |
| Redis `setpriv: setresuid failed` | over-dropped capabilities | add `cap_add:[SETUID,SETGID,SETPCAP,CHOWN,DAC_OVERRIDE]` |
| MySQL `Access denied for app_user` | stale volume / `${VAR}` not expanded | create user via image env; `docker compose down -v` (no real data yet) |
| `Failed to fetch` in browser | frontend calling unreachable API URL | fix `NEXT_PUBLIC_API_BASE_URL`, rebuild frontends |
| `read-only file system` writing logs | read_only blocked logs | logs volume is mounted (read_only is OFF on app containers) — keep it that way |
| Scenario 1: backend can't reach DB | host MySQL not listening / no grant | `bind-address=0.0.0.0`; grant `app_user'@'%'`; use `host.docker.internal` |
| 2 missing tables not created | sync didn't run / no privilege | ensure `DB_SYNC=true` once + `CREATE/ALTER` grants on the DB |
| cron job runs twice | backend scaled >1 | keep each backend at ONE replica |
| app OOM / killed | heap too low (runs at 91–95%) | raise `--max-old-space-size` and `mem_limit` together |
| CORS error in browser | allow-origin doesn't include the URL | set `ALLOWED_ORIGINS` in backend `.env` to the frontend URL |
| `*.gpg` / `.env.save` in image | stray secrets in working tree | `.dockerignore` blocks them; also remove from the repo |

---

## Security reminders (OWASP / CIS)
- Each `.env` is `chmod 600`, gitignored, and blocked from images; never `cat` on a shared screen.
- Move `microsoft.gpg` and `.env.save` out of the backend working tree (flagged in the report).
- Secrets backend-side only; frontends get a URL, never a secret.
- For real public access use HTTPS via Cloudflare/ALB — not raw public ports.
- Keep `DB_SYNC=false` in normal operation; only flip to true for the one-time table creation.

---

# SCENARIO 3 — Sequelize init from Docker (no source-code change)

Use this when MySQL + Redis are in Docker AND you want Sequelize to create the 2 missing
tables, but you must NOT edit their code.

## How it works (plain explanation)
Their `models/` folder already defines all the tables. You add ONE new file,
`docker-sync.js`, that imports those existing models and runs `sequelize.sync()` once. It runs
as a one-shot Docker service that exits after creating any missing tables. Their app code is
never touched, and their backend keeps running with sync off as normal.

## Steps
1. Add `app1/backend/docker-sync.js` and `app2/backend/docker-sync.js`
   (content in the files document). Adjust the one `require("./models")` line if their
   sequelize is exported elsewhere.
2. Use the Scenario 2 compose, plus the `app1-db-init` / `app2-db-init` one-shot services and
   the `service_completed_successfully` depends_on (in the files document).
3. Run:
   ```bash
   docker compose up -d mysql redis
   # load both dumps (tables + data):
   docker compose exec -T mysql mysql -u root -p"$MYSQL_ROOT_PASSWORD" app1_db    < db/dumps/app1-dump.sql
   docker compose exec -T mysql mysql -u root -p"$MYSQL_ROOT_PASSWORD" app2_db < db/dumps/app2-dump.sql
   # bring up everything — init runs sync once, creates missing tables, backends follow:
   docker compose up -d --build
   docker compose ps
   ```
4. Confirm the 2 tables exist:
   ```bash
   docker compose exec mysql mysql -u root -p"$MYSQL_ROOT_PASSWORD" -e "USE app1_db; SHOW TABLES;"
   ```
5. The init services exit after running (`restart: "no"`) — they won't run again on reboot.

## Safe because
- `docker-sync.js` is an ADDED file that reads their existing models — not a change to their app.
- Plain `sync()` only creates missing tables; it never drops or empties existing data.
- Idempotent — running again does nothing if tables already exist.
- NEVER `force: true`.

---

# Understanding the three architectures (simple)

**Scenario 1 — Database on the host, apps in Docker.**
```
HOST: MySQL + Redis (already installed, e.g. with Apache)
DOCKER: 2 frontends + 2 backends  ── connect out to host via host.docker.internal
```
Use when the team already runs MySQL/Redis directly on the server and you don't want to move
them. The only special part is reaching the host from inside containers.

**Scenario 2 — Everything in Docker, tables from the dump only.**
```
DOCKER: MySQL + Redis + 2 frontends + 2 backends   (one self-contained stack)
Tables/data come from the two SQL dumps. Sequelize sync stays OFF.
```
Use when you want one portable stack and the dumps already contain every table.

**Scenario 3 — Everything in Docker, Sequelize creates missing tables (no code change).**
```
DOCKER: MySQL + Redis + 2 frontends + 2 backends + one-shot init (docker-sync.js)
Dumps load most tables; the init runs sequelize.sync() ONCE to add the 2 missing tables,
reading the app's existing models — without editing the app.
```
Use when the dumps are missing some tables and you must build them from the models, but you
can't modify the source.

**Common to all three:** per-folder .env, both backends reach both databases, cron runs inside
the backends (one replica each), logs persist to the host, heap raised, 24/7 self-healing,
OWASP/CIS hardened, no secrets in images.
