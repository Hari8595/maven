# Docker Files — Two-Application Stack (per-folder .env)

All the files needed to run two applications in Docker, where **each app folder has its own
`.env`**. Copy each block into the matching file on your server.

**Architecture (generic):**
- **Application 1** — frontend (Next.js) + backend (Node.js)
- **Application 2** — frontend (Next.js) + backend (Node.js)
- **One MySQL server, two databases** (`app1_db`, `app2_db`); both backends connect to both
- **One Redis** (queue + cache)
- **Four separate `.env` files** — one per app folder (each owns its own secrets)
- Hardened (CIS/OWASP): non-root, read-only, capability-dropped, health checks, 24/7
- **No secrets baked into images** — each `.env` is injected at runtime via `env_file`

---

## 1. File layout

```
stack/
├── docker-compose.yml
├── db/init/00-init.sql
├── application-1/
│   ├── backend/    (Dockerfile, .dockerignore, .env  ← own secrets, your code)
│   └── frontend/   (Dockerfile, .dockerignore, .env  ← own .env, your code)
└── application-2/
    ├── backend/    (Dockerfile, .dockerignore, .env)
    └── frontend/   (Dockerfile, .dockerignore, .env)
```

Each `.env` lives **inside its own app folder** and is loaded by that container only.

---

## 2. `application-1/backend/.env`  (App 1 backend secrets)

```bash
# ---- App 1 backend ----
NODE_ENV=production
PORT=5001
APP_NAME=application-1-backend

# Both databases (this backend reaches both)
APP1_DB_HOST=mysql
APP1_DB_PORT=3306
APP1_DB_NAME=app1_db
APP1_DB_USER=app_user
APP1_DB_PASSWORD=change_me_app_password
APP2_DB_HOST=mysql
APP2_DB_PORT=3306
APP2_DB_NAME=app2_db
APP2_DB_USER=app_user
APP2_DB_PASSWORD=change_me_app_password

# Redis
REDIS_HOST=redis
REDIS_PORT=6379
REDIS_PASSWORD=change_me_redis_password

# Auth + integrations (backend-side only)
JWT_SECRET=change_me_app1_jwt
SENDGRID_API_KEY=change_me_sendgrid
TWILIO_ACCOUNT_SID=change_me_twilio_sid
TWILIO_AUTH_TOKEN=change_me_twilio_token
```

> **Using the team's real `.env`?** Drop their file in here as-is and change **only** the
> host variables (`*_DB_HOST` / `REDIS_HOST`) from `localhost` → `mysql` / `redis`. Leave
> every other variable exactly as they wrote it. See **Section 12** for the full method.

---

## 3. `application-1/frontend/.env`  (App 1 frontend — URL only, no secrets)

```bash
# ---- App 1 frontend ----
NODE_ENV=production
PORT=3001
# Public URL of App 1 backend. Use the value the BROWSER can reach:
#  - local/tunnel: http://localhost:5001
#  - public demo:  http://<server-ip>:5001
NEXT_PUBLIC_API_BASE_URL=http://localhost:5001
```

---

## 4. `application-2/backend/.env`  (App 2 backend secrets)

```bash
# ---- App 2 backend ----
NODE_ENV=production
PORT=5002
APP_NAME=application-2-backend

APP1_DB_HOST=mysql
APP1_DB_PORT=3306
APP1_DB_NAME=app1_db
APP1_DB_USER=app_user
APP1_DB_PASSWORD=change_me_app_password
APP2_DB_HOST=mysql
APP2_DB_PORT=3306
APP2_DB_NAME=app2_db
APP2_DB_USER=app_user
APP2_DB_PASSWORD=change_me_app_password

REDIS_HOST=redis
REDIS_PORT=6379
REDIS_PASSWORD=change_me_redis_password

JWT_SECRET=change_me_app2_jwt
SENDGRID_API_KEY=change_me_sendgrid
TWILIO_ACCOUNT_SID=change_me_twilio_sid
TWILIO_AUTH_TOKEN=change_me_twilio_token
```

