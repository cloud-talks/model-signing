# Dev Container

## Attendees Need
- Docker or Podman
- VS Code
- Git

## Included Tools
kubectl, kind, kustomize, openssl, model_signing

---

## Step-by-Step Setup

### 1. Install Docker Desktop
- Go to https://www.docker.com/products/docker-desktop
- Download for your OS (Mac/Windows/Linux)
- Install and start Docker Desktop
- Verify: open terminal, run `docker --version`

### 2. Install VS Code
- Go to https://code.visualstudio.com
- Download and install

### 3. Install Dev Containers Extension
- Open VS Code
- Click the **Extensions** icon on the left sidebar (or press `Ctrl+Shift+X` / `Cmd+Shift+X`)
- Search for **"Dev Containers"**
- Click **Install** on "Dev Containers" by Microsoft

### 4. Clone This Repository
- Open terminal
- Run:
  ```bash
  git clone <repository-url>
  ```

### 5. Open in Dev Container
- Open VS Code
- Click **File** → **Open Folder**
- Select the cloned `model-validation-operator` folder
- Click **Open**
- A popup appears at bottom-right: **"Folder contains a Dev Container configuration file"**
- Click **"Reopen in Container"**
- Wait 2-5 minutes (first time only, it's building the container)

### 6. Verify Everything Works
- Once container is ready, open terminal in VS Code: **Terminal** → **New Terminal**
- Run these commands to verify:
  ```bash
  kubectl version --client
  kind version
  kustomize version
  openssl version
  model_signing --version
  ```

---

## Alternative: Using CLI (No VS Code)

```bash
# Install devcontainer CLI
npm install -g @devcontainers/cli

# Clone and enter repo
git clone <repository-url>
cd model-validation-operator

# Start container
devcontainer up --workspace-folder .

# Open shell inside container
devcontainer exec --workspace-folder . bash
```
