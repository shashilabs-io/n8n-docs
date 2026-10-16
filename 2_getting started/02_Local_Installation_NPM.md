# Installing n8n Locally with npm
## A beginner-friendly guide to local development and self-hosting options


## 1. What this lesson teaches

The previous lesson introduced n8n Cloud. This lesson introduces running n8n on your own computer and explains the trade-off: you gain more direct control, but you also take on installation and maintenance responsibilities.

We have three options:
1. **Docker image** — presented as a good option for production-style deployment, including cloud servers.
2. **NPM / Node.js installation** — demonstrated on a local developer machine for learning, testing, and custom integrations.
3. **Desktop app** — It is no longer officially supported, so it is not covered in the walkthrough.

The actual step-by-step demonstration  uses **NPM on a local machine**.

---

## 2. Cloud vs local installation: the trade-off

| Topic | n8n Cloud | Local NPM setup |
|---|---|---|
| Who manages the runtime? | The hosted service manages the platform infrastructure | You manage the local installation and environment |
| Getting started | Browser-based sign-up | Install Node.js, then install and start n8n |
| Main advantage | Simpler setup and less maintenance | More direct control for development and testing |
| Main responsibility | Work within hosted plan and service limits | Handle updates, local availability, configuration, and security |
| Best use in the lesson | Beginners who want to learn quickly | Developers who want to test or customize workflows locally |

**Important:** a local development instance is not automatically production-ready. If workflows are business-critical, you need a properly managed server setup, secure configuration, backups, monitoring, and a plan for uptime.

---

## 3. What you need before starting

 NPM method requires **Node.js**. Make sure NPM (Node Package Manager) is installed alongside Node.js.

You should have:
- permission to install software on your computer;
- a terminal or command-line application;
- internet access to download Node.js and the n8n package;
- a Node.js version supported by the current n8n documentation.


## 4. Step-by-step: install n8n using NPM

### Step 1 — Open the official n8n documentation

Go to the official n8n documentation and open its self-hosting/installation section. Choose the NPM installation instructions and follow the current guide.

