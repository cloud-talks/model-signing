# Workshop Hands-On Guide

## Prerequisites

```bash
# Install model_signing CLI
pip install model-signing

# Verify kubectl is working
kubectl get nodes
```

---

## Part 1: Setup

### Create kind cluster (if needed)

```bash
# Create a kind cluster
kind create cluster --name ml-workshop

# Verify cluster is running
kubectl cluster-info
```

### Build and load operator image

```bash
# Clone the operator repository (if not already done)
git clone https://github.com/sigstore/model-validation-operator.git
cd model-validation-operator

# Configure the operator to use the latest model_signing CLI image
# This adds the MODEL_TRANSPARENCY_CLI_IMAGE environment variable to the production overlay
cat >> config/overlays/production/production-config.yaml << 'EOF'
        - name: MODEL_TRANSPARENCY_CLI_IMAGE
          value: "ghcr.io/sigstore/model-transparency-cli:latest"
EOF

# Build the operator image locally using Docker
# --load: Loads the image into the local Docker daemon (required for kind)
# -t: Tags the image with the expected name and version
docker build --load -t ghcr.io/sigstore/model-validation-operator:v0.0.1 -f Dockerfile .

# Load the image into the kind cluster
# Kind clusters run in Docker containers and don't share images with your local Docker daemon.
# This command copies the image from your local Docker into the kind cluster's container runtime.
kind load docker-image ghcr.io/sigstore/model-validation-operator:v0.0.1 --name ml-workshop
```

### Install cert-manager (required for production overlay)

**cert-manager** is a Kubernetes add-on that automates the management and issuance of TLS certificates. The model-validation-operator uses cert-manager to:

1. **Secure webhook communication**: Kubernetes admission webhooks (which the operator uses to intercept pod creation) require TLS certificates for secure communication with the API server.
2. **Automatic certificate rotation**: cert-manager handles certificate renewal automatically, ensuring the operator remains functional without manual intervention.

Without cert-manager, you would need to manually create and manage TLS certificates for the operator's webhook server.

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
kubectl apply -k https://github.com/sigstore/model-validation-operator/config/overlays/production

# Wait for operator to be ready
kubectl get pods -n model-validation-operator-system -w

# Create namespace and storage for workshop
kubectl apply -f devconf-workshop/k8s/01-setup.yaml

# Verify PVC is bound
kubectl get pvc -n ml-workshop
```

---

## Part 2: Unsigned Model (Blocked)

```bash
# Navigate to workshop directory (from model-validation-operator root)
cd devconf-workshop

# Download BERT model
git clone --depth=1 "https://huggingface.co/bert-base-uncased"

# Remove .git directory (IMPORTANT!)
# The .git folder contains repository metadata that changes frequently.
# We only want to sign the actual model files, not git tracking data.
rm -rf bert-base-uncased/.git

# Upload model to cluster using ONE of these methods:

### Method A: ConfigMap + Job (for models < 1MB)
```bash
# Package the model
COPYFILE_DISABLE=1 tar -czvf model.tar.gz bert-base-uncased/

# Upload to ConfigMap
kubectl create configmap model-data -n ml-workshop --from-file=model.tar.gz

# Extract to PVC via job
kubectl apply -f k8s/02-copy-model.yaml
kubectl wait --for=condition=complete job/copy-model -n ml-workshop --timeout=120s
```

### Method B: kubectl cp (for larger models - RECOMMENDED for BERT)
```bash
# Create a helper pod that mounts the PVC
kubectl apply -f k8s/helper-pod.yaml
kubectl wait --for=condition=Ready pod/model-helper -n ml-workshop --timeout=60s

# Copy model directly to PVC
kubectl cp bert-base-uncased/ ml-workshop/model-helper:/data/bert-base-uncased/

# Verify the copy
kubectl exec model-helper -n ml-workshop -- ls -la /data/bert-base-uncased/

# Delete helper pod
kubectl delete pod model-helper -n ml-workshop
```

# Try to deploy (will fail!)
kubectl apply -f k8s/04-unsigned-demo.yaml

# Watch it fail
kubectl get pods -n ml-workshop -w

# Check error
kubectl logs unsigned-workload -n ml-workshop -c model-validation

# Cleanup
kubectl delete pod unsigned-workload -n ml-workshop
kubectl delete job copy-model -n ml-workshop
kubectl delete configmap model-data -n ml-workshop
```

---

## Part 3: Sign the Model

```bash
# Sign with Sigstore (opens browser)
model_signing sign sigstore ./bert-base-uncased --signature ./bert-base-uncased/model.sig

# Verify signature was created
ls -la bert-base-uncased/model.sig
```

---

## Part 4: Verify Locally

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

## Part 5: Deploy Signed Model (Success)

Upload the signed model using ONE of these methods:

### Method A: ConfigMap + Job
```bash
# Package with signature
COPYFILE_DISABLE=1 tar -czvf model.tar.gz bert-base-uncased/

# Verify signature is included
tar -tvf model.tar.gz | grep model.sig

# Upload to cluster
kubectl create configmap model-data -n ml-workshop --from-file=model.tar.gz

# Copy to PVC
kubectl apply -f k8s/02-copy-model.yaml
kubectl wait --for=condition=complete job/copy-model -n ml-workshop --timeout=120s
```

### Method B: kubectl cp (RECOMMENDED for BERT)
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
# Edit k8s/03-verify.yaml with YOUR email first!
# Then deploy
kubectl apply -f k8s/03-verify.yaml

# Watch it succeed
kubectl get pods -n ml-workshop -w

# Check verification logs
kubectl logs ml-workload -n ml-workshop -c model-validation
```

---

## Part 6: Attack Demo (Tampered Model Blocked)

```bash
# Create tampered copy
cp -r bert-base-uncased bert-base-uncased-tampered

# Tamper with it
echo "BACKDOOR INJECTION" >> bert-base-uncased-tampered/config.json

# Copy original signature (attacker tries to reuse it)
cp bert-base-uncased/model.sig bert-base-uncased-tampered/model.sig

# Cleanup previous
kubectl delete pod ml-workload -n ml-workshop
kubectl delete job copy-model -n ml-workshop
kubectl delete configmap model-data -n ml-workshop

# Package tampered model with correct directory name
cd bert-base-uncased-tampered && tar -czvf ../model.tar.gz --transform 's,^,bert-base-uncased/,' . && cd ..

# Upload and deploy
kubectl create configmap model-data -n ml-workshop --from-file=model.tar.gz
kubectl apply -f k8s/02-copy-model.yaml
kubectl wait --for=condition=complete job/copy-model -n ml-workshop --timeout=120s
kubectl apply -f k8s/03-verify.yaml

# Watch it fail
kubectl get pods -n ml-workshop -w

# Check error
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
| `No such file: model.sig` | Sign the model first (Part 3) |
| `.git directory issues` | Run `rm -rf bert-base-uncased/.git` |
