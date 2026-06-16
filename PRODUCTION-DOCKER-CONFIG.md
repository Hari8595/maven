# Production Docker Configuration — Allzone
## Frontend + Backend + MySQL + Redis (CIS / OWASP hardened, 24/7)

This file contains every Docker file you need for production: the four service Dockerfiles, the `docker-compose.yml`, the `.dockerignore`, and the `.env.example`. Copy each block into the matching file on your server.

**Security model (what makes this production-grade):**
- Non-root users in every app container (CIS 4.1)
- Pinned image versions, Alpine base (small attack surface)
- Multi-stage builds — no build tools or source in the final image
- `no-new-privileges`, `cap_drop: ALL`, read-only root filesystem (CIS 5.x)
- Per-container memory + PID limits (prevents one container starving the host)
- Health checks on every service (enables 24/7 self-healing)
- `restart: always` — containers and the whole stack survive crashes and reboots
- Backend + DB + Redis on an **internal-only** network; only the frontend/nginx is reachable
- MySQL bound to localhost only; Redis never exposed; password-protected
- Secrets come from `.env` (kept out of the image via `.dockerignore`) — never hardcoded
- Node heap (`--max-old-space-size`) tuned for build and runtime

---

## 1. File layout on the server

```
prod/
├── docker-compose.yml
├── .env                      ← real secrets (gitignored, chmod 600)
├── .env.example              ← template (safe to commit)
├── backend/
│   ├── Dockerfile
│   ├── .dockerignore
│   └── (your backend code: src/, package.json, ...)
├── frontend/
│   ├── Dockerfile
│   ├── .dockerignore
│   └── (your frontend code: app/, package.json, ...)
└── db/
    └── init/
        └── 00-create-databases.sql
```

---

## 2. `backend/Dockerfile`

```dockerfile
# ---- Backend: Node.js (multi-stage, non-root, hardened) ----
# Stage 1: install dependencies and build
FROM node:24-alpine AS build
WORKDIR /app

# Install only production deps deterministically
COPY package*.json ./
RUN npm ci --omit=dev && npm cache clean --force

# Copy source
COPY . .

# Stage 2: minimal runtime image
FROM node:24-alpine AS runtime
WORKDIR /app

# Security: run as the built-in non-root 'node' user
ENV NODE_ENV=production
# Runtime heap ceiling — keep modest so MySQL/Redis aren't starved on an 8 GB box
ENV NODE_OPTIONS=--max-old-space-size=512

# Copy only what's needed from build stage, owned by node
COPY --from=build --chown=node:node /app /app

# Drop to non-root
USER node

# Health check (compose also defines one; this is a backup at image level)
HEALTHCHECK --interval=30s --timeout=5s --start-period=30s --retries=3 \
  CMD node -e "require('http').get('http://127.0.0.1:5000/health',r=>process.exit(r.statusCode===200?0:1)).on('error',()=>process.exit(1))"

EXPOSE 5000

# CHANGE this to your real entry file if different (e.g. server.js, dist/index.js)
CMD ["node", "src/index.js"]
```

---

## 3. `frontend/Dockerfile`

```dockerfile
# ---- Frontend: Next.js (multi-stage, non-root, hardened) ----
# Stage 1: build the Next.js production bundle
FROM node:24-alpine AS build
WORKDIR /app

COPY package*.json ./
RUN npm ci && npm cache clean --force

COPY . .

# Build-time heap ceiling — Next.js builds are memory-heavy.
# 1536 MB matches your TL's note; raise only if the build OOMs.
ENV NODE_OPTIONS=--max-old-space-size=1536
ENV NEXT_TELEMETRY_DISABLED=1
RUN npm run build

# Stage 2: minimal runtime
FROM node:24-alpine AS runtime
WORKDIR /app
ENV NODE_ENV=production
ENV NEXT_TELEMETRY_DISABLED=1
# Runtime heap — running Next is light; keep modest
ENV NODE_OPTIONS=--max-old-space-size=512

# Copy the build output and node_modules, owned by non-root node
COPY --from=build --chown=node:node /app/.next ./.next
COPY --from=build --chown=node:node /app/node_modules ./node_modules
COPY --from=build --chown=node:node /app/package.json ./package.json
COPY --from=build --chown=node:node /app/public ./public

USER node

HEALTHCHECK --interval=30s --timeout=5s --start-period=40s --retries=3 \
  CMD node -e "require('http').get('http://127.0.0.1:3000',r=>process.exit(r.statusCode<500?0:1)).on('error',()=>process.exit(1))"

EXPOSE 3000

CMD ["npm", "start"]
```

