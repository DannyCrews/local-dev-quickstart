# buwp-local Quickstart

**buwp-local** runs a complete BU WordPress environment on your laptop — the same stack as production, including multisite, Redis, S3, and Shibboleth SSO — using Docker. Your code lives on your local filesystem and is reflected live in the running site. No more monthly VM rebuilds that wipe your work.

> **New to this tool?** The key mental shift from hosting or VM-based workflows: you don't upload or install the plugin/theme you're building. You tell buwp-local where it lives on your Mac, and it mounts it directly into WordPress. The WordPress admin is where you activate and test it — just like production.

---

## Prerequisites

| Requirement | Notes |
|---|---|
| **Docker Desktop** | [docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop) — install and make sure it can run |
| **Node.js 18+** | [nodejs.org](https://nodejs.org) |
| **GitHub account** | Must be a member of the [bu-ist](https://github.com/bu-ist) org |
| **GitHub personal access token** | `read:packages` scope required — [create one here](https://github.com/settings/tokens) |
| **BU network or VPN** | Required once to copy the credentials file from the dev server |

---

## One-Time Machine Setup

Do these two steps **once per machine**. They apply to every project you work on afterward.

---

### Step 1 — Authenticate with GitHub's container registry

buwp-local uses a private Docker image hosted on GitHub. Run this once and Docker remembers it:

```bash
docker login ghcr.io
```

Enter your **GitHub username** and your **personal access token** (not your GitHub password) when prompted.

Expected output: `Login Succeeded`

> If you see `denied: denied`, your token is missing the `read:packages` scope. Generate a new one at [github.com/settings/tokens](https://github.com/settings/tokens), check only `read:packages`, and try again.

---

### Step 2 — Get the credentials file

The credentials file contains the database passwords, S3 keys, and Shibboleth certificates for the local environment. Copy it from the BU dev server. You must be on the BU network or connected to VPN (`vpn.bu.edu`):

```bash
scp user@ist-wp-app-dv01.bu.edu:/etc/ist-apps/buwp-local-credentials.json ~/Downloads/
```

You'll import this file into macOS Keychain as part of your first project setup below. Once imported, every project picks up credentials automatically — you won't need the file again.

---

## Starting a Project

Each project gets its own directory with a `.buwp-local.json` config file. Choose the setup that fits your work:

- **Pattern A** — You're working on a single plugin or theme
- **Pattern B** — You're working on multiple plugins/themes at the same time

---

### Pattern A — Single plugin or theme

Use this when you're developing or testing one plugin or theme on its own.

**1. Get the repo you want to work on:**

```bash
git clone git@github.com:bu-ist/my-plugin.git
cd my-plugin
```

**2. Install buwp-local:**

```bash
npm install @bostonuniversity/buwp-local --save-dev
```

**3. [First time only] Import credentials into Keychain:**

If you have already done this for a previous project, skip this step. Keychain credentials are global — you only import them once per machine.

```bash
npx buwp-local keychain setup --file ~/Downloads/buwp-local-credentials.json
```

When macOS asks for Keychain access, click **Always Allow**. When asked whether to delete the source file, choose **yes**.

Expected output:
```
✅ Successfully imported 15 credential(s) into keychain
```

**4. Initialize the project:**

```bash
npx buwp-local init --plugin    # or --theme, or --mu-plugin
```

The `init` command creates `.buwp-local.json` and automatically sets up the volume mapping — it will point your repo root at the correct location inside the WordPress container (e.g., `/var/www/html/wp-content/plugins/my-plugin`).

**5. Add your hostname to `/etc/hosts`:**

The `init` output will tell you exactly what to add. It will look like this:

```bash
echo "127.0.0.1 my-plugin.local" | sudo tee -a /etc/hosts
```

**6. Start the environment:**

```bash
npx buwp-local start
```

Your plugin is now live inside a running WordPress install. Open `https://my-plugin.local`, log in (see [Create a login](#create-a-wordpress-login) below), and activate it from the Plugins screen.

---

### Pattern B — Multiple plugins or themes at once

Use this when you need several repos running in the same WordPress instance — for example, testing a plugin alongside the theme it affects, or working on two related plugins.

**1. Create a "base camp" directory:**

```bash
mkdir ~/projects/my-sandbox
cd ~/projects/my-sandbox
```

**2. Set up npm and install buwp-local:**

The sandbox directory is not a plugin or theme repo, so it needs its own `package.json` first:

```bash
npm init -y
npm install @bostonuniversity/buwp-local --save-dev
```

**3. [First time only] Import credentials into Keychain:**

If you have already done this for a previous project, skip this step.

```bash
npx buwp-local keychain setup --file ~/Downloads/buwp-local-credentials.json
```

When macOS asks for Keychain access, click **Always Allow**. When asked whether to delete the source file, choose **yes**.

**4. Initialize the sandbox:**

```bash
npx buwp-local init --sandbox
```

You can also run `npx buwp-local init` without a flag to choose the sandbox type from the interactive prompt.

**5. Edit `.buwp-local.json` to map your repos:**

Open the generated `.buwp-local.json` and add a `mappings` entry for each repo. Paths can be relative to the base camp or absolute:

```json
"mappings": [
  {
    "local": "../my-plugin",
    "container": "/var/www/html/wp-content/plugins/my-plugin"
  },
  {
    "local": "../responsive-framework",
    "container": "/var/www/html/wp-content/themes/responsive-framework"
  }
]
```

**6. Add your hostname to `/etc/hosts`:**

```bash
echo "127.0.0.1 my-sandbox.local" | sudo tee -a /etc/hosts
```

**7. Start the environment:**

```bash
npx buwp-local start
```

All your mapped repos are now live in the same WordPress instance. Edit any file locally and the change is immediately reflected in the running site.

---

## Daily Startup

### 1 — Open Docker Desktop

Launch it from Applications and wait for the whale icon in the menu bar to stop animating. Docker must be fully running before you continue.

---

### 2 — Clear any competing containers

Port conflicts are the most common startup failure. Any other Docker project that was left running — including other buwp-local projects — may be holding ports 80, 443, 3306, or 6379. Stop everything first:

```bash
docker stop $(docker ps -q)
```

This is safe to run even if nothing is running. To double-check that ports are clear:

```bash
lsof -i :80 -i :443 -i :3306 -i :6379
```

You should see no `LISTEN` entries. (`ESTABLISHED` and `CLOSED` entries are normal outbound connections from other apps and won't cause a conflict.)

---

### 3 — Start your project

Navigate to your project directory and start:

```bash
cd ~/projects/my-plugin
npx buwp-local start
```

Expected output:
```
[+] Running 4/4
 ✔ Container my-plugin-db-1         Started
 ✔ Container my-plugin-redis-1      Started
 ✔ Container my-plugin-wordpress-1  Started
 ✔ Container my-plugin-s3proxy-1    Started
✅ Environment started successfully!
Access your site at: https://my-plugin.local
```

---

### 4 — Start the Shibboleth daemon

The Shibboleth daemon does not auto-start with the container. You need to start it manually after every `start`:

```bash
npx buwp-local shell
service shibd start
exit
```

Without this step, pages that require authentication will show a "Cannot connect to shibd process" error.

---

## Accessing the Site

### SSL warning

The environment uses a self-signed certificate. Your browser will warn you about this. Click **Advanced → Proceed to [hostname] (unsafe)** to continue. You only need to do this once per browser session.

---

### Shibboleth error after starting

If you see a "Cannot connect to shibd process" error, the Shibboleth authentication daemon hasn't started yet. This happens after every `start` (not just the first time) because the shibd service does not auto-start with the container. Fix it by starting it manually:

```bash
npx buwp-local shell    # Opens a terminal inside the WordPress container
service shibd start
exit
```

Refresh your browser. The site should now load. If it doesn't, run `npx buwp-local destroy` followed by `npx buwp-local start` to get a fresh environment. **Note: `destroy` deletes your local database.**

---

### Create a WordPress login

BU sites use Shibboleth SSO in production, but locally you'll want a regular WordPress account. Create one with WP-CLI:

```bash
npx buwp-local wp user create user user@bu.edu --role=administrator
npx buwp-local wp super-admin add user@bu.edu
```

Then log in at `https://my-plugin.local/wp-login.php`.

---

## Adding Plugins, Themes, and Site Content

### Your code goes in via mappings — not the WordPress admin

The plugin or theme you set up in Pattern A or B is already mounted in WordPress. Go to **Plugins → Installed Plugins** (or **Appearance → Themes**) in the WordPress admin and you'll see it there, ready to activate. You don't install it through the admin — it's already wired in.

To add additional plugins or themes, add entries to the `mappings` array in `.buwp-local.json` and restart.

---

### Pull real site content from production or staging

A fresh local environment has only default WordPress content. If you need to work against real content from a BU site, use `snapshot-pull`:

```bash
npx buwp-local wp site-manager snapshot-pull \
  --source=https://www.bu.edu/my-site/ \
  --destination=http://my-plugin.local/my-site
```

This downloads the database and media files from the source site into your local environment. The command runs immediately — no additional processes are needed.

> **Note about `watch-jobs`:** The `watch-jobs` command is a separate tool for processing snapshot jobs queued through the **WordPress admin UI** (Network Admin → Site Manager). It replaces the production cron job that processes those queued jobs. You do not need `watch-jobs` when using `snapshot-pull` from the command line — `snapshot-pull` runs the import directly.

---

### Create a blank subsite

To add a fresh empty subsite to your local multisite network:

```bash
npx buwp-local wp site create --slug=test-site --title="Test Site"
```

Access it at `https://my-plugin.local/test-site/`.

---

## Common Commands

All commands are run from your **project directory** (where `.buwp-local.json` lives).

| Command | What it does |
|---|---|
| `npx buwp-local init` | Interactive setup — creates `.buwp-local.json` |
| `npx buwp-local start` | Start all containers |
| `npx buwp-local start --xdebug` | Start with Xdebug enabled |
| `npx buwp-local start --no-s3` | Start without the S3 proxy service |
| `npx buwp-local start --no-redis` | Start without Redis |
| `npx buwp-local stop` | Stop containers — your database and content are preserved |
| `npx buwp-local destroy` | Remove all containers and data — full reset |
| `npx buwp-local logs` | View recent logs from all containers |
| `npx buwp-local logs --follow` | Stream logs in real time |
| `npx buwp-local shell` | Open a bash shell inside the WordPress container |
| `npx buwp-local wp <command>` | Run any WP-CLI command inside the container |
| `npx buwp-local wp plugin list` | List all plugins and their status |
| `npx buwp-local wp plugin activate <slug>` | Activate a plugin |
| `npx buwp-local wp cache flush` | Flush the Redis object cache |
| `npx buwp-local watch-jobs` | Process admin UI snapshot queue jobs (run in a separate terminal) |
| `npx buwp-local update` | Pull the latest WordPress image, preserve your database |
| `npx buwp-local config --validate` | Validate your `.buwp-local.json` |
| `npx buwp-local keychain status` | Check that credentials are loaded correctly |
| `docker stop $(docker ps -q)` | Stop all Docker containers on your machine |

> **Tip:** If you use buwp-local frequently, add these aliases to your `~/.zshrc` to save typing:
> ```bash
> alias buwp='npx buwp-local'
> alias buwp-wp='npx buwp-local wp'
> ```

---

## Stopping the Environment

```bash
npx buwp-local stop
```

Your database and any content you've created are preserved. Run `npx buwp-local start` next time to pick up where you left off.

> **Good habit:** Stop your environment before closing your laptop. Leaving containers running is the most common cause of port conflicts at next startup.

---

## Troubleshooting

### Port already allocated

```
Error: Bind for 0.0.0.0:3306 failed: port is already allocated
```

Another Docker container is holding that port. Run `docker stop $(docker ps -q)` to clear everything, then `npx buwp-local start` again.

---

### Cannot connect to Docker daemon

```
Cannot connect to the Docker daemon at unix:///var/run/docker.sock
```

Docker Desktop isn't running. Open it from Applications and wait for the menu bar icon to stop animating before retrying.

---

### docker login returns "denied: denied"

Your GitHub personal access token is expired or missing the `read:packages` scope. Generate a new one at [github.com/settings/tokens](https://github.com/settings/tokens), check only `read:packages`, and run `docker login ghcr.io` again.

---

### Credentials missing on start

```
npx buwp-local keychain status
```

This shows which of the 15 credentials are loaded. If they're missing, repeat the keychain import from [Step 3](#step-3--import-credentials-into-macos-keychain).

---

### scp fails — connection refused or timeout

You are not on the BU network. Connect via Cisco AnyConnect at `vpn.bu.edu` and retry.

---

### buwp-local: command not found

The npm package is not installed in this directory. Make sure you're in your project directory (the one with `.buwp-local.json` and `package.json`) and run:

```bash
npm install @bostonuniversity/buwp-local --save-dev
```

---

## Project File Structure

```
my-plugin/
├── .buwp-local.json        # Your project config — safe to commit to git
├── .env.local              # Per-project credential overrides — NEVER commit this
├── .buwp-local/            # Runtime files generated on start — do not commit
│   └── docker-compose.yml  # Regenerated every time you run `start`
├── package.json
├── node_modules/
└── ...                     # Your plugin, theme, or project files
```

Add `.env.local` and `.buwp-local/` to your `.gitignore`.

---

## .buwp-local.json reference

The `init` command generates this file for you. Here's a full example with all fields:

```json
{
  "projectName": "my-plugin",
  "image": "ghcr.io/bu-ist/bu-wp-docker-mod_shib:arm64-latest",
  "hostname": "my-plugin.local",
  "multisite": true,
  "services": {
    "redis": true,
    "s3proxy": true,
    "shibboleth": true
  },
  "ports": {
    "http": 80,
    "https": 443,
    "db": 3306,
    "redis": 6379
  },
  "mappings": [
    {
      "local": "./",
      "container": "/var/www/html/wp-content/plugins/my-plugin"
    }
  ],
  "env": {
    "WP_DEBUG": true,
    "XDEBUG": false
  }
}
```

| Field | What it does |
|---|---|
| `projectName` | Unique identifier for this project — used for Docker container names |
| `hostname` | The local URL you'll access in your browser |
| `multisite` | `true` for a BU-style multisite network |
| `services` | Toggle Redis, S3 proxy, and Shibboleth on or off |
| `ports` | Local port bindings — change these if another project is using the defaults |
| `mappings` | Maps local directories into the container for live code sync |
| `env` | Environment variables passed into the WordPress container |

---

## Further Reading

All documentation lives in the `docs/` folder inside buwp-local:

- **[COMMANDS.md](buwp-local/docs/COMMANDS.md)** — Full reference for every command and flag
- **[VOLUME_MAPPINGS.md](buwp-local/docs/VOLUME_MAPPINGS.md)** — Detailed guide to all three development patterns
- **[XDEBUG.md](buwp-local/docs/XDEBUG.md)** — Step debugging setup for VS Code, PHPStorm, and Zed
- **[CREDENTIALS.md](buwp-local/docs/CREDENTIALS.md)** — How credential storage and overrides work
- **[MIGRATION_FROM_VM.md](buwp-local/docs/MIGRATION_FROM_VM.md)** — Coming from the old VM sandbox workflow
