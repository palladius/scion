# Scion User Manual: Local Workstation & Swarm Setup

This manual documents the exact setup, authentication, and secret configuration patterns established for coordinating local workstation commands with the hosted Scion Hub on the GCE instance (`https://hub.scion.palladius.it`).

---

## 1. Local CLI Settings
To configure your local `scion` CLI client to connect to the hosted Hub server, run the following configuration commands:

```bash
scion config set --global hub.enabled true
scion config set --global hub.endpoint https://hub.scion.palladius.it
```

These settings are written globally to `~/.scion/settings.yaml`.

---

## 2. Authentication: Personal Access Token (PAT)
If Google OAuth for CLI/Desktop client is not configured on the GCP console (resulting in `401: invalid_client`), you can use a Personal Access Token instead.

### Generating a Token
1. Go to the Hub UI: **`https://hub.scion.palladius.it/`**
2. Click your user avatar (top right) -> **Profile / Settings** -> **User Access Tokens**.
3. Click **Generate Token**. Use scope **`agent:manage`** (or `--scopes agent:manage` on CLI) to grant start, stop, create, and list permissions.

### Clearing Stale Sessions (Troubleshooting)
If the CLI keeps opening the browser and throwing OAuth client errors, it means a stale or invalid session is cached. Clear it by running:
```bash
scion hub auth logout
```
If that fails or you want to force-clear, delete the credentials cache files:
```bash
rm -f ~/.scion/credentials.json
rm -f ~/.scion/scion-token
```

### Applying the PAT
Export the PAT token in your terminal environment:
```bash
export SCION_HUB_TOKEN="scion_pat_..."
```
The CLI will automatically detect `SCION_HUB_TOKEN` and bypass OAuth.

---

## 3. Registering/Linking a Project
To link your local repository folder (e.g., `~/git/scion`) to the Hub, go to the folder and run:
```bash
scion hub link
```
This registers the project on the Hub (slugified as `scion` or using the project name you select, e.g. `Image Consistency`).

---

## 4. Secret Management: Project Secrets
Secrets are write-only values managed securely in the Hub database and injected into agent containers at boot time.

### Environment variables (Obsidian Username/Password)
To inject secrets as environment variables inside agent containers:
```bash
scion hub secret set --project "Image Consistency" OBSIDIAN_USER "your_username"
scion hub secret set --project "Image Consistency" OBSIDIAN_PASSWORD "your_password"
```

### File Projection (GitHub / Git SSH Keys)
To project files (like private SSH keys) into the agent's container path (e.g., `/home/gemini/.ssh/id_rsa_scion`) with correct permissions (`0600`):
1. Generate a dedicated Ed25519 key locally on your workstation:
   ```bash
   ssh-keygen -t ed25519 -C "scion-agent@derek" -f ~/.ssh/id_rsa_scion -N ""
   ```
2. Upload the private key to the Hub as a file-type secret:
   ```bash
   scion hub secret set \
     --project "Image Consistency" \
     --type file \
     --target /home/gemini/.ssh/id_rsa_scion \
     GITHUB_SSH_KEY @~/.ssh/id_rsa_scion
   ```
3. Add the corresponding public key from `~/.ssh/id_rsa_scion.pub` to your GitHub account keys (**`https://github.com/settings/keys`**).

*Note:* If you project the key as `/home/gemini/.ssh/id_rsa_scion` (instead of the default `id_rsa` / `id_ed25519`), you should make sure the agent's shell or Git commands are configured to find it. You can project a custom `config` file to `/home/gemini/.ssh/config` using a `file`-type secret containing:
```text
Host github.com
  IdentityFile ~/.ssh/id_rsa_scion
  StrictHostKeyChecking no
```

---

## 5. Starting the Swarm
Use a custom script (e.g., `scion-progettino`) to start the coder and reviewer agents in your project:

```bash
#!/usr/bin/env bash
PROJECT_NAME="Image Consistency"

echo "Spawning coding swarm..."

# Start coder agent
scion start coder \
  --project "$PROJECT_NAME" \
  --harness gemini-cli \
  --role baseline \
  "You are the coder. Collaborate with the reviewer."

# Start reviewer agent
scion start reviewer \
  --project "$PROJECT_NAME" \
  --harness gemini-cli \
  --role baseline \
  "You are the reviewer. Audit changes made by the coder."
```
