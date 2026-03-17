# buwp-local Quickstart

**buwp-local** runs a complete BU WordPress environment on your laptop using Docker. Your code lives on your local filesystem and is reflected live in the running site.

The environment includes the same backend services as production — Redis (caching), S3 (file storage), and Shibboleth (BU login). **You don't need to understand any of those to use this tool.** buwp-local takes care of them automatically.

> **New to this tool?** The key mental shift: you don't upload or install the plugin/theme you're building. You tell buwp-local where it lives on your Mac, and it mounts it directly into WordPress. The WordPress admin is where you activate and test it — just like production.

---

## Prerequisites

| Requirement | Notes |
|---|---|
| **Docker Desktop** | [docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop) — install and make sure it can run. [OrbStack](https://orbstack.dev) and [Podman](https://podman.io) also work if you prefer a lighter alternative. |
| **Node.js 18+** | [nodejs.org](https://nodejs.org) — after installing, verify with `node -v` (should print `v18.x` or higher) |
| **GitHub account** | Must be a member of the [bu-ist](https://github.com/bu-ist) org |
| **GitHub personal access token** | Acts as a password so Docker can download private BU images — `read:packages` scope required — [create one here](https://github.com/settings/tokens). Choose a **classic token** if you plan to use this across multiple projects — fine-grained tokens are scoped to specific repos and won't work here. |
| **BU network or VPN** | Required once to copy the credentials file from the dev server |

---

## One-Time Machine Setup

Do these two steps **once per machine**. They apply to every project you work on afterward.

---

### Step 1 — Give Docker permission to download private BU images

BU's WordPress image is stored privately on GitHub. This command logs Docker into GitHub so it can pull it:

```bash
docker login ghcr.io
```

Enter your **GitHub username** and your **personal access token** (not your GitHub password) when prompted.

Expected output: `Login Succeeded`

> If you see `denied: denied`, your token is missing the `read:packages` scope. Generate a new one at [github.com/settings/tokens](https://github.com/settings/tokens), check only `read:packages`, and try again.

---

### Step 2 — Get the credentials file

The credentials file contains the database passwords, S3 keys, and authentication certificates for the local environment. Copy it from the BU dev server using this command. You must be on the BU network or connected to VPN (`vpn.bu.edu`):

```bash
scp <your-bu-username>@ist-wp-app-dv01.bu.edu:/etc/ist-apps/buwp-local-credentials.json ~/Downloads/
```

> **Replace `<your-bu-username>`** with your actual BU login (e.g., `jsmith`). Don't copy-paste the command as-is — the angle brackets are a placeholder.

> `scp` copies files between computers over a secure connection — it works just like `cp` but with a remote path. If you'd prefer a GUI, you can use [Cyberduck](https://cyberduck.io) with SFTP to connect to `ist-wp-app-dv01.bu.edu` and download the same file.

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

This starts an **interactive setup wizard** — it will ask you a series of questions in the terminal. Read each prompt before pressing Enter. Here's what it will ask and what to choose:

| Prompt | What it means | Recommended answer |
|---|---|---|
| **Hostname** | The `.local` domain name for your site | Accept the default (based on your folder name) |
| **Shibboleth** | BU's login system — enables logging in with a WordPress account locally | **Yes** |
| **S3 proxy** | Mirrors BU's cloud file storage so uploaded images work locally | **Yes** unless you don't need media |
| **Redis** | A fast cache layer that matches production | **Yes** |
| **Xdebug** | A PHP debugger that lets you step through code line-by-line in your editor | **No** unless you plan to debug PHP (you can enable it later with `--xdebug`) |

When it finishes, `init` creates a `.buwp-local.json` config file that captures your choices. You can edit this file later to change any setting.

**5. Add your hostname to `/etc/hosts`:**

Your computer uses `/etc/hosts` to map local domain names to IP addresses — without this entry, your browser won't know how to find your local site. The `init` output will print the exact command for you. It will look like this:

```bash
echo "127.0.0.1 my-plugin.local" | sudo tee -a /etc/hosts
```

> **Use the hostname that `init` chose for you** — it's the `.local` domain name shown in the `init` output and saved in `.buwp-local.json` under `"hostname"`. It's based on your project's folder name (e.g., folder `my-plugin` → hostname `my-plugin.local`). The value here must match exactly.

> **`.localhost` vs `.local`:** The `init` command defaults to `.local` hostnames (e.g., `my-plugin.local`). On some Macs, `.local` is reserved for mDNS device discovery and can cause occasional slowdowns. Using `.localhost` instead (e.g., `my-plugin.localhost`) avoids this — it's a valid alternative if you experience hostname resolution issues. You can change the hostname in `.buwp-local.json` and your `/etc/hosts` entry at any time.

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

This starts an **interactive setup wizard** in the terminal — it will ask about your hostname, which services to enable, and other options. Read each prompt before pressing Enter. See the [table in Pattern A, step 4](#pattern-a--single-plugin-or-theme) for what each question means and the recommended answers.

You can also run `npx buwp-local init` without a flag to choose the project type from the interactive prompt.

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

Use the hostname that `init` chose for you (printed in the `init` output and saved in `.buwp-local.json` under `"hostname"`):

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

Every network service on your computer communicates through numbered channels called **ports**. buwp-local uses four of them: 80 (web), 443 (secure web), 3306 (database), and 6379 (cache). If any other Docker project was left running, it may already be using one of those ports — and buwp-local won't be able to start.

Stop everything first:

```bash
docker stop $(docker ps -q)
```

This is safe to run even if nothing is running. To double-check that all four ports are free:

```bash
lsof -i :80 -i :443 -i :3306 -i :6379
```

You should see no `LISTEN` entries in the output. (`ESTABLISHED` and `CLOSED` entries are normal outbound connections from other apps and won't cause a conflict.) If you see nothing at all, you're clear.

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

### 4 — Start the BU login service

BU's authentication system (Shibboleth) requires a background process to be running inside the container. This process doesn't start automatically — you have to start it yourself every time you run `npx buwp-local start`:

```bash
npx buwp-local shell    # Opens a terminal inside the WordPress container
service shibd start
exit
```

Without this step, pages that require login will show a "Cannot connect to shibd process" error.

---

## Accessing the Site

### SSL warning

buwp-local uses HTTPS (secure web) locally to match production as closely as possible. Because the SSL certificate is self-generated rather than issued by a public authority, your browser doesn't automatically trust it and will show a security warning. This is expected — it's safe to proceed. Click **Advanced → Proceed to [hostname] (unsafe)** to continue. You only need to do this once per browser session.

---

### "Cannot connect to shibd process" error

BU's login background process hasn't started yet. This happens after every `start`. Fix it:

```bash
npx buwp-local shell    # Opens a terminal inside the WordPress container
service shibd start
exit
```

Refresh your browser. The site should now load. If it doesn't, run `npx buwp-local destroy` followed by `npx buwp-local start` to get a fresh environment. **Note: `destroy` deletes your local database.**

---

### Create a WordPress login

BU sites use Shibboleth SSO in production, but that system requires a real BU login. Locally, you'll create a regular WordPress account instead.

> **If you answered "yes" to Shibboleth during `init`**, a default admin account was already created for you. You only need the second command below to grant super admin access. Skip the `user create` line.

```bash
# Skip this line if Shibboleth was enabled during init — the account already exists:
npx buwp-local wp user create <your-username> <your-username>@bu.edu --role=administrator

# Always run this — it grants access to Network Admin screens:
npx buwp-local wp super-admin add <your-username>@bu.edu
```

> **Replace `<your-username>`** with whatever username you want (e.g., `keri`, `kayla`). This is a local-only account — it doesn't need to match your real BU login.

The `super-admin add` command grants **super admin** access — a level above regular administrator that lets you see the Network Admin panel and manage all sites in the local multisite installation. Without it, you won't be able to access certain admin screens.

Then log in at `https://my-plugin.local/wp-login.php`.

---

## Adding Plugins, Themes, and Site Content

### Your code goes in via mappings — not the WordPress admin

The plugin or theme you set up in Pattern A or B is already mounted in WordPress. Go to **Plugins → Installed Plugins** (or **Appearance → Themes**) in the WordPress admin and you'll see it there, ready to activate. You don't install it through the admin — it's already wired in.

To add additional plugins or themes, add entries to the `mappings` array in `.buwp-local.json` and restart.

---

### Pull real site content from production or staging

A fresh local environment has only default WordPress content. If you need to work against real content from a BU site, there are two ways to pull a snapshot.

> **What is a snapshot?** A snapshot is a saved copy of a site's database and media files, stored in BU's shared cloud storage. Developers push snapshots from staging or devl environments so others can restore them locally.

---

#### Method 1 — CLI snapshot pull (recommended)

Use this when you know the URL of the snapshot you want. The `--source` must match a snapshot that was previously pushed from that environment. The `--destination` is the local URL where the site will be created — use `https://` and the subsite path at the end (e.g., `/bulb`) must not already exist in your local install.

```bash
npx buwp-local wp site-manager snapshot-pull \
  --source=https://developer.bu.edu/bulb \
  --destination=https://my-plugin.local/bulb
```

The command runs synchronously and prints `Success: Snapshot pulled and installed.` when done. No background process needed.

> **What snapshots are available?** The Snapshot Management admin UI (Method 2 below) shows all available snapshots organized by source hostname. Ask a teammate maintaining the devl environment if you're unsure what's been pushed.

---

#### Method 2 — Snapshot Management admin UI

Use this when you want to browse available snapshots and pick one from a list.

**Where to find it:** Log in to your local WordPress admin → **Network Admin → Sites → Snapshots**

The page has two sections:

- **Push Snapshot** — enter a site URL to push a copy of that site's content to BU's shared storage. Use this to share a snapshot with other devs or to save a point-in-time copy.
- **Pull Snapshot** — shows an accordion list of all available snapshots, organized by the hostname they came from. Expand a hostname to see its saved snapshots, then choose one to restore to a site on your local network.

When you pull via the admin UI, the operation is queued as a background job. You need `watch-jobs` running in a separate terminal to process it:

```bash
# In a separate terminal — runs until you stop it with Ctrl+C
npx buwp-local watch-jobs

# Or to process just one queued job manually:
npx buwp-local wp site-manager process-jobs
```

> **Tip:** If you only need to process a single job, `process-jobs` is simpler than running `watch-jobs` in the background. Use `watch-jobs` when you want ongoing processing (e.g., while doing a large import that spawns multiple jobs).

---

### Create a blank subsite

BU WordPress is a **multisite** installation — one WordPress instance that hosts many separate sites under a single domain (e.g., `www.bu.edu/admissions`, `www.bu.edu/law`). Your local environment mirrors this. To add a fresh empty subsite:

```bash
npx buwp-local wp site create --slug=test-site --title="Test Site"
```

Access it at `https://my-plugin.local/test-site/`.

---

## Changing the WordPress Version

The Docker image ships with a specific version of WordPress. Most of the time you won't need to change it — but if you're testing compatibility with a particular version, here's how.

### Check what version is running

```bash
npx buwp-local wp core version
```

### Update to a specific version

```bash
npx buwp-local wp core update --version=6.4
```

Replace `6.4` with the version you need. Your database and content are preserved. This changes only the WordPress core files inside the running container — your mapped code is untouched.

### Revert to the version that ships with the image

```bash
npx buwp-local update
```

This pulls the latest Docker image and resets WordPress core back to the version baked into the image. Your **database is preserved** — content, users, and plugin settings survive. The WordPress core files are replaced.

> **Good practice:** Run `npx buwp-local update` periodically even when you're not switching versions — it also picks up security patches and BU plugin updates that ship with new image releases.

---

## Common Commands

All commands are run from your **project directory** (where `.buwp-local.json` lives).

| Command | What it does |
|---|---|
| `npx buwp-local init` | Interactive setup — creates `.buwp-local.json` |
| `npx buwp-local start` | Start all containers |
| `npx buwp-local start --xdebug` | Start with Xdebug enabled (step-through PHP debugging) |
| `npx buwp-local start --no-s3` | Start without the S3 file storage service |
| `npx buwp-local start --no-redis` | Start without the Redis cache |
| `npx buwp-local stop` | Stop containers — your database and content are preserved |
| `npx buwp-local destroy` | Remove all containers and data — full reset |
| `npx buwp-local logs` | View recent logs from all containers |
| `npx buwp-local logs --follow` | Stream logs in real time |
| `npx buwp-local shell` | Open a bash shell inside the WordPress container |
| `npx buwp-local wp <command>` | Run any WP-CLI command inside the container |
| `npx buwp-local wp core version` | Show the currently running WordPress version |
| `npx buwp-local wp core update --version=X.X` | Switch to a specific WordPress version |
| `npx buwp-local wp plugin list` | List all plugins and their status |
| `npx buwp-local wp plugin activate <slug>` | Activate a plugin |
| `npx buwp-local wp cache flush` | Flush the Redis object cache |
| `npx buwp-local watch-jobs` | Process admin UI snapshot queue — runs continuously, mirrors production cron (separate terminal) |
| `npx buwp-local wp site-manager process-jobs` | Process a single queued snapshot job |
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

Another Docker container is already using that port. Run `docker stop $(docker ps -q)` to stop everything, then `npx buwp-local start` again.

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

```bash
npx buwp-local keychain status
```

This shows which of the 15 credentials are loaded. If they're missing, repeat the keychain import:

```bash
npx buwp-local keychain setup --file ~/Downloads/buwp-local-credentials.json
```

---

### scp fails — connection refused or timeout

You are not on the BU network. Connect via Cisco AnyConnect at `vpn.bu.edu` and retry.

---

### Shibboleth crashes after it was working

The BU login background process can crash intermittently — not just on first start. You'll see the same "Cannot connect to shibd process" error even though you already started it. The fix is to clear the stale lock file before restarting:

```bash
npx buwp-local shell
rm -f /var/run/shibboleth/shibd.sock && service shibd start
exit
```

Or as a one-liner without entering the shell:

```bash
docker exec $(docker ps --filter "name=wordpress" --format "{{.Names}}" | head -1) bash -c "rm -f /var/run/shibboleth/shibd.sock && service shibd start"
```

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
    "WP_ENVIRONMENT_TYPE": "local",
    "WP_DEBUG": true,
    "XDEBUG": false
  }
}
```

| Field | What it does |
|---|---|
| `projectName` | Unique name for this project — used to label Docker containers |
| `hostname` | The local URL you'll access in your browser |
| `multisite` | `true` for a BU-style multisite network (almost always leave this as `true`) |
| `services` | Toggle individual backend services on or off |
| `ports` | The port numbers Docker listens on — only change these if another project is already using the defaults |
| `mappings` | Connects your local directories to paths inside the WordPress container |
| `env` | Environment variables passed into the WordPress container |

---

## Further Reading

All documentation lives in the `docs/` folder inside the buwp-local repo ([github.com/jdub233/buwp-local](https://github.com/jdub233/buwp-local)):

- **[COMMANDS.md](https://github.com/jdub233/buwp-local/blob/main/docs/COMMANDS.md)** — Full reference for every command and flag
- **[VOLUME_MAPPINGS.md](https://github.com/jdub233/buwp-local/blob/main/docs/VOLUME_MAPPINGS.md)** — Detailed guide to all three development patterns
- **[XDEBUG.md](https://github.com/jdub233/buwp-local/blob/main/docs/XDEBUG.md)** — Step debugging setup for VS Code, PHPStorm, and Zed
- **[CREDENTIALS.md](https://github.com/jdub233/buwp-local/blob/main/docs/CREDENTIALS.md)** — How credential storage and overrides work
- **[MIGRATION_FROM_VM.md](https://github.com/jdub233/buwp-local/blob/main/docs/MIGRATION_FROM_VM.md)** — Coming from the old VM sandbox workflow