---

## 5. `application-2/frontend/.env`  (App 2 frontend — URL only)

```bash
# ---- App 2 frontend ----
NODE_ENV=production
PORT=3002
NEXT_PUBLIC_API_BASE_URL=http://localhost:5002
```

---

## 6. `db/init/00-init.sql`  (creates both databases + shared app user)

```sql
-- Runs on FIRST MySQL start only (empty data dir).
-- Creates both databases. The app user is created by the MySQL image
-- (via env in compose) and granted access to the second DB here.
CREATE DATABASE IF NOT EXISTS app1_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE DATABASE IF NOT EXISTS app2_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- The MySQL image creates app_user with rights to the first DB (MYSQL_DATABASE).
-- Grant the same user access to the SECOND DB too:
GRANT SELECT, INSERT, UPDATE, DELETE ON app2_db.* TO 'app_user'@'%';
FLUSH PRIVILEGES;

-- Your real tables + data come from your SQL dump, loaded after startup.
```

> **This file already creates both empty databases automatically on first start.** You do
> **not** create the databases by hand — you just feed your data/dump into them afterward.
> See **Section 13** for exactly how.

---

## 7. `application-1/backend/Dockerfile`  (Node.js, non-root, no secrets)

```dockerfile
# ---- Backend: Node.js (multi-stage, non-root, hardened, NO secrets baked) ----
FROM node:22-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm install --omit=dev && npm cache clean --force
COPY . .

FROM node:22-alpine AS runtime
WORKDIR /app
ENV NODE_ENV=production
ENV NODE_OPTIONS=--max-old-space-size=512
ENV TZ=Asia/Kolkata
RUN apk add --no-cache tzdata
COPY --from=build --chown=node:node /app /app
USER node
HEALTHCHECK --interval=30s --timeout=5s --start-period=25s --retries=3 \
  CMD node -e "require('http').get('http://127.0.0.1:'+(process.env.PORT||5001)+'/health',r=>process.exit(r.statusCode===200?0:1)).on('error',()=>process.exit(1))"
# Uses the "start" script in your package.json. Your in-app cron starts with it.
CMD ["npm", "start"]
```

> Use the **same** Dockerfile for `application-2/backend/` (the PORT comes from its own `.env`).

---

## 8. `application-1/frontend/Dockerfile`  (Next.js, non-root, no secrets)

```dockerfile
# ---- Frontend: Next.js on Node 22 (multi-stage, non-root, hardened) ----
FROM node:22-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm install && npm cache clean --force
COPY . .
ENV NEXT_TELEMETRY_DISABLED=1
# NEXT_PUBLIC_API_BASE_URL is baked at build time — passed as a build arg (see compose)
ARG NEXT_PUBLIC_API_BASE_URL
ENV NEXT_PUBLIC_API_BASE_URL=$NEXT_PUBLIC_API_BASE_URL
RUN NODE_OPTIONS="--max-old-space-size=1536" npm run build

FROM node:22-alpine AS runtime
WORKDIR /app
ENV NODE_ENV=production
ENV NEXT_TELEMETRY_DISABLED=1
ENV NODE_OPTIONS=--max-old-space-size=512
ENV TZ=Asia/Kolkata
RUN apk add --no-cache tzdata
COPY --from=build --chown=node:node /app /app
USER node
HEALTHCHECK --interval=30s --timeout=5s --start-period=35s --retries=3 \
  CMD node -e "require('http').get('http://127.0.0.1:'+(process.env.PORT||3001)+'/',r=>process.exit(r.statusCode<500?0:1)).on('error',()=>process.exit(1))"
CMD ["npm", "start"]
```

> Use the **same** Dockerfile for `application-2/frontend/`.

---

## 9. `.dockerignore`  (place in ALL FOUR app folders — blocks .env from the image)

```
node_modules
npm-debug.log
.next
.git
.gitignore
.env
.env.*
Dockerfile
.dockerignore
*.log
.vscode
README.md
```

