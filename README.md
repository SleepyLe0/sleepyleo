# sleepyleo

Personal portfolio & hub — a monorepo containing the public-facing website and a private admin panel, orchestrated with Docker Compose.

## Structure

```
sleepyleo/
├── sleepyleo-website/   # Public portfolio site (port 3000)
├── sleepyleo-admin/     # Admin panel (port 3001)
├── docker-compose.yml          # Development (with hot-reload watch)
├── docker-compose.prod.yml     # Production
├── .env                        # Local secrets (gitignored)
└── .env.example                # Template — copy this to .env
```

## Tech Stack

| Layer | Tech |
|---|---|
| Framework | Next.js 16 + React 19 |
| Language | TypeScript |
| Styling | Tailwind CSS v4 |
| Animation | Framer Motion / Motion |
| Database | PostgreSQL 16 (via Docker) |
| ORM | Prisma 7 + `@prisma/adapter-pg` |
| Package manager | Bun |
| Container | Docker + Docker Compose |

## Setup

### 1. Clone & init submodules

```bash
git clone <repo-url>
cd sleepyleo
bun run init        # git submodule update --init --recursive
```

### 2. Configure environment

```bash
cp .env.example .env
```

Edit `.env` and fill in your values:

| Variable | Description |
|---|---|
| `DATABASE_URL` | PostgreSQL connection string |
| `POSTGRES_USER / PASSWORD / DB` | DB credentials for Docker |
| `GITHUB_USERNAME` | Your GitHub username (for project sync) |
| `GITHUB_TOKEN` | GitHub PAT — optional, increases API rate limit |
| `ADMIN_USERNAME / PASSWORD` | Admin panel login |
| `AUTH_SECRET` | Random secret for session signing |
| `OPENROUTER_API_KEY` | For the AI Intern feature |
| `VM_SSH_HOST / USER / PASSWORD` | SSH access for server health monitoring |
| `ADMIN_URL` | URL of the admin panel (default: `http://localhost:3001`) |

### 3. Generate Prisma client

```bash
cd sleepyleo-website
bunx prisma generate
cd ..
```

## Development

```bash
bun dev     # docker compose watch — starts all services with hot-reload
```

| Service | URL |
|---|---|
| Website | http://localhost:3000 |
| Admin | http://localhost:3001 |
| Database | `localhost:5432` (internal only) |

File changes in `app/`, `components/`, and `lib/` sync into the containers automatically via Docker watch → Next.js HMR picks them up without a restart.

## Database Migrations

```bash
cd sleepyleo-website
bunx prisma migrate dev     # create & apply a new migration
bunx prisma studio          # open Prisma Studio GUI
```

## Production Deploy

```bash
bun run deploy    # docker compose -f docker-compose.prod.yml up -d --build
```

## Other Commands

```bash
bun run start   # docker compose up (no watch)
bun run stop    # docker compose down
bun run update  # git submodule update --remote --merge
```

## Admin Panel

The admin panel at `localhost:3001` lets you:

- **Import projects** from your GitHub repos
- **Toggle visibility** and **featured** status per project
- **Edit** project details (description, live URL, meme GIF)
- **Delete** projects from the database
- **AI Intern** — chat interface powered by OpenRouter
- **Server Health** — SSH-based monitoring of your VM

---

> Built with ☕ by SleepyLeo