> Note: if your `next.config.js` uses `output: 'standalone'`, the copy lines differ slightly (copy `.next/standalone` and `.next/static`). Tell me if you use standalone and I'll adjust.

---

## 4. `.dockerignore` (place in BOTH `backend/` and `frontend/`)

```
node_modules
npm-debug.log
.next
.git
.gitignore
.env
.env.*
!.env.example
Dockerfile
.dockerignore
README.md
*.log
coverage
.vscode
```

This keeps secrets, git history, and local junk out of the image — a real CIS/OWASP point (no secrets baked into image layers).

---

## 5. `db/init/00-create-databases.sql`

```sql
-- Runs automatically on first MySQL start (empty data dir only).
-- Creates the databases; your real dump loads the tables + data afterwards.
CREATE DATABASE IF NOT EXISTS emp360_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE DATABASE IF NOT EXISTS rec360_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- App user with least privilege (no root for the app)
CREATE USER IF NOT EXISTS 'allzone_app'@'%' IDENTIFIED BY '${MYSQL_PASSWORD}';
GRANT SELECT, INSERT, UPDATE, DELETE ON emp360_db.* TO 'allzone_app'@'%';
GRANT SELECT, INSERT, UPDATE, DELETE ON rec360_db.* TO 'allzone_app'@'%';
FLUSH PRIVILEGES;
```

> Adjust the database names to match what's inside your real SQL dump (check with `head -40 your-dump.sql`).

---

## 6. `docker-compose.yml`

```yaml
# ═══════════════════════════════════════════════════════════════════
# Allzone Production Stack — CIS/OWASP hardened, 24/7 self-healing
# frontend + backend + MySQL + Redis
# ═══════════════════════════════════════════════════════════════════

# Shared hardening applied to every service
x-hardening: &hardening
  restart: always                 # 24/7: restart on crash AND on server reboot
  security_opt:
    - no-new-privileges:true      # CIS 5.25 — block privilege escalation
  logging:
    driver: json-file
    options:
      max-size: "20m"             # log rotation — prevent disk fill (CIS)
      max-file: "5"

services:
  # ---------- MySQL ----------
  mysql:
    <<: *hardening
    image: mysql:8.0.39
    cap_drop: [ALL]
    cap_add: [CHOWN, SETGID, SETUID, DAC_OVERRIDE]   # minimum MySQL needs
    command:
      - --default-authentication-plugin=caching_sha2_password
      - --skip-name-resolve
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    volumes:
      - mysql_data:/var/lib/mysql
      - ./db/init:/docker-entrypoint-initdb.d:ro
    networks: [backend-net]
    # Bind to localhost on the host ONLY (never expose MySQL publicly)
    ports:
      - "127.0.0.1:3306:3306"
    mem_limit: 1500m
    pids_limit: 300
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "127.0.0.1", "-p${MYSQL_ROOT_PASSWORD}"]
      interval: 15s
      timeout: 5s
      retries: 5
      start_period: 40s

  # ---------- Redis ----------
  redis:
    <<: *hardening
    image: redis:7.4-alpine
    cap_drop: [ALL]
    command: ["redis-server", "--appendonly", "yes", "--requirepass", "${REDIS_PASSWORD}"]
    volumes:
      - redis_data:/data
    networks: [backend-net]
    # NOT exposed to the host — only reachable inside the network
    mem_limit: 512m
    pids_limit: 100
    read_only: true
    tmpfs:
      - /tmp
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]
      interval: 15s
      timeout: 5s
      retries: 5
      start_period: 10s

  # ---------- Backend (Node.js) ----------
  backend:
    <<: *hardening
    build:
      context: ./backend
      dockerfile: Dockerfile
    image: allzone-backend:latest
    cap_drop: [ALL]
    environment:
      NODE_ENV: production
      PORT: 5000
      DB_HOST: mysql
      DB_PORT: 3306
      DB_NAME: ${MYSQL_DATABASE}
      DB_USER: ${MYSQL_USER}
      DB_PASSWORD: ${MYSQL_PASSWORD}
      REDIS_HOST: redis
      REDIS_PORT: 6379
      REDIS_PASSWORD: ${REDIS_PASSWORD}
      JWT_SECRET: ${JWT_SECRET}
      EMP_AUTH_COOKIE: ${EMP_AUTH_COOKIE}
      EMP_ALLOWED_ROUTES_COOKIE: ${EMP_ALLOWED_ROUTES_COOKIE}
    depends_on:
      mysql: { condition: service_healthy }
      redis: { condition: service_healthy }
    networks: [backend-net]
    # Bind to localhost only; the frontend reaches it inside the network
    ports:
      - "127.0.0.1:5000:5000"
    mem_limit: 1024m
    pids_limit: 200
    read_only: true
    tmpfs:
      - /tmp

  # ---------- Frontend (Next.js) ----------
  frontend:
    <<: *hardening
    build:
      context: ./frontend
      dockerfile: Dockerfile
    image: allzone-frontend:latest
    cap_drop: [ALL]
    environment:
      NODE_ENV: production
      PORT: 3000
      # Public URL only — never put secrets behind NEXT_PUBLIC_
      NEXT_PUBLIC_API_BASE_URL: ${NEXT_PUBLIC_API_BASE_URL}
    depends_on:
      - backend
    networks: [backend-net, frontend-net]
    # Bind to localhost; a reverse proxy / SSH tunnel / ALB sits in front for public access
    ports:
      - "127.0.0.1:3000:3000"
    mem_limit: 768m
    pids_limit: 150
    read_only: true
    tmpfs:
      - /tmp
      - /app/.next/cache

networks:
  backend-net:
    driver: bridge
    internal: true        # DB/Redis/backend cannot reach the internet directly
  frontend-net:
    driver: bridge

volumes:
  mysql_data:
  redis_data:
```