> Important: `.env` is ignored so it's **never copied into the image**. It is loaded at
> runtime by `env_file` in compose — secrets live only on the host, never in the image.

---

## 10. `docker-compose.yml`  (one stack; each container reads its OWN .env)

```yaml
# ═══════════════════════════════════════════════════════════════════════
# Two-Application Stack — each app folder has its OWN .env (env_file).
# One MySQL (two databases) + one Redis. Hardened, 24/7, no secrets in images.
# ═══════════════════════════════════════════════════════════════════════

x-hardening: &hardening
  restart: always
  security_opt: [no-new-privileges:true]
  logging:
    driver: json-file
    options: { max-size: "20m", max-file: "5" }

services:
  mysql:
    <<: *hardening
    image: mysql:8.0.40
    cap_drop: [ALL]
    cap_add: [CHOWN, SETGID, SETUID, DAC_OVERRIDE]
    command: ["--default-authentication-plugin=caching_sha2_password", "--skip-name-resolve"]
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD:-rootpass_change_me}
      MYSQL_DATABASE: app1_db
      MYSQL_USER: app_user
      MYSQL_PASSWORD: ${MYSQL_APP_PASSWORD:-change_me_app_password}
    volumes:
      - mysql_data:/var/lib/mysql                        # ← data persists HERE (local, on the server)
      - ./db/init:/docker-entrypoint-initdb.d:ro         # ← runs init SQL on FIRST start only
    networks: [data-net]
    ports: ["127.0.0.1:3306:3306"]                        # local only, not public
    mem_limit: 1500m
    pids_limit: 300
    healthcheck:
      test: ["CMD","mysqladmin","ping","-h","127.0.0.1","-p${MYSQL_ROOT_PASSWORD:-rootpass_change_me}"]
      interval: 15s
      timeout: 5s
      retries: 5
      start_period: 40s

  redis:
    <<: *hardening
    image: redis:7.4-alpine
    cap_drop: [ALL]
    cap_add: [SETUID, SETGID, SETPCAP, CHOWN, DAC_OVERRIDE]
    command: ["redis-server","--appendonly","yes","--requirepass","${REDIS_PASSWORD:-change_me_redis_password}"]
    volumes: [redis_data:/data]
    networks: [data-net]
    mem_limit: 512m
    pids_limit: 100
    read_only: true
    tmpfs: [/tmp]
    healthcheck:
      test: ["CMD","redis-cli","-a","${REDIS_PASSWORD:-change_me_redis_password}","ping"]
      interval: 15s
      timeout: 5s
      retries: 5
      start_period: 10s

  app1-backend:
    <<: *hardening
    build: { context: ./application-1/backend }
    image: application-1-backend:latest
    cap_drop: [ALL]
    env_file: ./application-1/backend/.env      # reads its OWN .env (ALL variables in it)
    depends_on:
      mysql: { condition: service_healthy }
      redis: { condition: service_healthy }
    networks: [data-net, app-net]
    ports: ["127.0.0.1:5001:5001"]
    mem_limit: 1024m
    pids_limit: 250
    read_only: true
    tmpfs: [/tmp]

  app1-frontend:
    <<: *hardening
    build:
      context: ./application-1/frontend
      args:
        NEXT_PUBLIC_API_BASE_URL: ${APP1_PUBLIC_API_BASE_URL:-http://localhost:5001}
    image: application-1-frontend:latest
    cap_drop: [ALL]
    env_file: ./application-1/frontend/.env
    depends_on: [app1-backend]
    networks: [app-net]
    ports: ["127.0.0.1:3001:3001"]
    mem_limit: 768m
    pids_limit: 150
    read_only: true
    tmpfs: [/tmp, /app/.next/cache]

  app2-backend:
    <<: *hardening
    build: { context: ./application-2/backend }
    image: application-2-backend:latest
    cap_drop: [ALL]
    env_file: ./application-2/backend/.env
    depends_on:
      mysql: { condition: service_healthy }
      redis: { condition: service_healthy }
    networks: [data-net, app-net]
    ports: ["127.0.0.1:5002:5002"]
    mem_limit: 1024m
    pids_limit: 250
    read_only: true
    tmpfs: [/tmp]

  app2-frontend:
    <<: *hardening
    build:
      context: ./application-2/frontend
      args:
        NEXT_PUBLIC_API_BASE_URL: ${APP2_PUBLIC_API_BASE_URL:-http://localhost:5002}
    image: application-2-frontend:latest
    cap_drop: [ALL]
    env_file: ./application-2/frontend/.env
    depends_on: [app2-backend]
    networks: [app-net]
    ports: ["127.0.0.1:3002:3002"]
    mem_limit: 768m
    pids_limit: 150
    read_only: true
    tmpfs: [/tmp, /app/.next/cache]

networks:
  data-net: { driver: bridge, internal: true }   # MySQL + Redis: no internet
  app-net:  { driver: bridge }                    # backends: outbound for SendGrid/Twilio

volumes:
  mysql_data:        # ← the named volume that keeps your LOCAL database on the server
  redis_data:
```

