# bash-scaffold

> A niche, partially vibe-coded bash script that acts as a simpler iterative deployment scripting tool. It trades complex capabilities for ease of use and zero dependencies.
> Made for my own use case, feel free to add, fork, or whatever.

**bash-scaffold** is a single-file deployment engine designed for Linux servers. It bridges the gap between your system configuration (secrets, certs, env vars) and your application source code. Instead of wrestling with complex CI/CD pipelines for simple projects, `scaffold` lets you define deployment logic using a procedural syntax directly on your server.

## Features

* **Zero Dependencies:** Runs on standard Bash (requires `perl`, `git`, `unzip`, `curl` which are standard on most distros).
* **Procedural Logic:** Define `ITERATE` (build), `UP` (start), and `DOWN` (stop) procedures in a simple text file.
* **Smart Versioning:** Automatically tracks Major.Minor.Patch versions and maintains a history tree.
* **Injection Engine:** Inject secrets or configuration into files *before* the application starts (e.g., inject an API key into a compiled JS file or `.env`).
* **Context Aware:** Execute Bash commands scoped to specific directories using variables.
* **Rollbacks:** Switch between versions instantly with `scaffold up`.

---

## Installation

Since `bash-scaffold` is a single script, installation is manual but instant.

### Option 1: Copy & Paste
1.  Create a file named `scaffold` in `/usr/local/bin`:
    ```bash
    sudo touch /usr/local/bin/scaffold
    sudo chmod +x /usr/local/bin/scaffold
    sudo nano /usr/local/bin/scaffold
    ```
2.  Paste the **v3.1** source code into the editor.
3.  Save and exit.

### Option 2: Curl (If hosted)
```bash
# Assuming you host the raw script somewhere
sudo curl -L [https://your-url.com/scaffold](https://your-url.com/scaffold) -o /usr/local/bin/scaffold
sudo chmod +x /usr/local/bin/scaffold
````

-----

## Quick Start

### 1\. Initialize

Go to the directory where you want your deployments to live (e.g., `/var/www/myapp`).

```bash
scaffold init
```

This creates:

  * `.scmanif`: A hidden database tracking versions.
  * `SCAFFOLD`: The default configuration rule file.

### 2\. Configure

Edit the `SCAFFOLD` file to define your logic.

```text
# Define variables (Quoted strings supported)
VARIABLE app_port "3000"
VARIABLE deploy_path "."

# Define where the code comes from
SCAFFOLD BASE GIT [https://github.com/user/my-repo.git](https://github.com/user/my-repo.git)

# Build Logic (Runs once per version download)
PROCEDURE ITERATE BEGIN
    SCAFFOLD BASH "npm install" AT deploy_path
    SCAFFOLD BASH "npm run build" AT deploy_path
PROCEDURE ITERATE END

# Startup Logic
PROCEDURE UP BEGIN
    SCAFFOLD BASH "pm2 start app.js --name myapp-%app_port%" AT deploy_path
PROCEDURE UP END

# Shutdown Logic (Runs on the OLD version before UP runs on the NEW one)
PROCEDURE DOWN BEGIN
    SCAFFOLD BASH "pm2 delete myapp-%app_port%" AT deploy_path
PROCEDURE DOWN END
```

### 3\. Deploy

Run an iteration. This downloads the code, runs `ITERATE`, shuts down the old version (if any), and starts the new one.

```bash
scaffold iterate --minor
```

-----

## The SCAFFOLD Syntax

The `SCAFFOLD` file controls everything.

### Variables & Base

```text
VARIABLE name "value"
SCAFFOLD BASE <GIT|ZIP|FOLDER> <URL_OR_PATH>
```

  * Variables can be used anywhere via `%name%`.
  * Variables matching a directory name allow smart usage in `AT` commands (e.g., `AT %path%` or just `AT path`).

### Files (System Assets)

Register files that exist on the **Server** so they can be injected into the **Source Code**.

```text
FILE prod_env "./secrets/.env.production"
```

### Commands (Inside Procedures)

#### `SCAFFOLD FILE`

Moves registered system files into the deployment folder.

```text
# Copy file
SCAFFOLD FILE PLACE prod_env . --AS .env

# Symlink file (Good for certs/large assets)
SCAFFOLD FILE TARGET my_cert ./ssl/
```

#### `SCAFFOLD ALTER`

Modifies text files inside the source code using Perl-compatible Regex.

```text
# Inject variable content at a specific line
SCAFFOLD ALTER INJECT my_var config.js --at 5

# Replace a specific string marker
SCAFFOLD ALTER INJECT my_var config.js --at "REPLACE_ME"

# Replace a block of text between two markers
SCAFFOLD ALTER INPLACE my_var file.txt --at "# START" --end "# END"
```

#### `SCAFFOLD BASH`

Runs shell commands.

```text
SCAFFOLD BASH "docker compose up -d" AT .
SCAFFOLD BASH "npm install" AT my_sub_folder
```

-----

## CLI Reference

### `scaffold iterate`

Deploys a new version.

  * `--git <url>`, `--zip <file>`, `--folder <path>`: Override the source defined in the SCAFFOLD file.
  * `--release`: Increment Major version (X.0.0).
  * `--version`: Increment Minor version (0.X.0).
  * `--minor`: Increment Patch version (0.0.X).
  * `--rv <X> --vv <Y> --mv <Z>`: Manually define the version number.
  * `--v<VarName> "<Value>"`: Override a variable for this run (e.g., `--vport "8080"`).

### `scaffold list`

Shows a tree view of deployed versions.

  * `--rv <X>`: Filter by Release version.

### `scaffold up`

Rollback or switch to a specific version.

  * `--rv <X> --vv <Y> --mv <Z>`: Target version to switch to.
  * *Note: This runs `DOWN` on the current version, then `UP` on the target.*

### `scaffold down`

Stops the currently active deployment (runs `PROCEDURE DOWN`).

### `scaffold proc`

Manually execute a specific procedure.

  * `scaffold proc <NAME>`
  * Useful for maintenance tasks (e.g., `PROCEDURE DB_BACKUP`).

### `scaffold compactrule`

Debugging tool. Prints the fully parsed `SCAFFOLD` file with:

  * `INCLUDE`s resolved.
  * Variables expanded.
  * CLI overrides applied.
  * Active values commented for verification.

-----

## Example: Advanced Usage

**Scenario:** You want to deploy a hotfix, but override the port just for this run, and use a local folder instead of git.

```bash
scaffold iterate --minor --folder ./hotfix-v1 --vapp_port "4000" --debug
```

This will:

1.  Copy source from `./hotfix-v1`.
2.  Override `%app_port%` to `4000` in all your scripts.
3.  Run in debug mode (printing every bash execution).

<!-- end list -->

```
```
