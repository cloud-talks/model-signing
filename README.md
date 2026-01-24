# Securing Your ML Model Supply Chain with OpenSSF Model Signing and Sigstore
As machine learning models are increasingly reused, shared, and deployed across environments, ensuring their authenticity and integrity has become a critical security challenge. Attacks such as model backdooring, malicious payloads, and tampered artifacts are no longer theoretical.

This repository contains hands-on examples showcasing how to secure the ML model supply chain using the OpenSSF Model Signing Standard and Sigstore. Let's learn how to sign models, verify them prior to loading, and embed verification into an end-to-end ML workflow.

## Prerequisites

| Tool | Min Version | Purpose |
|------|-------------|---------|
| **Kubernetes** | 1.29+ | Cluster to deploy on |
| **kubectl** | 1.29+ | Interact with cluster |
| **OpenSSL** | 1.1.1+ | Generate TLS certificates |
| **model_signing** | 1.0.1+ | Sign models (from [model-transparency-cli](https://github.com/sigstore/model-transparency)) |
