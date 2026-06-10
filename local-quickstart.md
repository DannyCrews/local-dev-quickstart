# buwp-local: The BU WordPress Local Development Manual

**buwp-local** runs a complete BU WordPress environment on your laptop using Docker. Your code lives on your local filesystem and is reflected live in the running site — edit a file, refresh your browser, see the change.

The environment includes the same backend services as production — Redis (caching), S3 (file storage), and Shibboleth (BU login). **You don't need to understand any of those to use this tool.** buwp-local takes care of them automatically.

> **New to this tool?** The key mental shift: you don't upload or install the plugin/theme you're building. You tell buwp-local where it lives on your Mac, and it mounts it directly into WordPress. The WordPress admin is where you activate and test it — just like production.

> **Validated against buwp-local 0.7.6.** Every command in this manual has been run against the actual tool. If something doesn't match what you see, first run `npm i @bostonuniversity/buwp-local` to update — several commands (`update`, `--sandbox`, `WP_ENVIRONMENT_TYPE` support) only exist in recent versions.

---

## How to use this manual

- **First time?** Read top to bottom through [Daily Startup](#daily-startup). It's a guided path — every step includes the exact command and what you should see.
- **Returning user?** Jump straight to [Common Commands](#common-commands), [Snapshots](#pull-real-site-content-from-production), or [Troubleshooting](#troubleshooting). Every error message we've seen in the field is indexed there verbatim.

**Contents:** [What's running under the hood](#whats-running-under-the-hood) · [Prerequisites](#prerequisites) · [One-Time Machine Setup](#one-time-machine-setup) · [Starting a Project](#starting-a-project) · [The init wizard](#what-the-init-wizard-asks) · [Daily Startup](#daily-startup) · [Accessing the Site](#accessing-the-site) · [Adding Code & Content](#adding-plugins-themes-and-site-content) · [WordPress Versions & Images](#changing-the-wordpress-version-or-image) · [Common Commands](#common-commands) · [Stopping](#stopping-the-environment) · [Troubleshooting](#troubleshooting) · [File Structure](#project-file-structure) · [Config Reference](#buwp-localjson-reference) · [Credentials Reference](#credentials-reference) · [Further Reading](#further-reading)

---

## What's running under the hood

You never have to touch these directly, but knowing the four containers exist makes every error message in this manual easier to place:

| Container | Image | Role |
|---|---|---|
| `wordpress` | `ghcr.io/bu-ist/bu-wp-docker-mod_shib` | Apache + PHP + WordPress core + BU plugins/themes + Shibboleth SP |
| `db` | `mariadb:latest` | The WordPress database (bound to `127.0.0.1:3306`) |
| `redis` | `redis:alpine` | Object cache, same as production (bound to `127.0.0.1:6379`) |
| `s3proxy` | `aws-sigv4-proxy` | Signs requests to BU's S3 media storage so uploads work locally |

Two Docker volumes persist between sessions: `wp_build` (WordPress core files) and `db_data` (your database). Your own code is **not** in either volume — it's mounted live from your Mac via the `mappings` in [`.buwp-local.json`](#buwp-localjson-reference). That separation is why [`update`](#changing-the-wordpress-version-or-image) can refresh WordPress without touching your work, and why media uploads survive everything (they live in S3, not in any container).

---

## Prerequisites

| Requirement | How to verify | Notes |
|---|---|---|
| **Docker Desktop** | `docker info` runs without error | [docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop). [OrbStack](https://orbstack.dev) and [Podman](https://podman.io) also work if you prefer a lighter alternative. |
| **Node.js 18+** | `node -v` prints `v18.x` or higher | [nodejs.org](https://nodejs.org) |
| **GitHub account** | — | Must be a member of the [bu-ist](https://github.com/bu-ist) org |
| **GitHub personal access token** | — | Acts as a password so Docker can download private BU images — `read:packages` scope required — [create one here](https://github.com/settings/tokens). Choose a **classic token** if you plan to use this across multiple projects — fine-grained tokens are scoped to specific repos and won't work here. |
| **BU network or VPN** | — | Required once, to copy the credentials file from the dev server |

---

## One-Time Machine Setup

Do these two steps **once per machine**. They apply to every project you work on afterward.

### Step 1 — Give Docker permission to download private BU images

BU's WordPress image is stored privately on GitHub. This command logs Docker into GitHub so it can pull it:

```bash
docker login ghcr.io
```

Enter your **GitHub username** and your **personal access token** (not your GitHub password) when prompted.

Expected output: `Login Succeeded`

> If you see `denied: denied`, your token is missing the `read:packages` scope. See [Troubleshooting](#docker-login-returns-denied-denied).

### Step 2 — Get the credentials file

The credentials file contains the database passwords, S3 keys, and authentication certificates for the local environment ([all 15 are catalogued below](#credentials-reference)). Copy it from the BU dev server. You must be on the BU network or connected to VPN (`vpn.bu.edu`):

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

(A third pattern — mapping an entire WordPress build for full IDE context — exists for advanced use; see [VOLUME_MAPPINGS.md](https://github.com/jdub233/buwp-local/blob/main/docs/VOLUME_MAPPINGS.md#pattern-c-monolithic-sandbox).)

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

No `.npmrc` or npm authentication is needed — the package installs from the public npm registry. (Only the Docker *image* requires the GitHub token, which you handled in [machine setup](#step-1--give-docker-permission-to-download-private-bu-images).)

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
npx buwp-local init
```

This starts an **interactive setup wizard** — see [What the init wizard asks](#what-the-init-wizard-asks) below for every prompt and the recommended answer. `init` auto-detects that you're in a plugin or theme directory (it looks for a `Plugin Name:` or `Theme Name:` header) and pre-selects the right project type.

When it finishes, `init` creates a `.buwp-local.json` file capturing your choices, including a mapping that mounts this repo into the right place in WordPress. You can edit that file at any time — see the [reference](#buwp-localjson-reference).

**5. Add your hostname to `/etc/hosts`:**

Your computer uses `/etc/hosts` to map local domain names to IP addresses — without this entry, your browser won't know how to find your local site. The `init` output prints the exact command. It will look like this:

```bash
echo "127.0.0.1 dcrews-my-plugin.local" | sudo tee -a /etc/hosts
```

> **Use the hostname `init` chose** — the `.local` domain name shown in the `init` output and saved in `.buwp-local.json` under `"hostname"`. The wizard's default is `<your-username>-<project-name>.local` (e.g., user `dcrews` + folder `my-plugin` → `dcrews-my-plugin.local`). It is a domain name, not a folder name, and the value here must match the config exactly.

> **Forgot this step?** No harm done — `start` checks `/etc/hosts` for you and prints the same command if the entry is missing.

**6. Start the environment:**

```bash
npx buwp-local start
```

Your plugin is now live inside a running WordPress install. Open `https://dcrews-my-plugin.local` (your hostname), accept the [SSL warning](#ssl-warning), start [shibd](#4--start-the-bu-login-service), log in (see [Create a login](#create-a-wordpress-login)), and activate your plugin from the Plugins screen.

### Pattern B — Multiple plugins or themes at once

Use this when you need several repos running in the same WordPress instance — for example, testing a plugin alongside the theme it affects, or working on two related plugins. Your plugin/theme repos stay completely untouched; a separate "base camp" directory owns the environment.

**1. Create a base camp directory:**

```bash
mkdir ~/projects/my-sandbox
cd ~/projects/my-sandbox
```

**2. Set up npm and install buwp-local:**

The base camp is not a plugin or theme repo, so it needs its own `package.json` first:

```bash
npm init -y
npm install @bostonuniversity/buwp-local --save-dev
```

**3. [First time only] Import credentials into Keychain:**

Same as [Pattern A step 3](#pattern-a--single-plugin-or-theme) — skip if already done on this machine.

```bash
npx buwp-local keychain setup --file ~/Downloads/buwp-local-credentials.json
```

**4. Initialize the sandbox:**

```bash
npx buwp-local init
```

This starts the same **interactive setup wizard** described in [What the init wizard asks](#what-the-init-wizard-asks). Choose **Sandbox** as the project type. For a sandbox, the wizard suggests `<your-username>.local` as the hostname and creates an **empty** mappings list for you to fill in.

> **Avoid `init --sandbox` outside the wizard.** If that flag runs in a non-interactive context, it generates a placeholder mapping that mounts your base camp directory itself as a plugin — which you'd then have to delete by hand. Running plain `npx buwp-local init` and picking Sandbox from the menu is the clean path.

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

The `container` path determines what kind of thing WordPress sees: `wp-content/plugins/<name>` for plugins, `wp-content/themes/<name>` for themes, `wp-content/mu-plugins/<name>` for must-use plugins.

**6. Add your hostname to `/etc/hosts`:**

Use the hostname `init` chose (printed in the output and saved in `.buwp-local.json` under `"hostname"`):

```bash
echo "127.0.0.1 dcrews.local" | sudo tee -a /etc/hosts
```

**7. Start the environment:**

```bash
npx buwp-local start
```

All your mapped repos are now live in the same WordPress instance. Edit any file locally and the change is immediately reflected in the running site.

### What the init wizard asks

`npx buwp-local init` walks you through these prompts, in this order. Press Enter to accept a default. Press Ctrl+C to cancel at any time.

| Prompt | What it means | Recommended answer |
|---|---|---|
| **Project name** | Labels your Docker containers and volumes | Accept the default (your folder name) |
| **Project type** | Plugin, MU Plugin, Theme, or Sandbox — controls what mapping `init` creates | Accept the detected type; choose **Sandbox** for a Pattern B base camp |
| **Hostname** | The local domain for your site, used in your browser and `/etc/hosts` | Accept the default (`<username>-<project>.local`, or `<username>.local` for sandboxes) — see the [`.localhost` note](#localhost-vs-local) |
| **WordPress Docker image** | Which BU WordPress build to run | Accept the default unless you need a [specific WP version](#changing-the-wordpress-version-or-image) |
| **HTTP / HTTPS / Database ports** | The ports Docker listens on (80 / 443 / 3306) | Accept the defaults unless another local project needs them |
| **Enable Redis?** | A fast cache layer that matches production | **Yes** |
| **Enable S3 proxy?** | Mirrors BU's cloud file storage so uploaded images work locally | **Yes** unless you don't need media |
| **Enable Shibboleth?** | BU's login system — also auto-creates your WordPress user on first login | **Yes** |
| **Enable Xdebug by default?** | A PHP debugger that steps through code line-by-line in your editor | **No** — you can enable it per-session with `start --xdebug` ([Xdebug guide](https://github.com/jdub233/buwp-local/blob/main/docs/XDEBUG.md)) |

<a name="localhost-vs-local"></a>
> **`.localhost` vs `.local`:** The wizard defaults to `.local` hostnames. On Macs, `.local` is reserved for mDNS/Bonjour device discovery (RFC 6762) and can cause occasional resolution slowdowns. Using `.localhost` instead (e.g., `dcrews.localhost`) avoids this, and is the direction the team is heading. Either works today — if you choose `.localhost`, also note that pulled subsites need `https://<hostname>/<site>` destinations for HTTPS to behave (see [snapshots](#method-1--cli-snapshot-pull-recommended)). You can change the hostname in `.buwp-local.json` and `/etc/hosts` at any time.

---

## Daily Startup

### 1 — Open Docker Desktop

Launch it from Applications and wait for the whale icon in the menu bar to stop animating. Docker must be fully running before you continue.

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

### 3 — Start your project

Navigate to your project directory and start:

```bash
cd ~/projects/my-plugin
npx buwp-local start
```

`start` validates your config and credentials, regenerates `docker-compose.yml` from `.buwp-local.json`, checks `/etc/hosts`, and brings up the containers:

```
[+] Running 4/4
 ✔ Container my-plugin-db-1         Started
 ✔ Container my-plugin-redis-1      Started
 ✔ Container my-plugin-wordpress-1  Started
 ✔ Container my-plugin-s3proxy-1    Started

✅ Environment started successfully!
Access your site at: https://dcrews-my-plugin.local
```

The first-ever start pulls the Docker image and initializes the database — allow a few minutes. Subsequent starts take seconds.

### 4 — Start the BU login service

BU's authentication system (Shibboleth) requires a background process (`shibd`) inside the container. **It does not start automatically — you must start it after every `start`,** or pages that require login will return errors:

```bash
npx buwp-local shell    # Opens a terminal inside the WordPress container
service shibd start
exit
```

> This is the single most common "my site is broken" cause. If you see *"Cannot connect to shibd process"* or an HTTP 500 right after starting: it's this. See [Troubleshooting](#cannot-connect-to-shibd-process-error) for the one-liner version and the fix for when shibd crashes mid-session.

---

## Accessing the Site

### SSL warning

buwp-local uses HTTPS locally to match production as closely as possible. Because the SSL certificate is self-generated rather than issued by a public authority, your browser will show a security warning. This is expected — it's safe to proceed. Click **Advanced → Proceed to [hostname] (unsafe)**. You only need to do this once per browser session.

### Create a WordPress login

BU sites use Shibboleth SSO in production, but that requires a real BU login server. Locally, you use a regular WordPress account.

> **If you answered "yes" to Shibboleth during `init`**, visiting `/wp-admin` and logging in creates a WordPress account for you automatically. You only need the second command below to grant it super admin access. Skip the `user create` line.

```bash
# Skip this line if Shibboleth was enabled during init — the account already exists:
npx buwp-local wp user create <your-username> <your-username>@bu.edu --role=administrator

# Always run this — it grants access to Network Admin screens:
npx buwp-local wp super-admin add <your-username>@bu.edu
```

> **Replace `<your-username>` in BOTH positions** — the username *and* the email. A recurring real-world failure: copying the example, customizing the email, but leaving the literal word `user` as the username. The result is a confusing *"Email address exists in database but could not authenticate"* error at login. If that happens to you, run `npx buwp-local wp user list` and check the `user_login` column — see [Troubleshooting](#email-address-exists-in-database-but-could-not-authenticate).

The `super-admin add` command grants **super admin** access — a level above regular administrator that lets you see the Network Admin panel and manage all sites in the local multisite installation. Without it, parts of the admin (including [Snapshot Management](#method-2--snapshot-management-admin-ui)) are invisible.

Then log in at `https://<your-hostname>/wp-login.php`.

---

## Adding Plugins, Themes, and Site Content

### Your code goes in via mappings — not the WordPress admin

The plugin or theme you set up in Pattern A or B is already mounted in WordPress. Go to **Plugins → Installed Plugins** (or **Appearance → Themes**) in the WordPress admin and you'll see it there, ready to activate. You don't install it through the admin — it's already wired in.

To add more code later, add entries to the `mappings` array in `.buwp-local.json` and run `npx buwp-local start` again (`start` regenerates the Docker config from the file; since 0.7.6, `update` does too).

### Pull real site content from production

A fresh local environment has only default WordPress content. To work against real content from a BU site, pull a **snapshot**.

> **What is a snapshot?** A saved copy of a site's database and media files, stored in BU's shared cloud storage. Snapshots are **pushed** from production first, then **pulled** into local environments. If the snapshot you want hasn't been pushed yet, the pull fails with a `404 / HeadObject` error — see [Troubleshooting](#snapshot-pull-fails-with-404--headobject-error).
>
> **To push a snapshot** (or see what's available): production network admins can use **[Snapshot Management on www.bu.edu](https://www.bu.edu/wp-admin/network/sites.php?page=bu-snapshot-management)**. Ask in `#wp-bu-local-dev` on Slack if you need a site pushed.

#### Method 1 — CLI snapshot pull (recommended)

Use this when you know which site you want. `--source` is the production URL of the site; `--destination` is the local URL where it will land — keep the **same subsite path** on both sides, and use your real hostname:

```bash
npx buwp-local wp site-manager snapshot-pull \
  --source=https://www.bu.edu/dos/ \
  --destination=https://dcrews-my-plugin.local/dos
```

The command runs synchronously in your terminal and prints `Success: Snapshot pulled and installed.` when done. **No background process is needed** — `watch-jobs` is only for admin-UI pulls (Method 2).

Practical notes, learned the hard way:

- **Match the path.** Pulling `bu.edu/law`? Pull it to `<hostname>/law`, not the site root.
- **Use `https://` in the destination** if you want HTTPS to work on the pulled site.
- **The destination path must not already exist** in your local install.
- **Big sites take time.** A typical site is minutes; the full root site is hours (see below). Visiting the site mid-pull can show strange results — let it finish.
- **Older snapshots may break on PHP 8 images.** Snapshots made from PHP 7-era sites can throw PHP errors on the 6.9/7.0 images. Pull onto the matching-era image, or expect to fix forward.

#### Pulling the root site (www.bu.edu itself)

The root site snapshot is large (~4.6 GB) and the root of a multisite network can't simply be deleted and replaced, so this is a special case:

- **Easiest:** pull it to a non-root path, e.g. `--destination=https://<hostname>/home`.
- **To actually overwrite your local root site**, queue it from inside the container with the explicit overwrite flag:

```bash
npx buwp-local shell
wp site-manager queue-job snapshot-pull \
  --source=https://www.bu.edu/ \
  --destination=https://<your-hostname>/ \
  --confirm-root-overwrite
wp site-manager process-jobs
```

Queueing it as a job (rather than running the pull directly) matters here: the full relink of post data takes **2–3 hours**, and a direct run dies if your shell session is interrupted. To check on a long-running job: `npx buwp-local shell`, then `ps aux | grep snapshot` — if the line is there, it's still working.

#### Method 2 — Snapshot Management admin UI

Use this when you want to browse available snapshots and pick from a list.

**Where to find it:** Log in to your local WordPress admin → **Network Admin → Sites → Snapshots** (requires [super admin](#create-a-wordpress-login)).

- **Push Snapshot** — push a copy of a local site's content to BU's shared storage, to share with other devs or save a point-in-time copy.
- **Pull Snapshot** — an accordion list of all available snapshots, organized by source hostname. Pick one to restore to a site on your local network.

Pulls queued via the UI are background jobs. Process them with one of:

```bash
# Process whatever is queued right now, once:
npx buwp-local wp site-manager process-jobs

# Or run a job-runner that polls continuously (separate terminal, Ctrl+C to stop):
npx buwp-local watch-jobs
```

> `watch-jobs` mirrors the cron/EventBridge trigger production uses. It polls every 60 seconds by default; tune with `--interval <seconds>` (minimum 10) or a `"jobWatchInterval"` entry in `.buwp-local.json`, and silence routine output with `--quiet`. For a single queued job, `process-jobs` is simpler.

### Create a blank subsite

BU WordPress is a **multisite** installation — one WordPress instance hosting many sites under one domain (e.g., `www.bu.edu/admissions`, `www.bu.edu/law`). Your local environment mirrors this. To add a fresh empty subsite:

```bash
npx buwp-local wp site create --slug=test-site --title="Test Site"
```

Access it at `https://<your-hostname>/test-site/`.

---

## Changing the WordPress Version or Image

The Docker image ships with a specific WordPress version, PHP version, and the BU plugin/theme stack. Most of the time you won't change it — but for compatibility testing, you have two levers: **swap the image** (changes WP + PHP together) or **move WordPress core within an image**.

### Available images

Set the `"image"` field in `.buwp-local.json`, then run `npx buwp-local update`:

| Image tag | WordPress | PHP |
|---|---|---|
| `ghcr.io/bu-ist/bu-wp-docker-mod_shib:arm64-latest` | Current BU production baseline | Matches production |
| `ghcr.io/bu-ist/bu-wp-docker-mod_shib:arm64-6.9-php8.3-latest` | 6.9 | 8.3 |
| `ghcr.io/bu-ist/bu-wp-docker-mod_shib:arm64-7.0-php8.3` | 7.0 | 8.3 |

(Announcements of new images land in the `#wp-bu-local-dev` Slack channel.)

### Check what's running

```bash
npx buwp-local wp core version
```

### Move WordPress core to a specific version

```bash
npx buwp-local wp core update --version=6.4
```

Your database, content, and mapped code are untouched — this only swaps WordPress core files inside the running container.

> **Downgrading?** WP-CLI refuses to move backward without force:
> ```bash
> npx buwp-local wp core update --version=5.8 --force
> ```

### Refresh from the image (`update`)

```bash
npx buwp-local update
```

This pulls the latest version of your configured image, resets WordPress core files to what the image ships, **regenerates `docker-compose.yml` from your `.buwp-local.json`** (so config edits take effect), and recreates the containers. Flags: `--all` also updates the Redis/S3/db service images; `--preserve-wpbuild` keeps your existing WordPress core files.

**What's always preserved:** your database (separate volume), your mapped code (lives on your Mac), and media uploads (stored in S3, not in any container).

> **Good practice:** Run `npx buwp-local update` periodically — it picks up security patches and BU plugin updates that ship with new image releases. Also run `npm i @bostonuniversity/buwp-local` now and then to update the CLI itself.

---

## Common Commands

All commands run from your **project directory** (where `.buwp-local.json` lives). Run any command with `--help` for its options.

| Command | What it does |
|---|---|
| `npx buwp-local init` | Interactive setup wizard — creates `.buwp-local.json` ([details](#what-the-init-wizard-asks)) |
| `npx buwp-local start` | Start all containers (regenerates Docker config from `.buwp-local.json`) |
| `npx buwp-local start --xdebug` | Start with Xdebug enabled — step-through PHP debugging on port 9003 ([setup guide](https://github.com/jdub233/buwp-local/blob/main/docs/XDEBUG.md)) |
| `npx buwp-local start --no-s3` | Start without the S3 file storage service |
| `npx buwp-local start --no-redis` | Start without the Redis cache |
| `npx buwp-local stop` | Stop containers — database and content are preserved |
| `npx buwp-local destroy` | Remove containers **and volumes** — deletes your local database; full reset (`-f` skips the confirmation) |
| `npx buwp-local update` | Pull latest image, refresh WP core, apply config changes; database preserved ([details](#refresh-from-the-image-update)) |
| `npx buwp-local logs` | View recent logs from all containers |
| `npx buwp-local logs --follow` | Stream logs in real time |
| `npx buwp-local logs --service wordpress` | Logs for one service: `wordpress`, `db`, `s3proxy`, or `redis` |
| `npx buwp-local shell` | Open a bash shell inside the WordPress container |
| `npx buwp-local wp <command>` | Run any [WP-CLI](https://developer.wordpress.org/cli/commands/) command inside the container |
| `npx buwp-local wp plugin list` | List all plugins and their status |
| `npx buwp-local wp plugin activate <slug>` | Activate a plugin |
| `npx buwp-local wp cache flush` | Flush the Redis object cache |
| `npx buwp-local wp db export backup.sql` | Export your local database |
| `npx buwp-local wp site-manager process-jobs` | Process queued snapshot jobs once |
| `npx buwp-local watch-jobs` | Continuously process admin-UI snapshot jobs ([details](#method-2--snapshot-management-admin-ui)) |
| `npx buwp-local config --validate` | Check `.buwp-local.json` for errors (including mapping paths that don't exist) |
| `npx buwp-local config --show` | Print the fully resolved config, secrets masked |
| `npx buwp-local keychain status` | Verify Keychain access and credential inventory |
| `npx buwp-local keychain list` | List which of the 15 credentials are stored (names only) |
| `npx buwp-local --version` | Print the CLI version (also `-V`) |
| `docker stop $(docker ps -q)` | Stop **all** Docker containers on your machine (port-conflict fixer) |

> **Tip:** If you use buwp-local frequently, add these aliases to your `~/.zshrc`:
> ```bash
> alias buwp='npx buwp-local'
> alias buwp-wp='npx buwp-local wp'
> ```

> **Tip:** Each project is isolated by its `projectName`, so you can keep configs for several projects — but only one can own ports 80/443/3306/6379 at a time. Give a second project [different ports](#buwp-localjson-reference) to run two at once.

---

## Stopping the Environment

```bash
npx buwp-local stop
```

Your database and any content you've created are preserved. Run `npx buwp-local start` next time to pick up where you left off (and [restart shibd](#4--start-the-bu-login-service)).

> You may see warnings like `variable is not set` during `stop` — they're cosmetic. The Docker config references credentials that are only injected during `start`.

> **Good habit:** Stop your environment before closing your laptop. Leaving containers running is the most common cause of port conflicts at next startup.

---

## Troubleshooting

Indexed by the error you actually see. If you're stuck beyond this list, ask in the `#wp-bu-local-dev` Slack channel — it's active and friendly.

### Port already allocated

```
Error: Bind for 0.0.0.0:3306 failed: port is already allocated
```

Another Docker container is already using that port. Run `docker stop $(docker ps -q)` to stop everything, then `npx buwp-local start` again. ([Why this happens](#2--clear-any-competing-containers).)

### Cannot connect to Docker daemon

```
Cannot connect to the Docker daemon at unix:///var/run/docker.sock
```

Docker Desktop isn't running. Open it from Applications and wait for the menu bar icon to stop animating before retrying.

### docker login returns "denied: denied"

Your GitHub personal access token is expired or missing the `read:packages` scope. Generate a new one at [github.com/settings/tokens](https://github.com/settings/tokens) (classic token, `read:packages` checked), and run `docker login ghcr.io` again. The same root cause produces *"Cannot access container image"* during `start`.

### "Cannot connect to shibd process" error

The BU login background process isn't running. **This happens after every `start`** — it is routine, not a sign something is broken:

```bash
npx buwp-local shell
service shibd start
exit
```

Refresh your browser.

**If shibd was working and then died mid-session** (it can crash intermittently), a stale socket file blocks the restart. Clear it first:

```bash
npx buwp-local shell
rm -f /var/run/shibboleth/shibd.sock && service shibd start
exit
```

Or as a one-liner without entering the shell:

```bash
docker exec $(docker ps --filter "name=wordpress" --format "{{.Names}}" | head -1) bash -c "rm -f /var/run/shibboleth/shibd.sock && service shibd start"
```

If the site still won't load after a clean shibd start, `npx buwp-local destroy` followed by `start` gets you a fresh environment. **Note: `destroy` deletes your local database.**

### "Email address exists in database but could not authenticate"

You created your user from the example command but only customized the email — the account's username is literally `user` (or `username`). Verify:

```bash
npx buwp-local wp user list
```

Check the `user_login` column. Log in with that username, or delete and recreate the user with the right one. ([Background](#create-a-wordpress-login).)

### Snapshot pull fails with 404 / HeadObject error

```
fatal error: An error occurred (404) when calling the HeadObject operation:
Key "database_snapshots/www.bu.edu__dos_snapshot.sql" does not exist
```

The snapshot you're requesting **hasn't been pushed yet**. Push it (or ask someone with production network access to) from [Snapshot Management on www.bu.edu](https://www.bu.edu/wp-admin/network/sites.php?page=bu-snapshot-management), then retry.

### Redis error during a long snapshot pull

```
Error establishing a Redis connection
MISCONF Redis is configured to save RDB snapshots...
```

Seen during multi-hour root-site pulls: the site UI may show this while the import is mid-flight. Check whether the job is actually still running before doing anything drastic:

```bash
npx buwp-local shell
ps aux | grep snapshot
```

If the `snapshot-pull` process is listed, the job is alive — leave it alone and let it finish. The comprehensive relink takes 2–3 hours for large sites, and browsing the site during the process can look broken even when nothing is wrong.

### `error: unknown command 'update'` (or any missing command)

Your installed CLI predates the command. Update the package:

```bash
npm i @bostonuniversity/buwp-local
```

### buwp-local: command not found

The npm package is not installed in this directory. Make sure you're in your project directory (the one with `package.json`) and run:

```bash
npm install @bostonuniversity/buwp-local --save-dev
```

### Credentials missing on start

`start` names exactly which credentials it can't find, and offers to launch Keychain setup. To inspect for yourself:

```bash
npx buwp-local keychain status   # access check
npx buwp-local keychain list     # which of the 15 credentials are stored
```

If they're missing, repeat the import:

```bash
npx buwp-local keychain setup --file ~/Downloads/buwp-local-credentials.json
```

(Need the file again? [Step 2 of machine setup](#step-2--get-the-credentials-file).)

### scp fails — connection refused or timeout

You are not on the BU network. Connect via Cisco AnyConnect at `vpn.bu.edu` and retry.

### Hostname is slow to resolve or flaky

If your hostname ends in `.local`, macOS mDNS may be interfering. Switch to `.localhost` — change `"hostname"` in `.buwp-local.json`, add the new name to `/etc/hosts`, and run `npx buwp-local start`. ([Details](#localhost-vs-local).)

### Code changes not appearing

Check the mapping actually points where you think:

```bash
npx buwp-local config --validate   # flags mapping paths that don't exist
npx buwp-local config --show       # see the resolved mappings
```

Then confirm the file inside the container: `npx buwp-local shell`, then `ls /var/www/html/wp-content/plugins/<your-plugin>`.

---

## Project File Structure

```
my-plugin/
├── .buwp-local.json        # Your project config — safe to commit to git
├── .env.local              # Per-project credential overrides — NEVER commit this
├── .buwp-local/            # Runtime files generated on start — do not commit
│   └── docker-compose.yml  # Regenerated by start/update — never edit by hand
├── package.json
├── node_modules/
└── ...                     # Your plugin, theme, or project files
```

Add `.env.local` and `.buwp-local/` to your `.gitignore`.

---

## .buwp-local.json reference

The `init` wizard generates this file. Here's a real generated example with every field:

```json
{
  "projectName": "my-plugin",
  "image": "ghcr.io/bu-ist/bu-wp-docker-mod_shib:arm64-latest",
  "hostname": "dcrews-my-plugin.local",
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
    "WP_ENVIRONMENT_TYPE": "local",
    "XDEBUG": false
  }
}
```

| Field | What it does |
|---|---|
| `projectName` | Names your Docker containers/volumes (sanitized to lowercase). Defaults to the folder name. |
| `image` | The BU WordPress image to run — see [available images](#available-images) |
| `hostname` | The local URL for your browser; must match your `/etc/hosts` entry exactly |
| `multisite` | `true` for a BU-style multisite network (leave it `true`) |
| `services` | Toggle Redis / S3 proxy / Shibboleth on or off |
| `ports` | Host ports Docker listens on — change only if [running two projects at once](#common-commands). DB and Redis bind to `127.0.0.1` only. |
| `mappings` | Connects local directories to container paths — the heart of the tool. Relative paths resolve from the project directory. `config --validate` checks they exist. |
| `env` | Environment variables passed to WordPress. `WP_ENVIRONMENT_TYPE: "local"` lets themes/plugins detect local dev via [`wp_get_environment_type()`](https://developer.wordpress.org/reference/functions/wp_get_environment_type/); set it to `"production"` to mimic prod behavior (e.g., robots.txt). |
| `jobWatchInterval` | *(optional)* Default polling seconds for [`watch-jobs`](#method-2--snapshot-management-admin-ui) |

This file is the **source of truth**: `start` and `update` both regenerate the Docker configuration from it. Edit it, run `start`, done. It contains no secrets, so it's safe to commit.

Power-user environment variables: `BUWP_CONFIG_FILE` points the CLI at an alternate config (e.g., `BUWP_CONFIG_FILE=.buwp-local.staging.json npx buwp-local start`); `BUWP_ENV_FILE` does the same for the env file.

---

## Credentials Reference

buwp-local needs 15 credentials (database passwords, Shibboleth certs, S3 keys, OLAP settings). You imported all of them in [machine setup](#step-2--get-the-credentials-file); this section is for when you need to inspect or override them. Full detail: [CREDENTIALS.md](https://github.com/jdub233/buwp-local/blob/main/docs/CREDENTIALS.md).

**Where they live:** macOS Keychain (global, encrypted, all projects). **Loading priority:** a `.env.local` file in your project overrides Keychain values — useful for testing one credential without disturbing the global set. `.env.local` must never be committed.

| Group | Keys |
|---|---|
| Database | `WORDPRESS_DB_PASSWORD`, `DB_ROOT_PASSWORD` |
| Shibboleth | `SP_ENTITY_ID`, `IDP_ENTITY_ID`, `SHIB_IDP_LOGOUT`, `SHIB_SP_KEY`, `SHIB_SP_CERT` |
| S3 | `S3_UPLOADS_BUCKET`, `S3_UPLOADS_REGION`, `S3_UPLOADS_ACCESS_KEY_ID`, `S3_UPLOADS_SECRET_ACCESS_KEY`, `S3_ACCESS_RULES_TABLE` |
| OLAP | `OLAP`, `OLAP_ACCT_NBR`, `OLAP_REGION` |

Useful subcommands: `keychain status` (access check), `keychain list` (inventory), `keychain get <KEY>` (print one value — careful in shared terminals), `keychain set <KEY> <value>` (update one), `keychain clear` (remove all 15; `--force` skips confirmation), `keychain setup --file <path>` (bulk import).

> Multiline credentials (the Shibboleth key and cert) are stored hex-encoded in Keychain. If you ever need one outside buwp-local: `security find-generic-password -s <KEY> -w | xxd -r -p`.

> Only `WORDPRESS_DB_PASSWORD` and `DB_ROOT_PASSWORD` are strictly required to start; the rest power Shibboleth login and S3 media. `start` checks for the required ones and walks you through setup if they're missing.

---

## Further Reading

All tool documentation lives in the `docs/` folder of the buwp-local repo ([github.com/jdub233/buwp-local](https://github.com/jdub233/buwp-local)):

- **[COMMANDS.md](https://github.com/jdub233/buwp-local/blob/main/docs/COMMANDS.md)** — Full reference for every command and flag
- **[VOLUME_MAPPINGS.md](https://github.com/jdub233/buwp-local/blob/main/docs/VOLUME_MAPPINGS.md)** — Detailed guide to all three development patterns, with pros/cons
- **[XDEBUG.md](https://github.com/jdub233/buwp-local/blob/main/docs/XDEBUG.md)** — Step debugging setup for VS Code, PHPStorm, and Zed, per pattern
- **[CREDENTIALS.md](https://github.com/jdub233/buwp-local/blob/main/docs/CREDENTIALS.md)** — How credential storage, priority, and overrides work
- **[GETTING_STARTED.md](https://github.com/jdub233/buwp-local/blob/main/docs/GETTING_STARTED.md)** — The repo's own quick start
- **[MIGRATION_FROM_VM.md](https://github.com/jdub233/buwp-local/blob/main/docs/MIGRATION_FROM_VM.md)** — Coming from the old VM sandbox workflow
- **[CHANGELOG.md](https://github.com/jdub233/buwp-local/blob/main/docs/CHANGELOG.md)** — What changed in each release

Questions, bug reports, and new-image announcements: the **#wp-bu-local-dev** Slack channel.
