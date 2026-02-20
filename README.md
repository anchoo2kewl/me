# me

A generic, zero-knowledge project runner framework. It provides a unified CLI to manage multiple projects across environments (local, docker, staging, uat, production) without knowing anything about the projects it manages.

All project-specific context comes from `.me.sh` adapter files that live in each project's root directory. The framework is designed to be added as a **git submodule** (`.me/`) inside each project.

## How It Works

```
┌──────────────────────────────────────────────┐
│  ./scripts/server [env] <cmd> [project]      │  Unified CLI
├──────────────────────────────────────────────┤
│  scripts/server                              │  Generic framework
│  ├── Auto-discovers projects via .me.sh      │  (this repo)
│  ├── Parses env / command / project          │
│  └── Dispatches to adapter functions         │
├──────────────────────────────────────────────┤
│  project-a/.me.sh                            │  Adapters
│  project-b/.me.sh                            │  (in each project)
│  project-c/.me.sh                            │
└──────────────────────────────────────────────┘
```

The framework scans `$PROJECTS_ROOT` for directories containing a `.me.sh` file, loads the adapter, and dispatches commands to the standard function interface. Each adapter translates the unified command into whatever the project's native build tool expects (Makefile, scripts/server, docker compose, etc.).

## Setup

### 1. Add as submodule to a project

```bash
cd ~/play/my-project
git submodule add https://github.com/anchoo2kewl/me .me
```

### 2. Create your local config

```bash
cp .me/config.local.sh.example .me/config.local.sh
```

Edit `.me/config.local.sh` with your settings:

```bash
PROJECTS_ROOT="$HOME/play"      # Where your projects live

STAGING_HOST="1.2.3.4"          # Remote server IPs
UAT_HOST="5.6.7.8"
PROD_HOST="9.10.11.12"
REMOTE_USER="ubuntu"
```

`config.local.sh` is gitignored — it never gets committed.

### 3. (Optional) Global fallback config

If you want the runner to work from any submodule location without per-project configs:

```bash
mkdir -p ~/.config/me
cp config.local.sh.example ~/.config/me/config.sh
# Edit with your settings
```

The load order is: `config.local.sh` (repo-local) → `~/.config/me/config.sh` (global fallback).

### 4. Create a `.me.sh` adapter in your project

