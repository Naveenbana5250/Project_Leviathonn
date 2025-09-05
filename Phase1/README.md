# Kyverno + Cosign — Defense Team (Days 1–5)

> **Scope**: Deploy Kyverno, enforce signed images, set up Cosign signing, and wire CI/CD to build → push → sign by **digest** → enforce → test → deploy. This README documents **what’s in this repo** and **how to run/verify** it — no large file dumps here; we reference paths only.

## System Architecture Diagram

![Architecture Diagram](https://github.com/Naveenbana5250/yashwardhan/blob/main/Phase1/SystemArchitectureDiagram.png)

## Outcomes

* Kyverno admission control **denies** unsigned/unauthorized images.
* Public Cosign key published to the cluster as Secret **`kyverno/cosign-pub`**.
* GitHub Actions workflow builds → pushes → **signs by digest**; installs Kyverno; applies policies; runs a **negative test**; pins the digest in manifests; deploys signed workload.
* Reproducible validation with an evidence checklist.


## Repo Layout (authoritative paths)

* `.github/workflows/build-and-sign.yml` — CI pipeline (build, sign-by-digest, enforce, test, deploy)
* `Dockerfile` — container image for the app
* `Phase1/cluster/kyverno/policies/kyverno-foundation.yaml` — Kyverno base policies (signed-images, registry allowlist, security context, labels, etc.)
* `Phase1/cluster/manifests/deployment.yaml` — K8s workload; CI pins image **digest** here before deploy

> Never commit private keys. Only the **public** key is stored in-cluster via Secret; the private key lives in GitHub Secrets.



## Prerequisites

* Kubernetes cluster reachable from CI (provide kubeconfig via secret `KUBECONFIG_CONTENT`).
* Helm v3.
* Docker Hub account & token.
* Cosign installed locally (optional) for manual verification.



## Required GitHub Secrets

| Secret               | Purpose                                      |
| -------------------- | -------------------------------------------- |
| `DOCKERHUB_USERNAME` | Docker Hub username (e.g., `naveenbana5250`) |
| `DOCKERHUB_TOKEN`    | Docker Hub access token                      |
| `COSIGN_PRIVATE_KEY` | **Contents** of `cosign.key` (private)       |
| `COSIGN_PUBLIC_KEY`  | **Contents** of `cosign.pub` (public)        |
| `COSIGN_PASSWORD`    | Password used at key generation              |
| `KUBECONFIG_CONTENT` | Full kubeconfig YAML for the target cluster  |

> If you prefer base64 secrets, adjust the workflow to `base64 -d` them; current pipeline reads literal contents.



## How the Pipeline Works (overview)

**Workflow:** `.github/workflows/build-and-sign.yml`

1. Login to Docker Hub
2. Build & push `leviathan-app` (`latest` + `${GITHUB_SHA}` tags)
3. Export the **digest** and set `IMAGE=<repo>@sha256:...`
4. Install Cosign; write keys from GitHub Secrets; **sign by digest**
5. Verify signature (defense-in-depth in CI)
6. Setup `kubectl` + `helm`; write kubeconfig from secret
7. Install Kyverno, create/update `kyverno/cosign-pub` secret
8. Apply **`Phase1/cluster/kyverno/policies/kyverno-foundation.yaml`**
9. **Negative test**: try to run an unsigned/unauthorized image — expect **DENY**
10. Ensure standard labels in the deployment manifest
11. **Pin digest** into `Phase1/cluster/manifests/deployment.yaml`
12. Deploy the signed workload to the cluster



## Policy Pack (what’s enforced)

* **verify-signed-images**: Require Cosign signature, verify by digest, mutate to digest
* **restrict-registries**: Allow only `docker.io/naveenbana5250/*`
* **require-digest-and-ban-latest**: Audit `:latest`, require `@sha256` pinning
* **baseline-security-context**: Non-root, read-only FS, no privilege escalation, drop ALL caps, seccomp `RuntimeDefault`
* **require-resources**: CPU/memory requests & limits
* **require-standard-labels**: `app.kubernetes.io/name`, `owner`, `env`

> All rules live in: `Phase1/cluster/kyverno/policies/kyverno-foundation.yaml`.

## Runbook (local & CI)

### A) One-time (local)

1. Generate keys: `cosign generate-key-pair` (keep `cosign.key` private)
2. Put **contents** of keys into GitHub Secrets (`COSIGN_PRIVATE_KEY`, `COSIGN_PUBLIC_KEY`); add `COSIGN_PASSWORD`
3. Add `KUBECONFIG_CONTENT` (paste full kubeconfig YAML)

### B) Trigger a run

* Push to `main` → CI runs automatically.

### C) What to look for in CI

* Build step shows `digest: sha256:...`
* Cosign **sign** step succeeds
* Cosign **verify** step succeeds
* Kyverno installed and ready
* **Negative test** shows **blocked** (unsigned or unauthorized image)
* Deployment applied with image **pinned by digest**


## Evidence Checklist

1. CI summary with the pushed **digest**
2. `cosign sign` and `cosign verify` success logs (CI)
3. `kubectl -n kyverno get deploy,po` showing Kyverno ready
4. `kubectl get cpol` showing policies in place
5. Negative test: CI logs printing **"Unsigned or unauthorized image was blocked"**
6. Deployment in `Running` state with image pinned by digest (`kubectl get deploy leviathan-app -o yaml | grep image:`)



## Troubleshooting

* **`decrypt: encrypted: decryption failed`** → Wrong `COSIGN_PASSWORD` or mismatched key. Regenerate keys and update: `COSIGN_PRIVATE_KEY`, `COSIGN_PUBLIC_KEY`, `COSIGN_PASSWORD`, and refresh the in‑cluster `cosign-pub` secret with the new **public** key.
* **`no matching signatures`** → Ensure CI signs **by digest** (`image@sha256:...`) and policies verify by digest.
* **Kyverno deny not triggering** → `imageReferences` must match `docker.io/naveenbana5250/*`; re‑create pods (policy runs at admission); ensure `validationFailureAction: Enforce`.
* **Webhook/connectivity errors** → Verify kubeconfig in `KUBECONFIG_CONTENT` is correct; cluster reachable from GitHub runners; Kyverno rollout is **ready** before applying policies.



## Attribution

* Docker Hub: `docker.io/naveenbana5250`
* Image name used: `leviathan-app` (adjust if renamed)