Official documentation: [n8n installation overview](https://docs.n8n.io/hosting/installation/)

### Step 2 — Install Node.js

Install the Node.js from its official Site. NPM is installed with it.


### Step 3 — Verify Node.js and npm

Open Terminal on macOS/Linux or a terminal such as PowerShell on Windows, then run:

```bash
node -v
npm -v
```

These commands print the installed versions. If the terminal says a command is not recognized, confirm the installation completed and reopen the terminal.

### Step 4 — Install n8n globally

Run a global NPM installation command:

```bash
npm install -g n8n
```

The `-g` option requests a global installation, so the `n8n` command can be run from the terminal. Global installation permissions and command behaviour can differ by operating system and Node.js setup.

### Step 5 — Start n8n

After the installation completes, run:

```bash
n8n start
```

Access n8n in a browser at:

```text
http://localhost:5678
```

`localhost` means your own computer.

### Step 6 — Create the local owner account

On the first visit, You will see an initial setup form and enters a name, email, and password. The form also asks onboarding questions about the organization and intended use.

Use a strong password and keep it safe. The form and available options may look different in a newer n8n version.

### Step 7 — Optional: request and activate the free license key shown in the lesson

You can enter a free license key by email and entering it through the instance’s settings. This unlocks certain features in that version, such as workflow history and additional debugging/search features.


### Step 8 — Keep the terminal process running

For a local instance, Keep the terminal process running to run n8n. If you stop that process or shut down the computer, the local service may become unavailable. This is one reason a local learning setup differs from a managed production deployment.

---

## 5. Commands at a glance

| Command | Purpose |
|---|---|
| `node -v` | Display the installed Node.js version |
| `npm -v` | Display the installed npm version |
| `npm install -g n8n` | Install n8n globally through npm, as shown in the lesson |
| `n8n start` | Start the local n8n service, as shown in the lesson |
| `npx n8n` | The transcript mentions this as a way to try n8n without doing the global installation |

 `npx n8n` can be used to try n8n without installing it globally. Check the current official documentation for the recommended command and behaviour for your version.

---

## 6. Updating or choosing an n8n version

- **Update to the latest stable release:** See in the Example > a global npm update command.
- **Install a specific version:** a version can be appended to the package name using `@version`.
- **Try a prerelease (`next`) version:** this is a way to test newer features, with less stability assurance than a stable release.

Examples:

```bash
npm update -g n8n
```

```bash
npm install -g n8n@VERSION
```

Replace `VERSION` with the exact version you intend to install.

**Before upgrading:** read the current release notes, back up any important workflows and data, and verify the upgrade method in the official documentation.

---

## 7. Docker vs NPM: 

### Docker image

Docker is a better fit for production-style deployment, including cloud servers such as AWS, Google Cloud, Azure, or DigitalOcean. Containers help package an application and isolate it from other software on the host. 

### NPM on a developer machine

Use NPM to run n8n locally for development and testing. This can be useful when experimenting with workflows or custom integrations before moving them to a managed server environment.

### A practical learning path

```text
Learn the interface on n8n Cloud
             |
             v
Practice workflow logic
             |
             v
Try local NPM installation (optional)
             |
             v
Plan a proper server deployment if needed
```



## 8. Common beginner issues

These checks are general troubleshooting guidance:

- **`node` or `npm` is not recognized:** verify Node.js installed correctly, then reopen the terminal.
- **Global installation fails due to permissions:** check your operating system’s Node.js/npm installation guidance; avoid changing permissions blindly.
- **The browser does not open automatically:** try navigating to `http://localhost:5678` while the n8n process is running.
- **Port 5678 is already in use:** another process may be using the port. Read the terminal error and consult the current n8n documentation before changing configuration.
- **The local page stops responding:** check whether the terminal process is still running.
- **A command differs from the video:** use the current official installation documentation because commands and supported versions can change.

---

## 9. Security and production reminders

A local development installation is intended for learning and testing—not automatically for exposing workflows to the public internet.

Before running important or public-facing automations:
- keep n8n and dependencies updated;
- protect the owner account and credentials;
- avoid exposing the local port directly to the internet;
- plan backups and recovery;
- monitor failures and resource usage;
- use a deployment approach suited to your availability and security needs.

The lesson’s central trade-off is that self-hosting can reduce dependence on a hosted subscription, but the work of operating the instance becomes yours.

---

## 10. Active-recall quiz

Try answering before checking the key.

1. What is the difference between NPM and Node.js?
2. What does `node -v` show?
3. What does the `-g` option mean in the installation command shown?
4. What address does the lesson use to open the local instance?
5. Why is a local development setup not automatically production-ready?
6. What is the difference between installing a stable version and a prerelease?
7. Why does self-hosting involve more responsibility than n8n Cloud?
8. Does this transcript provide a complete Docker installation walkthrough?
9. What should you check before upgrading n8n?

### Answer key

1. Node.js is the runtime; NPM is the package manager distributed with Node.js in the demonstrated setup.
2. The installed Node.js version.
3. It requests a global package installation.
4. `http://localhost:5678`.
5. It may lack production deployment practices such as secure configuration, backups, monitoring, and uptime management.
6. Stable releases are intended for normal use; prereleases may contain newer features but have less stability assurance.
7. You manage the installation, configuration, updates, and local/server environment.
8. No. Docker is discussed conceptually, but the excerpt demonstrates NPM.
9. Current official documentation, release notes, compatibility, and a backup of important workflows/data.

---

## 11. Key takeaways

- n8n can be used through Cloud or run on infrastructure you manage.
- The lesson demonstrates a local NPM installation: install Node.js → verify `node` and `npm` → install n8n → run `n8n start` → open `localhost:5678` → complete initial setup.
- The lesson presents Docker as an option for production-style deployments, but does not show the Docker procedure in this excerpt.
- Self-hosting gives more direct control but transfers operational responsibilities to you.
- Prefer current official documentation over old recorded commands or version numbers.