Create a `.me.sh` file at your project root. See [Adapter Interface](#adapter-interface) below.

## CLI Syntax

```
./scripts/server [env] <command> [project] [args...]
```

- **env**: `local` (default), `docker`, `staging`, `uat`, `prod`
- **command**: `start`, `stop`, `status`, `test`, etc.
- **project**: Project name from `.me.sh` — omit to run against all discovered projects

When running from a project's `.me/` submodule, the current project is auto-detected.

### Examples

```bash
# From the me/ directory (manages all projects)
./scripts/server status                     # Local status, all projects
./scripts/server start blog                 # Start blog locally
./scripts/server docker start folioworth    # Start folioworth in Docker
./scripts/server staging health             # Health check all on staging
./scripts/server prod logs taskai           # Tail production logs for taskai
./scripts/server promote blog staging uat    # Promote blog: staging → uat

# From a project's .me/ submodule (auto-detects project)
cd ~/play/blog
.me/scripts/server start                    # Start blog locally
.me/scripts/server staging status           # Blog staging status
```

## Commands

### Local (default environment)

| Command | Description |
|---------|-------------|
| `start [project]` | Start app natively (go run, npm dev, etc.) |
| `stop [project]` | Stop local processes |
| `dev [project]` | Start with hot reload |
| `status [project]` | Show running status via port checks |
| `restart [project]` | Stop then start |
| `logs <project>` | Tail local logs |
| `test [project]` | Run all tests |
| `test-backend [project]` | Backend tests only |
| `test-frontend [project]` | Frontend tests only |
| `db-migrate [project]` | Run database migrations |
| `db-reset [project]` | Reset database |
| `db-seed [project]` | Seed data |
| `users [project]` | List users |
| `create-admin <project> <email>` | Create admin user |

### Docker

| Command | Description |
|---------|-------------|
| `docker start [project]` | `docker compose up --build` |
| `docker stop [project]` | `docker compose down` |
| `docker status [project]` | `docker compose ps` |
| `docker logs <project>` | `docker compose logs -f` |
| `docker restart [project]` | `docker compose restart` |

### Remote (staging / uat / prod)

| Command | Description |
|---------|-------------|
| `<env> status [project]` | Service status via SSH |
| `<env> logs <project> [service]` | Tail logs via SSH |
| `<env> health [project]` | HTTP health check |
| `<env> restart [project]` | Restart services via SSH |
| `<env> users <project>` | List users |
| `<env> create-admin <project> <email>` | Create admin |

### Cross-environment

| Command | Description |
|---------|-------------|
| `promote [project] staging uat` | Merge main → uat branch |
| `promote [project] uat prod` | Create PR: uat → production branch |
| `actions [project]` | Show recent GitHub Actions runs |
| `deploy [project] [message]` | Commit + push to trigger deployment |
| `info` | Show all discovered projects with domains and ports |
| `help` | Show help |

## Adapter Interface

Each project creates a `.me.sh` file at its root. The framework sources this file and calls the standard functions.

### Required Variables

```bash
PROJECT_NAME="myapp"                    # Short name (used in CLI)
PROJECT_DOMAIN="myapp.com"              # Production domain
PROJECT_REPO="org/repo"                 # GitHub org/repo
PROJECT_STACK="Go + React + PostgreSQL" # Description
PROJECT_PORT_BACKEND=8080               # Local backend port
PROJECT_PORT_FRONTEND=3000              # Local frontend port (optional)
PROJECT_DB="postgres"                   # Database type
```

### Required Functions

**Local environment** (apps run natively, DBs in Docker):

```bash
local_start()         # Start the app
local_stop()          # Stop the app
local_dev()           # Start with hot reload
local_restart()       # Restart
local_status()        # Show status
local_logs()          # Tail logs
local_test()          # Run all tests
local_test_backend()  # Backend tests
local_test_frontend() # Frontend tests
local_db_migrate()    # Run migrations
local_db_reset()      # Reset database
local_db_seed()       # Seed data
local_users()         # List users
local_create_admin()  # Create admin ($1 = email)
```

**Docker environment** (everything in containers):

```bash
docker_start()        # docker compose up
docker_stop()         # docker compose down
docker_status()       # docker compose ps
docker_logs()         # docker compose logs
docker_restart()      # docker compose restart
```

**Remote environments** (staging/uat/prod — first arg is always the env name):

```bash
remote_status()       # $1=env — check service status
remote_logs()         # $1=env, $2+=args — tail logs
remote_health()       # $1=env — HTTP health check
remote_restart()      # $1=env — restart services
remote_users()        # $1=env — list users
remote_create_admin() # $1=env, $2=email — create admin
```

### Utilities Available to Adapters

The framework exports these utilities for use in `.me.sh` files:

| Function | Description |
|----------|-------------|
| `print_header "text"` | Blue header |
| `print_status "text"` | Green info message |
| `print_warning "text"` | Yellow warning |
| `print_error "text"` | Red error |
| `print_success "text"` | Green success |
| `port_alive <port>` | Check if port is listening (returns 0/1) |
| `port_status <port> <label>` | Print up/down status for a port |
| `kill_port <port>` | Kill process on a port |
| `require_cmd <cmd> [hint]` | Assert a command exists |
| `resolve_server <env>` | Returns `user@host` for an environment |
| `ssh_cmd <server> <command>` | Execute command on remote server |

Color variables are also exported: `$RED`, `$GREEN`, `$YELLOW`, `$BLUE`, `$PURPLE`, `$CYAN`, `$BOLD`, `$DIM`, `$NC`.

### Example Adapter

```bash
#!/usr/bin/env bash
PROJECT_NAME="myapp"
PROJECT_DOMAIN="myapp.com"
PROJECT_REPO="org/myapp"
PROJECT_STACK="Go + React + PostgreSQL"
PROJECT_PORT_BACKEND=8080
PROJECT_PORT_FRONTEND=3000
PROJECT_DB="postgres"

_dir="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
_run() { (cd "$_dir" && ./scripts/server "$@"); }

# ── Local ─────────────────────────────────────────────
local_start()         { _run start; }
local_stop()          { _run stop; }
local_dev()           { _run dev; }
local_restart()       { _run restart; }
local_status()        { port_status "$PROJECT_PORT_BACKEND" "backend"; port_status "$PROJECT_PORT_FRONTEND" "frontend"; }
local_logs()          { _run logs "$@"; }
local_test()          { _run test-all; }
local_test_backend()  { (cd "$_dir" && go test ./...); }
local_test_frontend() { (cd "$_dir/web" && npm test); }
local_db_migrate()    { _run migrate; }
local_db_reset()      { _run db-reset; }
local_db_seed()       { _run seed; }
local_users()         { _run users; }
local_create_admin()  { _run create-admin "$1"; }

# ── Docker ────────────────────────────────────────────
docker_start()   { (cd "$_dir" && docker compose up -d --build); }
docker_stop()    { (cd "$_dir" && docker compose down); }
docker_status()  { (cd "$_dir" && docker compose ps); }
docker_logs()    { (cd "$_dir" && docker compose logs -f "$@"); }
docker_restart() { (cd "$_dir" && docker compose restart); }

# ── Remote ────────────────────────────────────────────
remote_status() {
    local env="$1" server; server=$(resolve_server "$env") || return 1
    ssh_cmd "$server" "cd /home/ubuntu/myapp && docker compose ps"
}

remote_logs() {
    local env="$1"; shift; local server; server=$(resolve_server "$env") || return 1
    ssh_cmd "$server" "cd /home/ubuntu/myapp && docker compose logs --tail=100 -f ${1:-}"
}

remote_health() {
    local env="$1" domain="$PROJECT_DOMAIN"
    [[ "$env" == "staging" ]] && domain="staging.$PROJECT_DOMAIN"
    [[ "$env" == "uat" ]]     && domain="uat.$PROJECT_DOMAIN"
    if curl -sf "https://$domain/health" >/dev/null 2>&1; then
        echo -e "  ${GREEN}[healthy]${NC} $PROJECT_NAME  https://$domain"
    else
        echo -e "  ${RED}[down]${NC}    $PROJECT_NAME  https://$domain"
    fi
}

remote_restart() {
    local env="$1" server; server=$(resolve_server "$env") || return 1
    ssh_cmd "$server" "cd /home/ubuntu/myapp && docker compose restart"
}

remote_users()        { print_warning "$PROJECT_NAME: not implemented"; }
remote_create_admin() { print_warning "$PROJECT_NAME: not implemented"; }
```

## Promotion Flow

Promotion is **forward-only** and enforces the pipeline direction:

```
staging  ──→  uat  ──→  prod
 (main)      (uat)    (production)
```

Backwards promotion (e.g., `prod staging`) is rejected.

**Staging → UAT** — direct branch merge:

```bash
# From project submodule (project auto-detected)
.me/scripts/server promote staging uat

# From me/ directory (specify project)
./scripts/server promote myapp staging uat

# → git fetch origin && git checkout uat && git merge origin/main && git push origin uat
```

**UAT → Prod** — creates a GitHub PR (production requires review):

```bash
.me/scripts/server promote uat prod
# → gh pr create --base production --head uat --title "Promote uat to production"
```

Invalid examples (all rejected):

```bash
promote prod staging    # backwards
promote prod uat        # backwards
promote staging prod    # skips uat
promote local staging   # local is not a promotion source
```

## Project Discovery

The framework discovers projects by scanning `$PROJECTS_ROOT` for directories containing a `.me.sh` file. It extracts `PROJECT_NAME` from each adapter and builds a registry at startup.

When running as a submodule (e.g., `~/play/blog/.me/scripts/server`), the framework also walks up from its own location to detect the current project, so commands work without specifying a project name.

## File Structure

```
me/                              # This repo
├── scripts/
│   └── server                   # The framework (single file)
├── config.local.sh.example      # Template for local config
├── config.local.sh              # Your config (gitignored)
├── .gitignore
└── README.md

~/play/
├── blog/
│   ├── .me/                     # ← submodule pointing to this repo
│   ├── .me.sh                   # ← adapter file
│   └── ...
├── pingrly/
│   ├── .me/
│   ├── .me.sh
│   └── ...
└── me/                          # Standalone clone for multi-project management
    └── scripts/server
```

## Design Principles

- **Zero knowledge**: The framework has no project-specific text. All context comes from `.me.sh` adapters and `config.local.sh`.
- **Single file**: The entire framework is one bash script — easy to audit, easy to update via `git submodule update`.
- **Adapter pattern**: Each project translates the unified interface into its own native commands. The framework doesn't care if a project uses Make, a custom script, or raw docker commands.
- **Subshell isolation**: Each adapter is sourced and executed in a subshell, so function definitions don't leak between projects.
- **No secrets in repo**: Server IPs, credentials, and environment-specific config live in `config.local.sh` (gitignored) or `~/.config/me/config.sh`.

## License

MIT
