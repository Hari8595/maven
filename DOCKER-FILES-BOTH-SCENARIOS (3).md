# Docker Deployment — Two Applications (Both Scenarios)

Complete, hardened Docker setup for **App 1** and **App 2**
(each Next.js frontend + Node.js 22 backend), built to **OWASP / CIS** standards.

Covers everything discussed:
- Each of the 4 folders has its **own `.env`** (path conveyed explicitly via `env_file`)
- Each backend connects to **both databases** (values overridden per app)
- Cron jobs run **inside the Node backends** (kept single-instance so they don't double-fire)
- **Logs persist** to the host (and the read-only conflict is solved)
- **Heap memory** raised (apps run hot at 91–95%)
- **CORS / allow-origin** preserved from each backend's own `.env`
- **Two SQL dumps** + the safe way to add the **2 missing tables** via Sequelize sync
- **24/7** — auto-restart on crash and on reboot

Two scenarios — pick the one that matches reality:
- **Scenario 1** — MySQL + Redis run on the **host** (apps in Docker connect out to them)
- **Scenario 2** — MySQL + Redis run **in Docker** (override DB/Redis host to service names)

---

## Folder layout (both scenarios)

```
stack/
├── docker-compose.yml                  (Scenario 1 OR 2 — see below)
├── .env                                (small shared infra values)
├── db/
│   ├── init/00-create-databases.sql    (Scenario 2 only)
│   └── dumps/
│       ├── app1-dump.sql        ← put dump 1 here
│       └── app2-dump.sql     ← put dump 2 here
├── app1/
│   ├── backend/    (Dockerfile, .dockerignore, .env, logs/, your code)
│   └── frontend/   (Dockerfile, .dockerignore, .env, logs/, your code)
└── app2/
    ├── backend/    (same shape)
    └── frontend/   (same shape)
```

---

## 1. Backend Dockerfile (`app1/backend/Dockerfile`)

Same file for both backends. Non-root, Node 22, no secrets, logs writable, heap raised.

```dockerfile
# ---- Backend: Node.js 22 (non-root, hardened, NO secrets baked in) ----
FROM node:22-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm install --omit=dev && npm cache clean --force
COPY . .

FROM node:22-alpine AS runtime
WORKDIR /app
ENV NODE_ENV=production
# Heap raised — these apps run hot (91–95%). Tune up if they still OOM.
ENV NODE_OPTIONS=--max-old-space-size=1024
ENV TZ=Asia/Kolkata
RUN apk add --no-cache tzdata

COPY --from=build --chown=node:node /app /app
# Ensure the logs folder exists and is owned by the non-root user
RUN mkdir -p /app/logs && chown -R node:node /app/logs
USER node

HEALTHCHECK --interval=30s --timeout=5s --start-period=30s --retries=3 \
  CMD node -e "require('http').get('http://127.0.0.1:'+(process.env.PORT||5001)+'/health',r=>process.exit(r.statusCode===200?0:1)).on('error',()=>process.exit(1))"

# Uses YOUR package.json "start" script. Cron jobs inside the backend start with it.
CMD ["npm", "start"]
```

> If your backend's start command is via PM2/ecosystem, set `"start": "node app.js"` in
> package.json (your entry is `app.js`). Docker replaces PM2 — the container itself is the
> process manager now (restart: always).

---

## 2. Frontend Dockerfile (`app1/frontend/Dockerfile`)

```dockerfile
# ---- Frontend: Next.js on Node 22 (non-root, hardened) ----
FROM node:22-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm install && npm cache clean --force
COPY . .
ENV NEXT_TELEMETRY_DISABLED=1
ARG NEXT_PUBLIC_API_BASE_URL
ENV NEXT_PUBLIC_API_BASE_URL=$NEXT_PUBLIC_API_BASE_URL
# Frontend build is memory-heavy — heap raised on the build command (your TL's note)
RUN NODE_OPTIONS="--max-old-space-size=1536" npm run build

FROM node:22-alpine AS runtime
WORKDIR /app
ENV NODE_ENV=production
ENV NEXT_TELEMETRY_DISABLED=1
ENV NODE_OPTIONS=--max-old-space-size=512
ENV TZ=Asia/Kolkata
RUN apk add --no-cache tzdata
COPY --from=build --chown=node:node /app /app
RUN mkdir -p /app/logs && chown -R node:node /app/logs
USER node
HEALTHCHECK --interval=30s --timeout=5s --start-period=35s --retries=3 \
  CMD node -e "require('http').get('http://127.0.0.1:'+(process.env.PORT||3001)+'/',r=>process.exit(r.statusCode<500?0:1)).on('error',()=>process.exit(1))"
CMD ["npm", "start"]
```

---

## 3. `.dockerignore` (all 4 folders — blocks .env and logs from the image)

```
node_modules
.next
.git
.gitignore
.env
.env.*
logs
*.log
Dockerfile
.dockerignore
README.md
.vscode
# do NOT ship stray secrets that were seen in the working tree:
*.gpg
.env.save
```

---

## 4. The `.env` files — explicit paths, both-DB override

### `app1/backend/.env`
```bash
NODE_ENV=production
PORT=5001
APP_NAME=app1-backend

# ===== BOTH databases (this backend reaches both) =====
# App 1 database
APP1_DB_HOST=__DB_HOST__         # Scenario 1: host.docker.internal | Scenario 2: mysql
APP1_DB_PORT=3306
APP1_DB_NAME=app1_db
APP1_DB_USER=app_user
APP1_DB_PASSWORD=__app_password__
# App 2 database
APP2_DB_HOST=__DB_HOST__
APP2_DB_PORT=3306
APP2_DB_NAME=app2_db
APP2_DB_USER=app_user
APP2_DB_PASSWORD=__app_password__

# ===== Redis =====
REDIS_HOST=__REDIS_HOST__       # Scenario 1: host.docker.internal | Scenario 2: redis
REDIS_PORT=6379
REDIS_PASSWORD=__redis_password__

# ===== Auth / integrations (backend-side only) =====
JWT_SECRET=__app1_jwt__
SENDGRID_API_KEY=__sendgrid__
TWILIO_ACCOUNT_SID=__twilio_sid__
TWILIO_AUTH_TOKEN=__twilio_token__

# ===== CORS / allow-origin (keep your real value) =====
ALLOWED_ORIGINS=http://localhost:3001

# ===== Sequelize table creation (see section 7) =====
# Set to true ONCE to create the 2 missing tables, then back to false.
DB_SYNC=false
```

> Keep YOUR real variable names. The names above (`APP1_DB_*`, `ALLOWED_ORIGINS`, `DB_SYNC`)
> are examples — match them to what the code actually reads. Only the HOST values change
> between scenarios.

### `app1/frontend/.env`
```bash
NODE_ENV=production
PORT=3001
# Public URL the BROWSER uses to reach the backend (baked at build time)
NEXT_PUBLIC_API_BASE_URL=http://localhost:5001
```

### `app2/backend/.env` — same shape, PORT=5002, APP_NAME, REC JWT
### `app2/frontend/.env` — PORT=3002, NEXT_PUBLIC_API_BASE_URL=http://localhost:5002

---

## 5. SCENARIO 1 — MySQL + Redis on the HOST

Apps in Docker; database and Redis on the host machine.

**In every backend `.env`:** `APP1_DB_HOST` / `APP2_DB_HOST` / `REDIS_HOST` = `host.docker.internal`

```yaml
# docker-compose.yml  (Scenario 1 — host MySQL + host Redis)
x-hardening: &hardening
  restart: always
  security_opt: [no-new-privileges:true]
  logging: { driver: json-file, options: { max-size: "20m", max-file: "5" } }
  extra_hosts:
    - "host.docker.internal:host-gateway"   # lets containers reach host MySQL/Redis

services:
  app1-backend:
    <<: *hardening
    build: { context: ./app1/backend }
    image: app1-backend:latest
    cap_drop: [ALL]
    env_file: ./app1/backend/.env
    volumes:
      - ./app1/backend/logs:/app/logs      # logs persist on host
    networks: [app-net]
    ports: ["127.0.0.1:5001:5001"]
    mem_limit: 1024m
    pids_limit: 250
    # NOTE: read_only is OFF because the apps write to logs/. (tmpfs alt. in notes.)

  app1-frontend:
    <<: *hardening
    build:
      context: ./app1/frontend
      args: { NEXT_PUBLIC_API_BASE_URL: "${APP1_PUBLIC_API_BASE_URL:-http://localhost:5001}" }
    image: app1-frontend:latest
    cap_drop: [ALL]
    env_file: ./app1/frontend/.env
    volumes:
      - ./app1/frontend/logs:/app/logs
    depends_on: [app1-backend]
    networks: [app-net]
    ports: ["127.0.0.1:3001:3001"]
    mem_limit: 768m
    pids_limit: 150

  app2-backend:
    <<: *hardening
    build: { context: ./app2/backend }
    image: app2-backend:latest
    cap_drop: [ALL]
    env_file: ./app2/backend/.env
    volumes:
      - ./app2/backend/logs:/app/logs
    networks: [app-net]
    ports: ["127.0.0.1:5002:5002"]
    mem_limit: 1024m
    pids_limit: 250

  app2-frontend:
    <<: *hardening
    build:
      context: ./app2/frontend
      args: { NEXT_PUBLIC_API_BASE_URL: "${APP2_PUBLIC_API_BASE_URL:-http://localhost:5002}" }
    image: app2-frontend:latest
    cap_drop: [ALL]
    env_file: ./app2/frontend/.env
    volumes:
      - ./app2/frontend/logs:/app/logs
    depends_on: [app2-backend]
    networks: [app-net]
    ports: ["127.0.0.1:3002:3002"]
    mem_limit: 768m
    pids_limit: 150

networks:
  app-net: { driver: bridge }
```

**Host MySQL must accept container connections:**
```
# /etc/mysql/mysql.conf.d/mysqld.cnf
bind-address = 0.0.0.0
```
```sql
CREATE USER 'app_user'@'%' IDENTIFIED BY 'your_password';
GRANT ALL PRIVILEGES ON app1_db.* TO 'app_user'@'%';
GRANT ALL PRIVILEGES ON app2_db.* TO 'app_user'@'%';
FLUSH PRIVILEGES;
```
Load the dumps into the **host** MySQL:
```bash
mysql -u root -p app1_db    < db/dumps/app1-dump.sql
mysql -u root -p app2_db < db/dumps/app2-dump.sql
```

---

## 6. SCENARIO 2 — MySQL + Redis in DOCKER

Everything in containers. **In every backend `.env`:** DB host = `mysql`, Redis host = `redis`.

```yaml
# docker-compose.yml  (Scenario 2 — MySQL + Redis in Docker)
x-hardening: &hardening
  restart: always
  security_opt: [no-new-privileges:true]
  logging: { driver: json-file, options: { max-size: "20m", max-file: "5" } }

services:
  mysql:
    <<: *hardening
    image: mysql:8.0.40
    cap_drop: [ALL]
    cap_add: [CHOWN, SETGID, SETUID, DAC_OVERRIDE]
    command: ["--default-authentication-plugin=caching_sha2_password","--skip-name-resolve"]
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: app1_db
      MYSQL_USER: app_user
      MYSQL_PASSWORD: ${MYSQL_APP_PASSWORD}
    volumes:
      - mysql_data:/var/lib/mysql
      - ./db/init:/docker-entrypoint-initdb.d:ro
    networks: [data-net]
    ports: ["127.0.0.1:3306:3306"]
    mem_limit: 1500m
    pids_limit: 300
    healthcheck:
      test: ["CMD","mysqladmin","ping","-h","127.0.0.1","-p${MYSQL_ROOT_PASSWORD}"]
      interval: 15s
      timeout: 5s
      retries: 5
      start_period: 40s

  redis:
    <<: *hardening
    image: redis:7.4-alpine
    cap_drop: [ALL]
    cap_add: [SETUID, SETGID, SETPCAP, CHOWN, DAC_OVERRIDE]   # Redis needs these to start
    command: ["redis-server","--appendonly","yes","--requirepass","${REDIS_PASSWORD}"]
    volumes: [redis_data:/data]
    networks: [data-net]
    mem_limit: 512m
    pids_limit: 100
    read_only: true
    tmpfs: [/tmp]
    healthcheck:
      test: ["CMD","redis-cli","-a","${REDIS_PASSWORD}","ping"]
      interval: 15s
      timeout: 5s
      retries: 5

  app1-backend:
    <<: *hardening
    build: { context: ./app1/backend }
    image: app1-backend:latest
    cap_drop: [ALL]
    env_file: ./app1/backend/.env
    volumes: [./app1/backend/logs:/app/logs]
    depends_on:
      mysql: { condition: service_healthy }
      redis: { condition: service_healthy }
    networks: [data-net, app-net]
    ports: ["127.0.0.1:5001:5001"]
    mem_limit: 1024m
    pids_limit: 250

  app1-frontend:
    <<: *hardening
    build:
      context: ./app1/frontend
      args: { NEXT_PUBLIC_API_BASE_URL: "${APP1_PUBLIC_API_BASE_URL:-http://localhost:5001}" }
    image: app1-frontend:latest
    cap_drop: [ALL]
    env_file: ./app1/frontend/.env
    volumes: [./app1/frontend/logs:/app/logs]
    depends_on: [app1-backend]
    networks: [app-net]
    ports: ["127.0.0.1:3001:3001"]
    mem_limit: 768m
    pids_limit: 150

  app2-backend:
    <<: *hardening
    build: { context: ./app2/backend }
    image: app2-backend:latest
    cap_drop: [ALL]
    env_file: ./app2/backend/.env
    volumes: [./app2/backend/logs:/app/logs]
    depends_on:
      mysql: { condition: service_healthy }
      redis: { condition: service_healthy }
    networks: [data-net, app-net]
    ports: ["127.0.0.1:5002:5002"]
    mem_limit: 1024m
    pids_limit: 250

  app2-frontend:
    <<: *hardening
    build:
      context: ./app2/frontend
      args: { NEXT_PUBLIC_API_BASE_URL: "${APP2_PUBLIC_API_BASE_URL:-http://localhost:5002}" }
    image: app2-frontend:latest
    cap_drop: [ALL]
    env_file: ./app2/frontend/.env
    volumes: [./app2/frontend/logs:/app/logs]
    depends_on: [app2-backend]
    networks: [app-net]
    ports: ["127.0.0.1:3002:3002"]
    mem_limit: 768m
    pids_limit: 150

networks:
  data-net: { driver: bridge, internal: true }   # MySQL + Redis: no internet
  app-net:  { driver: bridge }                    # backends: outbound for SendGrid/Twilio

volumes:
  mysql_data:
  redis_data:
```

**`db/init/00-create-databases.sql` (Scenario 2):**
```sql
CREATE DATABASE IF NOT EXISTS app1_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE DATABASE IF NOT EXISTS app2_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
-- app_user is created by the MySQL image (env). Grant it the SECOND database too:
GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, ALTER, INDEX, REFERENCES
  ON app2_db.* TO 'app_user'@'%';
GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, ALTER, INDEX, REFERENCES
  ON app1_db.* TO 'app_user'@'%';
FLUSH PRIVILEGES;
```
> The `CREATE, ALTER, INDEX` grants matter — Sequelize sync needs them to create the 2 missing tables.

**Load the dumps (Scenario 2, after MySQL is healthy):**
```bash
docker compose exec -T mysql mysql -u root -p"$MYSQL_ROOT_PASSWORD" app1_db    < db/dumps/app1-dump.sql
docker compose exec -T mysql mysql -u root -p"$MYSQL_ROOT_PASSWORD" app2_db < db/dumps/app2-dump.sql
```

---

## 7. The 2 missing tables — turning Sequelize sync ON safely

The dumps create most tables, but **2 tables are missing** from them. Sequelize sync (currently
OFF) can create those 2 — **safely**, because plain `sequelize.sync()` only creates tables that
**don't exist** and never drops or empties existing ones.

**The safe order:**
1. Create the databases (init SQL / Scenario 2, or manually / Scenario 1)
2. **Load both dumps** → builds all dump tables + data
3. **Turn sync ON once** → it creates ONLY the 2 missing tables, leaves the rest untouched
4. **Turn sync OFF again** (back to their normal production setting)

**How to turn it on without permanently editing their code** — if their code reads an env flag:
```js
// in their DB init (if not present, add a small guard like this):
await sequelize.sync({ alter: false });  // plain sync = create missing tables only
// guard it with an env flag so it's controllable:
if (process.env.DB_SYNC === "true") { await sequelize.sync(); }
```
Then:
```bash
# 1) set DB_SYNC=true in each backend .env, start once
docker compose up -d
# 2) confirm the 2 tables now exist
docker compose exec mysql mysql -u root -p"$MYSQL_ROOT_PASSWORD" -e "USE app1_db; SHOW TABLES;"
# 3) set DB_SYNC=false again and restart (so it doesn't sync every boot)
docker compose up -d
```

> NEVER use `sync({ force: true })` — that DROPS and recreates every table and **wipes all
> data** on each restart. Use plain `sync()` only. `alter: true` is also avoidable here since
> you only need to *add* 2 missing tables, not reshape existing ones.

---

## 8. Logs, heap, and 24/7 (your specific points)

**Logs (error.log / out.log):** every app writes to `logs/`. Each service mounts
`./<app>/<part>/logs:/app/logs`, so logs persist on the **host** and survive restarts/rebuilds.
That's why `read_only` is left OFF on the app containers (it would block log writes). If you
prefer read-only for extra hardening, replace the logs volume with `tmpfs: [/app/logs]` — but
then logs don't persist. Persisting logs is the better choice for healthcare audit needs.

**Heap (apps run at 91–95%):** runtime heap raised to 1024 MB for backends, 512 MB for
frontends, and the frontend **build** uses 1536 MB. If a service still OOMs, raise its number
and its `mem_limit` together. Also worth profiling for leaks (the report's advice).

**24/7 + survives other processes dying:** every service has `restart: always` → it restarts
on crash automatically, and on server reboot (with `sudo systemctl enable docker`). Each
container is independent — if one dies, the others keep running and the dead one self-heals.

**Cron in Node (their design):** the schedulers live inside the backends, so they start with
the backend and run continuously as designed. **Keep each backend at ONE replica** — if you
scale a backend while cron is inside it, the jobs fire once per copy (duplicate messages).

---

## 9. Hardening summary (OWASP / CIS)

- No secrets in images (`.env`, `*.gpg`, `.env.save`, `logs` all blocked by `.dockerignore`)
- Non-root, `no-new-privileges`, `cap_drop: ALL` (+minimum add-backs for MySQL/Redis)
- Per-container memory + PID limits; health checks on every service
- `restart: always` → 24/7 self-healing
- MySQL localhost-bound; Redis not publicly exposed; both password-protected
- Scenario 2: MySQL + Redis on an internal-only network (no internet)
- CORS preserved from each backend's own `.env` (`ALLOWED_ORIGINS`)
- Secrets injected at runtime via `env_file` — never baked in

---

# SCENARIO 3 — Docker MySQL + Docker Redis + Sequelize init from Docker (NO source-code change)

Same as Scenario 2 (MySQL + Redis in Docker), **plus** a one-time init step that runs
Sequelize `sync()` to create the tables — **without editing their application code**.

**The idea:** their `models/` folder already defines every table. A tiny separate init
script imports those existing models and calls `sync()` once, then exits. Their app is never
modified. The dump provides the data; Sequelize fills any missing tables from the models.

## 3.1 The init script (does NOT touch their code)

Create `app1/backend/docker-sync.js` — a NEW file beside their code (not a change to
their files). It loads their existing Sequelize setup and syncs:

```js
// docker-sync.js — one-time table initialization from their EXISTING models.
// Does not modify any of their source files. Run once, then it exits.
// Adjust the two require paths to match where their sequelize instance + models live.
(async () => {
  try {
    // Most apps export the configured sequelize from models/index.js
    const db = require("./models");                 // their existing models/index.js
    const sequelize = db.sequelize || db.default?.sequelize;
    if (!sequelize) throw new Error("Could not find sequelize export in ./models");

    await sequelize.authenticate();
    console.log("[docker-sync] connected. Running sync (create missing tables only)...");

    // Plain sync = create tables that don't exist. NEVER force:true (that wipes data).
    await sequelize.sync();                          // or { alter: true } if columns must adjust

    console.log("[docker-sync] done. Tables are in place.");
    process.exit(0);
  } catch (e) {
    console.error("[docker-sync] failed:", e.message);
    process.exit(1);
  }
})();
```

> If their sequelize instance lives elsewhere (e.g. `config/database.js`), change the
> `require("./models")` line to point there. This file is ADDED, not a modification of theirs.

## 3.2 Compose — add a one-shot init service

Use the **Scenario 2 compose**, and add this `app1-db-init` service (and a `app2-db-init` the
same way). It runs once, creates the tables, then exits — the backends start after it.

```yaml
  app1-db-init:
    image: app1-backend:latest      # reuses the backend image (has their models + deps)
    build: { context: ./app1/backend }
    env_file: ./app1/backend/.env
    command: ["node", "docker-sync.js"] # runs the init script, then exits
    depends_on:
      mysql: { condition: service_healthy }
    networks: [data-net]
    restart: "no"                        # one-shot — do NOT restart

  app2-db-init:
    image: app2-backend:latest
    build: { context: ./app2/backend }
    env_file: ./app2/backend/.env
    command: ["node", "docker-sync.js"]
    depends_on:
      mysql: { condition: service_healthy }
    networks: [data-net]
    restart: "no"
```

And make the backends wait for the init to finish:
```yaml
  app1-backend:
    depends_on:
      mysql: { condition: service_healthy }
      redis: { condition: service_healthy }
      app1-db-init: { condition: service_completed_successfully }
```

## 3.3 The order Scenario 3 runs in

1. MySQL + Redis start (Docker), databases created by init SQL
2. **You load the two dumps** (tables + data) — same commands as Scenario 2
3. `app1-db-init` / `app2-db-init` run `docker-sync.js` once → Sequelize creates any **missing**
   tables from their models (the 2 that aren't in the dumps), then exits
4. The backends start (they waited for init to finish) — sync stays OFF in their real code
5. Everything runs 24/7

```bash
# Step-by-step for Scenario 3
docker compose up -d mysql redis                # start data layer
# load dumps once MySQL is healthy:
docker compose exec -T mysql mysql -u root -p"$MYSQL_ROOT_PASSWORD" app1_db    < db/dumps/app1-dump.sql
docker compose exec -T mysql mysql -u root -p"$MYSQL_ROOT_PASSWORD" app2_db < db/dumps/app2-dump.sql
# now bring up everything — init runs, creates missing tables, backends follow:
docker compose up -d --build
docker compose ps
```

## 3.4 Why this is safe and clean

- **No source change:** `docker-sync.js` is an added file that imports their existing models;
  their app files are untouched. Their backend still runs with sync off.
- **Data safe:** plain `sync()` only creates tables that don't exist — it never drops or
  empties the tables the dumps loaded.
- **Runs once:** the init services are `restart: "no"` and exit after syncing, so they don't
  run on every boot.
- **Idempotent:** if you run it again, sync sees the tables already exist and does nothing.

> If their `models/index.js` does NOT export `sequelize`, open it to see the export name and
> adjust the one require line in `docker-sync.js`. That's the only thing to verify. You are
> reading their models, not changing them.

---

# COPY-PASTE PROMPT (for ChatGPT on another computer)

Paste this whole block if you need to rebuild or get unstuck elsewhere. It explains the
architecture with no company names.

```
Act as a DevOps expert. Explain simply, step by step — I am a junior engineer. I must NOT
modify the application source code.

ARCHITECTURE:
- Two web applications. Each has a Next.js frontend and a Node.js 22 backend (4 folders).
- Each of the 4 folders has its OWN .env file, loaded explicitly at runtime (env_file). Many
  variables each (DB, Redis, JWT, email API, WhatsApp/SMS API, CORS allow-origin).
- Two databases (one per app). BOTH backends connect to BOTH databases (one app promotes
  records into the other; shared staff use both). The DB host/credentials per app are set in
  each backend's .env and may need overriding for Docker.
- The backends use Sequelize ORM. In their code, sync is OFF (they do not auto-create tables).
- They have TWO SQL dumps (one per database) that contain CREATE TABLE + data. BUT two tables
  are missing from the dumps and must be created from the Sequelize models.
- Redis is used as a message queue. The cron/scheduled jobs are written in Node and run
  INSIDE the backends (continuously, as designed).
- The apps write logs to a logs/ folder (error.log, out.log). Heap runs hot (~91–95%), so the
  Node heap size must be raised.

I NEED three deployment scenarios, all hardened to OWASP and CIS standards, running 24/7 with
auto-restart on crash and on reboot:

SCENARIO 1 — MySQL and Redis run on the HOST machine (not Docker). The Dockerized backends
connect out to them using host.docker.internal (with extra_hosts: host.docker.internal:host-gateway).

SCENARIO 2 — MySQL and Redis run as Docker containers. Backends use DB_HOST=mysql and
REDIS_HOST=redis (service names). Provide init SQL to create both databases and a shared app
user with access to both, and the commands to load the two SQL dumps.

SCENARIO 3 — Same as Scenario 2 (MySQL + Redis in Docker), but ALSO initialize Sequelize from
Docker to create the two missing tables WITHOUT editing the source code. Do this by adding a
small separate one-time init script (docker-sync.js) that imports the app's EXISTING Sequelize
models and calls sequelize.sync() once, run as a one-shot Docker service that exits after it
finishes; the backends wait for it (depends_on: condition: service_completed_successfully).

REQUIREMENTS FOR ALL:
- Each container loads its own .env via env_file (do not rewrite their variables; only override
  the DB host and Redis host for Docker).
- No secrets baked into images: block .env, *.gpg, .env.save, and logs via .dockerignore.
- Non-root containers, no-new-privileges, drop all Linux capabilities (Redis needs SETUID,
  SETGID, SETPCAP, CHOWN, DAC_OVERRIDE added back or it fails with "setpriv: setresuid failed").
- Persist the logs/ folders to the host with a volume (so read_only must be off on app
  containers, or use tmpfs if persistence isn't needed).
- Raise Node heap: runtime ~1024 MB for backends, and build-time NODE_OPTIONS
  "--max-old-space-size=1536" on the Next.js frontend build (passed as a build arg, since
  NEXT_PUBLIC_ is baked at build time).
- Keep each backend at ONE replica (cron is inside the backend; more copies = duplicate jobs).
- Use passwords with no special characters (openssl rand -hex 24) to avoid MySQL/Redis errors.
- NEVER use sequelize.sync({force:true}) — it wipes data. Use plain sync().

Give me, one file at a time with comments and in simple language: the Dockerfiles (backend +
frontend), the .dockerignore, the four .env templates, the docker-compose.yml for each
scenario, the init SQL, the docker-sync.js init script, and clear run instructions including
how to load the two dumps and how to confirm the two missing tables were created.
```
