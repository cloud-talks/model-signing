# Securing Your ML Model Supply Chain with OpenSSF Model Signing and Sigstore
As machine learning models are increasingly reused, shared, and deployed across environments, ensuring their authenticity and integrity has become a critical security challenge. Attacks such as model backdooring, malicious payloads, and tampered artifacts are no longer theoretical.

This repository contains hands-on examples showcasing how to secure the ML model supply chain using the OpenSSF Model Signing Standard and Sigstore. 

Let's learn how to sign models, verify them prior to loading, and embed verification into an end-to-end ML workflow.

---

## Prerequisites

| Tool | Purpose |
|------|---------|
| **Docker** or **Podman** | Container runtime (required for kind) |
| **kind** | Local Kubernetes clusters |
| **kubectl** | Interact with Kubernetes |
| **model_signing** | Sign & verify ML models (from [model-transparency-cli](https://github.com/sigstore/model-transparency)) |
| **OpenSSL** | TLS certificates (optional) |
| **Git** | Clone repos |

---

## Installation

### macOS

```bash
# Docker Desktop (or Podman)
brew install --cask docker
# OR: brew install podman

# kind
brew install kind

# kubectl
brew install kubectl

# model_signing
pip install model-signing

# OpenSSL (optional - usually pre-installed)
brew install openssl
```

### Linux (Fedora/RHEL)

```bash
# Docker (or Podman)
sudo dnf install -y docker
# OR for Podman: sudo dnf install -y podman

# kind
curl -Lo ./kind https://kind.sigs.k8s.io/dl/latest/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

# kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# model_signing
pip install model-signing
```

---

## Verify Installation

```bash
docker --version      # or: podman --version
kind version
kubectl version --client
model_signing --version
```

All commands should return version numbers. You're ready!

---

## The Problem

### Why Model Signing?

ML models are shared and deployed at massive scale, but most organizations have **no way to verify**:
- Is this model authentic?
- Has it been tampered with?
- Who created/approved it?

### Real-World Attacks

| Attack | Description |
|--------|-------------|
| **Model Backdoors** | Malicious behavior triggered by specific inputs |
| **Weight Poisoning** | Subtle modifications that degrade performance or add bias |
| **Deserialization Attacks** | Pickle files executing arbitrary code on load |
| **Model Swapping** | Replacing legitimate models with malicious ones |

**Model signing solves this** by cryptographically proving:
1. **Integrity** - The model hasn't been modified
2. **Authenticity** - Who signed/approved it
3. **Non-repudiation** - Signing is recorded in a public log

---