---

## 11. Notes on the per-folder `.env` design

- Each backend/frontend container loads **only its own** `.env` through `env_file:`.
  Secrets stay scoped to that app, and `.dockerignore` keeps them out of the image.
- **MySQL root/app passwords** in the compose use a small set of top-level variables
  (`MYSQL_ROOT_PASSWORD`, `MYSQL_APP_PASSWORD`, `REDIS_PASSWORD`,
  `APP1_PUBLIC_API_BASE_URL`, `APP2_PUBLIC_API_BASE_URL`). Put these in a tiny **root**
  `.env` beside `docker-compose.yml`, OR export them in the shell before `up`. They must
  match the DB/Redis passwords inside the app `.env` files.
- **The frontend public URL is baked at build time** (Next.js requirement), passed as a
  build `arg`. To change it (local → public IP), update the build arg and rebuild only the
  frontends. See the instructions file.

---

## 12. Integrating the team's real `.env` (when it has MANY variables)

The key idea: **you do not need to understand or rewrite every variable.** `env_file:` loads
the *entire* file automatically, however many variables it has. You only change the handful
that point at a host, because inside Docker, services talk by **service name**, not
`localhost`.

### Step 1 — Use their file as-is
Put their real `.env` into the matching app folder. The compose already loads all of it:

```yaml
  app1-backend:
    env_file: ./application-1/backend/.env    # ← loads EVERY variable in their file
```

### Step 2 — Change ONLY the host variables
Out of all their variables, only the connection/host ones need editing for Docker:

| Their `.env` likely has        | Change it to        | Why                                         |
|--------------------------------|---------------------|---------------------------------------------|
| `DB_HOST=localhost` / `127.0.0.1` | `DB_HOST=mysql`  | inside Docker, MySQL is reached by its name |
| `REDIS_HOST=localhost`         | `REDIS_HOST=redis`  | same — Redis by service name                |
| `DB_PORT=3306`                 | *leave as-is*       | port is unchanged                           |

Everything else — JWT secrets, SendGrid/Twilio keys, cookie names, feature flags, any other
variables — **leave exactly as they are.** Docker passes them through untouched.

### Step 3 — Find what to change (don't read all of them)
You don't have to read every line. Just surface the host-related ones:

```bash
grep -iE "host|port|redis|mysql|database|localhost|127.0.0.1" application-1/backend/.env
```

Only the entries that point at `localhost` need changing — that's usually 2–3 lines.

### Step 4 — Keep their variable NAMES exactly
Their code reads specific names (e.g. `process.env.DB_HOST`). As long as you keep their
variable names unchanged and only fix the host *value*, their code still finds everything.
You are not renaming anything — you are reusing their file and pointing the host at Docker.

### Step 5 — Frontend `.env` — only the API URL matters
For each frontend, the one variable that matters is the public API URL the **browser** uses:

```bash
NEXT_PUBLIC_API_BASE_URL=...   # leave the rest of their frontend .env alone
```

