# Workshop Hands-On Guide

## How It Works

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           KUBERNETES CLUSTER                                    │
│                                                                                 │
│  ┌──────────────────────────────────────────────────────────────────────-───┐   │
│  │                         API SERVER                                       │   │
│  │                                                                          │   │
│  │   1. Pod created with label:                                             │   │
│  │      validation.ml.sigstore.dev/ml: "my-model-cr-name"                   │   │
│  │                           │                                              │   │
│  │                           ▼                                              │   │
│  │   2. MutatingWebhookConfiguration intercepts Pod creation                │   │
│  │                           │                                              │   │
│  └───────────────────────────┼──────────────────────────────────────────────┘   │
│                              │                                                  │
│                              ▼                                                  │
│  ┌──────────────────────────────────────────────┐    ┌────────────────────┐     │
│  │         MODEL VALIDATION OPERATOR            │◄───│   CERT-MANAGER     │     │
│  │         (Webhook Server)                     │    │                    │     │
│  │                                              │    │  Provides TLS      │     │
│  │  3. Looks up ModelValidation CR by label     │    │  certificates for  │     │
│  │  4. Injects init container with:             │    │  secure webhook    │     │
│  │     - model_signing verify command           │    │  communication     │     │
│  │     - signature path                         │    └────────────────────┘     │
│  │     - identity & OIDC issuer                 │                               │
│  └──────────────────────────────────────────────┘                               │
│                              │                                                  │
│                              ▼                                                  │
│  ┌───────────────────────────────────────────────────────────────────────-───┐  │
│  │                              POD                                          │  │
│  │                                                                           │  │
│  │   ┌─────────────────────────┐      ┌─────────────────────────┐            │  │
│  │   │  INIT CONTAINER         │      │  MAIN CONTAINER         │            │  │
│  │   │  (model-validation)     │      │  (your workload)        │            │  │
│  │   │                         │      │                         │            │  │
│  │   │  5. Runs FIRST          │ ──►  │  6. Runs ONLY if        │            │  │
│  │   │  Verifies signature     │      │  signature is valid     │            │  │
│  │   │  using Sigstore         │      │                         │            │  │
│  │   └─────────────────────────┘      └─────────────────────────┘            │  │
│  │                                                                           │  │
│  │   📁 /data (PVC)                                                          │  │
│  │   ├── model files...                                                      │  │
│  │   └── model.sig  ◄── Signature verified against Sigstore transparency log │  │
│  └───────────────────────────────────────────────────────────────────────────┘  │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### The required label

```yaml
metadata:
  labels:
    validation.ml.sigstore.dev/ml: "my-model-cr-name"  # ◄── This triggers the webhook!
```

When a Pod has this label, the webhook:
1. Finds the `ModelValidation` CR with matching name
2. Reads the signature config (identity, OIDC issuer, paths)
3. Injects an init container that verifies the model before your workload starts

**If verification fails → Pod stays in `Init:Error` → Main container never runs**

---

## Prerequisites

```bash
# Install model_signing CLI
pip install model-signing
```

---

## Part 1: Download and Sign Model Locally

### Download the model

```bash
# Download BERT model
git clone --depth=1 "https://huggingface.co/bert-base-uncased"

# Remove .git directory (IMPORTANT!)
# The .git folder contains repository metadata that changes frequently.
# We only want to sign the actual model files, not git tracking data.
rm -rf bert-base-uncased/.git
```

### Sign the model

```bash
# Sign with Sigstore (opens browser for authentication)
model_signing sign sigstore ./bert-base-uncased --signature ./bert-base-uncased/model.sig

# Verify signature was created
ls -la bert-base-uncased/model.sig
```

### Verify locally

```bash
# Replace YOUR_EMAIL with your actual email from signing
model_signing verify sigstore ./bert-base-uncased \
  --signature ./bert-base-uncased/model.sig \
  --identity "YOUR_EMAIL@example.com" \
  --identity_provider "https://github.com/login/oauth"
```

**Identity providers:**
- GitHub: `https://github.com/login/oauth`
- Google: `https://accounts.google.com`
- Microsoft: `https://login.microsoftonline.com`

---

## Part 2: Kubernetes Setup

### Create kind cluster

```bash
# Create a kind cluster
kind create cluster --name ml-workshop

# Verify cluster is running
kubectl cluster-info
```

### Install cert-manager

