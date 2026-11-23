# DevTrack

**DevTrack** is a fullstack job application tracker that replaces the messy Excel sheet: companies, vacancies, interviews, resumes and a dashboard with statistics in one place.

This repository contains the **Docker Compose setup** that runs the whole stack (PostgreSQL, backend, frontend) with one command.

- **Live app:** https://devtrack-frontend-e0ru.onrender.com
- **Live API:** https://devtrack-backend-sxcc.onrender.com (Swagger: `/api`, health: `/health`)
- **Frontend repository:** https://github.com/Kirilko-frontend/devtrack-frontend
- **Backend repository:** https://github.com/Kirilko-frontend/devtrack-backend

> The live app runs on Render's free plan and sleeps after 15 minutes of inactivity. The first request after a pause can take 30-60 seconds.

## Architecture

```
Browser
  |
  |-- localhost:5173 --> [frontend: nginx, static React build]
  |
  '-- localhost:3000 --> [backend: NestJS]
                              |
                              | (Docker network, host "db")
                              v
                         [db: PostgreSQL] <--> volume: pgdata
```

| Service | Image | Description |
|---|---|---|
| `db` | `postgres:16-alpine` | PostgreSQL with a health check and a named volume for data |
| `migrate` | built from `devtrack-backend` | One-off job that runs `prisma migrate deploy` and exits |
| `backend` | built from `devtrack-backend` | NestJS API, runs as a non-root user, has a health check, uploads stored in a volume |
| `frontend` | built from `devtrack-frontend` | React (Vite) build served by nginx |

Start order: `db` becomes healthy, then `migrate` completes successfully, then `backend` starts, then `frontend`.

## Tech stack

- **Frontend:** React, TypeScript, Vite, Recharts
- **Backend:** NestJS, TypeScript, Prisma 7, JWT in an httpOnly cookie
- **Database:** PostgreSQL 16
- **Infrastructure:** Docker, Docker Compose (multi-stage builds, health checks, volumes), nginx
- **Deployment:** Render

## Quick start

**Prerequisites:** [Docker Desktop](https://www.docker.com/products/docker-desktop/), running.

```bash
git clone https://github.com/Kirilko-frontend/devtrack
cd devtrack
git clone https://github.com/Kirilko-frontend/devtrack-backend
git clone https://github.com/Kirilko-frontend/devtrack-frontend

cp .env.example .env
```

Open `.env` and fill in the values:

| Variable | Description |
|---|---|
| `DB_PASSWORD` | Password for the local PostgreSQL container (use letters and digits, avoid `@`, `:`, `/`) |
| `JWT_SECRET` | Secret used to sign JWT tokens |
| `JWT_EXPIRES_IN` | Token lifetime, e.g. `1d` |

Generate a secret:

```bash
node -e "console.log(require('crypto').randomBytes(48).toString('hex'))"
```

Start everything:

```bash
docker compose up --build
```

Then open:

- App: http://localhost:5173
- API: http://localhost:3000 (Swagger: http://localhost:3000/api)

Register a new user in the app and log in. The local database starts empty.

## Folder structure

After cloning, the folder should look like this:

```
devtrack/
├── docker-compose.yml        production-like stack
├── docker-compose.dev.yml    development stack with hot reload
├── .env.example
├── .env                      your local secrets (not committed)
├── devtrack-backend/         cloned backend repository
└── devtrack-frontend/        cloned frontend repository
```

The two cloned folders are listed in `.gitignore` and are not part of this repository.

## Commands

```bash
# production-like stack
docker compose up --build        # build and start
docker compose down              # stop, keep database data and uploads
docker compose down -v           # stop and delete database data and uploads
docker compose logs -f backend   # follow backend logs
docker compose ps                # container status and health

# development stack with hot reload
docker compose -f docker-compose.dev.yml up
docker compose -f docker-compose.dev.yml down
```

### Development mode

`docker-compose.dev.yml` mounts the source code of both projects into the containers:

- the backend runs `npm run start:dev` (NestJS watch mode)
- the frontend runs the Vite dev server on port 5173
- `node_modules` stay inside the containers (an anonymous volume hides the host folder)
- the dev database uses its own volume, `pgdata_dev`, so it does not mix with production-like data

On Windows, file watching goes through polling (`usePolling` in `vite.config.ts` and `TSC_WATCHFILE` for the backend), because file system events do not reach Linux containers.

## Data and volumes

| Volume | Contents |
|---|---|
| `pgdata` | PostgreSQL data of the production-like stack |
| `uploads` | Resumes uploaded through the app |
| `pgdata_dev` | PostgreSQL data of the development stack |

`docker compose down` keeps volumes. `docker compose down -v` removes them.

## Troubleshooting

**`port is already allocated` or `EADDRINUSE`**
Ports 3000 or 5173 are used by another container or a locally running dev server. Run `docker ps`, stop the other container (`docker stop <id>`) or stop `npm run start:dev` / `npm run dev`.

**`failed to connect to the docker API`**
Docker Desktop is not running. Start it and wait until the engine is ready, then retry.

**500 errors or login problems after `docker compose down -v`**
The database was wiped, but the browser still has a cookie for a user that no longer exists. Log out (or clear cookies for `localhost`) and register again.

**Login works locally but not in Docker**
The local run and the Docker stack use different databases. Register the user again in the Docker version.

**Hot reload does not work (development stack)**
Check the container logs for `hmr update` (frontend) or `File change detected` (backend). Make sure the Docker stack is the only thing running on ports 3000 and 5173.

## Deployment

The live version runs on Render:

- the **backend** is deployed as a Docker Web Service from `devtrack-backend` (health check path `/health`)
- the **database** is a managed PostgreSQL instance; migrations are applied with `npx prisma migrate deploy`
- the **frontend** is a Static Site that proxies `/api/*` to the backend

Details are in the READMEs of the [backend](https://github.com/Kirilko-frontend/devtrack-backend) and [frontend](https://github.com/Kirilko-frontend/devtrack-frontend) repositories.

## Author

**Kyrylo Kakulia**, Frontend / Fullstack Developer
https://github.com/Kirilko-frontend
https://www.linkedin.com/in/kyrylo-kakulia-3557b9371/
kkir614@gmail.com