Remember: `NEXT_PUBLIC_*` values are **baked at build time**, so changing this needs a
frontend **rebuild**, not just a restart.

### Step 6 — Apply the change
```bash
chmod 600 application-1/backend/.env          # lock it down
docker compose up -d app1-backend             # recreate so it re-reads the .env
```

---

## 13. Loading your data into the LOCAL database

Your MySQL is **local** — it runs as the `mysql:` container in this stack, and its data lives
in the `mysql_data` volume on the server. The `db/init/00-init.sql` already created both
empty databases (`app1_db`, `app2_db`) on first start, so **you do not create databases by
hand** — you just feed data into them.

### First — check what's inside your dump
```bash
head -40 your-dump.sql
```
- **Case A — dump has `CREATE DATABASE` / `USE`:** it builds the DB itself; just load it.
- **Case B — dump has only `CREATE TABLE ...`:** it loads into whatever DB you name in the
  command. Your databases already exist, so this is fine.

> ⚠️ If the dump's `CREATE DATABASE` uses a **different name** (e.g. `employment_db`) than
> `app1_db`, either rename it in the dump, or set your app's `DB_NAME` to match the dump.
> The database name in the dump and in the app `.env` must agree.

### Source A — dump file from your laptop
```bash
# copy the dump up to the server
scp -i "key.pem" app1-dump.sql ubuntu@<server-ip>:~/stack/
scp -i "key.pem" app2-dump.sql ubuntu@<server-ip>:~/stack/

# load each dump into its database (name the DB in the command)
docker compose exec -T mysql mysql -u root -p"$MYSQL_ROOT_PASSWORD" app1_db < app1-dump.sql
docker compose exec -T mysql mysql -u root -p"$MYSQL_ROOT_PASSWORD" app2_db < app2-dump.sql
```

### Source B — dump fetched from S3
```bash
# confirm the server can reach S3 (needs an IAM role or bucket access)
aws sts get-caller-identity        # returns identity = good; "Unable to locate credentials" = attach role

# download
aws s3 cp s3://your-bucket/path/app1-dump.sql ./app1-dump.sql
aws s3 cp s3://your-bucket/path/app2-dump.sql ./app2-dump.sql

# load
docker compose exec -T mysql mysql -u root -p"$MYSQL_ROOT_PASSWORD" app1_db < app1-dump.sql
docker compose exec -T mysql mysql -u root -p"$MYSQL_ROOT_PASSWORD" app2_db < app2-dump.sql
```

**Gzipped dump (`.sql.gz`)** — stream without unzipping to disk (good when disk is tight):
```bash
gunzip -c app1-dump.sql.gz | docker compose exec -T mysql mysql -u root -p"$MYSQL_ROOT_PASSWORD" app1_db
```

**Stream straight from S3 into MySQL** (never touches the server's disk):
```bash
# plain .sql
aws s3 cp s3://your-bucket/path/app1-dump.sql - | docker compose exec -T mysql mysql -u root -p"$MYSQL_ROOT_PASSWORD" app1_db
# gzipped
aws s3 cp s3://your-bucket/path/app1-dump.sql.gz - | gunzip -c | docker compose exec -T mysql mysql -u root -p"$MYSQL_ROOT_PASSWORD" app1_db
```

### Source C — app builds its own tables (migrations)
If the backend runs migrations on startup, the tables build themselves when the backend
starts. You then have nothing to load — just bring the stack up and let it migrate.

### Verify the load worked
```bash
docker compose exec mysql mysql -u root -p"$MYSQL_ROOT_PASSWORD" -e "USE app1_db; SHOW TABLES;"
docker compose exec mysql mysql -u root -p"$MYSQL_ROOT_PASSWORD" -e "USE app2_db; SHOW TABLES;"
```
You should see your tables listed for each database.

> If `$MYSQL_ROOT_PASSWORD` isn't set in your shell, type the actual password in the command
> instead: `... -p"YourActualRootPassword" app1_db < app1-dump.sql`