### Clone the operator repository

```bash
git clone https://github.com/infernus01/model-validation-operator.git
cd model-validation-operator
```
```bash
# Install cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.17.2/cert-manager.yaml

# Wait for cert-manager to be ready
kubectl wait --for=condition=Available deployment --all -n cert-manager --timeout=120s

# Verify all pods are running
kubectl get pods -n cert-manager
```


### Deploy the operator

```bash
# Deploy the operator (production overlay with cert-manager)
kubectl apply -k config/overlays/production

# Wait for operator to be ready
kubectl get pods -n model-validation-operator-system -w

# Create namespace and storage for workshop
kubectl apply -f k8s/01-setup.yaml

# Verify PVC is bound
kubectl get pvc -n ml-workshop
```

---

## Part 3: Deploy Signed Model

#### Method A: ConfigMap + Job
```bash
# Package with signature
tar -czvf model.tar.gz bert-base-uncased/

# Upload to cluster
kubectl create configmap model-data -n ml-workshop --from-file=model.tar.gz

# Copy to PVC
kubectl apply -f k8s/02-copy-model.yaml
kubectl wait --for=condition=complete job/copy-model -n ml-workshop --timeout=120s
```

#### Method B: kubectl cp (RECOMMENDED)
```bash
# Create helper pod
kubectl apply -f k8s/helper-pod.yaml
kubectl wait --for=condition=Ready pod/model-helper -n ml-workshop --timeout=60s

# Copy signed model directly
kubectl cp bert-base-uncased/ ml-workshop/model-helper:/data/bert-base-uncased/

# Verify copy (should include model.sig)
kubectl exec model-helper -n ml-workshop -- ls -la /data/bert-base-uncased/

# Delete helper pod
kubectl delete pod model-helper -n ml-workshop
```

### Deploy and Verify

```bash
# Edit k8s/03-verify.yaml with correct identity first!
# Then deploy
kubectl apply -f k8s/03-verify.yaml

# Watch it succeed
kubectl get pods -n ml-workshop -w

# Check verification logs
kubectl logs ml-workload -n ml-workshop -c model-validation
```

---

## Part 4: Attack Demo (Tampered Model Blocked)

```bash
# Create tampered copy
cp -r bert-base-uncased bert-base-uncased-tampered

# Tamper with it
echo "BACKDOOR INJECTION" >> bert-base-uncased-tampered/config.json

# Copy original signature (attacker tries to reuse it)
cp bert-base-uncased/model.sig bert-base-uncased-tampered/model.sig

# Cleanup previous deployment
kubectl delete pod ml-workload -n ml-workshop

# Upload tampered model using helper pod
kubectl apply -f devconf-workshop/k8s/helper-pod.yaml
kubectl wait --for=condition=Ready pod/model-helper -n ml-workshop --timeout=60s

# Overwrite the model with tampered version
kubectl exec model-helper -n ml-workshop -- rm -rf /data/bert-base-uncased
kubectl cp bert-base-uncased-tampered/ ml-workshop/model-helper:/data/bert-base-uncased/

kubectl delete pod model-helper -n ml-workshop

# Deploy workload (will fail verification)
kubectl apply -f devconf-workshop/k8s/03-verify.yaml

# Watch it fail
kubectl get pods -n ml-workshop -w

# Check error - signature won't match tampered content
kubectl logs ml-workload -n ml-workshop -c model-validation
```

---

## Cleanup

```bash
# Option 1: Delete everything but keep cluster
kubectl delete namespace ml-workshop
kubectl delete -k config/overlays/production
kubectl delete -f https://github.com/cert-manager/cert-manager/releases/download/v1.17.2/cert-manager.yaml

# Option 2: Delete the entire kind cluster (faster)
kind delete cluster --name ml-workshop

# Clean local files
rm -f model.tar.gz
rm -rf bert-base-uncased/ bert-base-uncased-tampered/
```

---

## Quick Troubleshooting

| Error | Fix |
|-------|-----|
| `Certificate's SANs do not match` | Use the email shown in error message |
| `Extra files found in model` | Use `COPYFILE_DISABLE=1 tar ...` |
| `No such file: model.sig` | Sign the model first (Part 1) |
| `.git directory issues` | Run `rm -rf bert-base-uncased/.git` |
| `not enough timestamps validated` | CLI version mismatch - verify locally first, then check operator uses latest CLI image |