---

## 7. `.env.example` (copy to `.env`, fill real values, `chmod 600 .env`)

```bash
# ===== Database =====
MYSQL_ROOT_PASSWORD=change_me_root_password
MYSQL_DATABASE=emp360_db
MYSQL_USER=allzone_app
MYSQL_PASSWORD=change_me_app_password

# ===== Redis =====
REDIS_PASSWORD=change_me_redis_password

# ===== Backend secrets (server-side ONLY) =====
JWT_SECRET=change_me_long_random_jwt_secret
EMP_AUTH_COOKIE=emp_auth_token
EMP_ALLOWED_ROUTES_COOKIE=emp_allowed_routes

# ===== Frontend (public — URL only, NEVER secrets) =====
NEXT_PUBLIC_API_BASE_URL=http://localhost:5000
```

> Generate each strong value with: `openssl rand -base64 32`
> ⚠️ **Never** put a secret behind `NEXT_PUBLIC_` — those ship to the browser. JWT secret stays here, backend-side only.

---

## 8. Notes on the hardening choices

- **`restart: always`** is what gives you 24/7 — every container restarts on crash, and the whole stack comes back automatically after a server reboot. (`unless-stopped` is similar but won't restart ones you manually stopped; `always` is the stronger 24/7 choice.)
- **`internal: true`** on `backend-net` means MySQL, Redis, and the backend cannot make outbound internet connections — a strong CIS control. If your backend genuinely needs outbound (e.g. Twilio API), keep it on a non-internal network or add a NAT path; tell me and I'll adjust.
- **`read_only: true` + `tmpfs`** — the container filesystem is read-only except small temp areas. If an app needs to write files (uploads, logs to disk), remove `read_only` for that service or mount a named volume for the writable path.
- **Node heap:** build-time 1536 MB (frontend build is heavy), runtime 512 MB per app (keeps MySQL/Redis healthy on an 8 GB box). All adjustable.
- **Secrets:** this uses `.env`. For the strongest production posture, secrets come from **AWS Secrets Manager** at deploy time instead of a file on disk — that's the next hardening step in your infra plan.
