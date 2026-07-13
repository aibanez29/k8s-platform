# K8s Platform

A single-node Kubernetes platform that actually serves traffic: GitOps, admission
control, TLS, and workload identity on one VPS. No cloud dependency — no managed
control plane, no cloud KMS, no cloud load balancer. Every component is
MPL / Apache / MIT.

Four commands to stand it up. Everything after that is a `git push`.

## Stack

| Layer | Tool | Why this one |
|-------|------|--------------|
| Cluster | k3s | Runs as a systemd unit — survives a reboot. `kind` does not. |
| GitOps | ArgoCD | App-of-apps; this repo is the only source of truth |
| Ingress | ingress-nginx | DaemonSet + hostPort. There is no cloud LB to provision. |
| TLS | cert-manager | Let's Encrypt HTTP-01, auto-renewed |
| Policy | Kyverno | Admission control |
| Secrets (boot) | Sealed Secrets | Encrypted in Git; cluster self-heals after a reboot |
| Secrets (dynamic) | OpenBao | Short-lived creds via ServiceAccount JWT |

### Why OpenBao and not Vault

Vault moved to the BUSL license in 2023 — it is no longer open source.
[OpenBao](https://openbao.org) is the MPL-2.0 fork under the Linux Foundation and is
API-compatible, so the Kubernetes auth method is unchanged.

### Why two secret managers

They solve different problems, and picking only one of them is a bug:

- **Sealed Secrets** holds *boot-critical* secrets — the database password an app needs
  in order to start. Encrypted with the controller's public key, safe to commit to a
  public repo, decrypted in-cluster with no human in the loop. **The cluster comes back
  up on its own after a reboot.**
- **OpenBao** holds *dynamic* secrets — credentials minted on demand, scoped to a pod's
  identity, that expire by themselves. This is what a cloud hands you for free as IRSA
  (EKS) or Workload Identity (GKE). Here you run it yourself.

OpenBao comes up **sealed** after every reboot and needs a manual unseal, which is
exactly why nothing boot-critical is allowed to depend on it. See the
[unseal runbook](docs/openbao-runbook.md) — a deliberate trade, not an oversight.

## Bootstrap

Ubuntu 24.04. 2 vCPU / 4 GB minimum, 8 GB comfortable.

```bash
# 1. Cluster. Traefik and ServiceLB are disabled on purpose: this repo brings its own
#    ingress, and ServiceLB would fight it for :80/:443.
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable traefik --disable servicelb --write-kubeconfig-mode 644" sh -
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml

# 2. ArgoCD. --server-side is not optional: the ApplicationSet CRD is larger than the
#    262kB cap on the annotation that a client-side apply stashes its config in, and a
#    plain `kubectl apply` fails on that one CRD alone.
kubectl create namespace argocd
kubectl apply --server-side -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl wait --for=condition=available --timeout=300s -n argocd deployment/argocd-server

# 3. The app-of-apps. Last thing applied by hand — ArgoCD takes over from here and
#    pulls the rest of the platform out of this repo.
kubectl apply -f bootstrap/root-app.yaml

# 4. Watch it converge.
kubectl get applications -n argocd -w
```

ArgoCD admin password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d
```

## Layout

```
bootstrap/
  root-app.yaml            the one Application applied by hand
k8s/
  platform/                one ArgoCD Application per component, values inline
  config/                  ClusterIssuers, namespaces
  kyverno/policies/        admission policies
  apps/                    workloads (kustomize base + dev/staging/prod overlays)
```

Nothing is installed with `helm install`. The Helm charts are consumed *as ArgoCD
sources*, with their values committed here — so the cluster's whole configuration is
diffable in a pull request, and manual drift gets reverted by `selfHeal`.

## Exposing an app

Point an A record at the VPS IP, then:

```yaml
metadata:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-staging   # letsencrypt-prod once it works
spec:
  ingressClassName: nginx
  tls:
    - hosts: [app.example.com]
      secretName: app-tls
```

cert-manager solves the HTTP-01 challenge through ingress-nginx and renews on its own.
Start on `letsencrypt-staging`: production Let's Encrypt rate-limits you to 5 failures
per hour per domain, and one typo locks you out for the rest of it.

## Sealing a secret

`kubeseal` encrypts against the in-cluster public key. The output is safe to commit to
this public repo.

```bash
kubectl create secret generic db --dry-run=client --from-literal=password=hunter2 -o yaml \
  | kubeseal --controller-name sealed-secrets-controller --controller-namespace sealed-secrets -o yaml \
  > k8s/apps/base/db-sealed.yaml
```

**Back up the controller's private key.** Lose it and every SealedSecret in this repo is
permanently undecryptable:

```bash
kubectl -n sealed-secrets get secret \
  -l sealedsecrets.bitnami.com/sealed-secrets-key -o yaml > sealed-secrets-key.yaml   # keep this OFF the server
```

## Known limits

Single node: no HA, and a reboot is a full outage until the node is back. Storage is
`local-path` — it lives on this node's disk and is not replicated. OpenBao needs a
manual unseal on every boot ([why](docs/openbao-runbook.md)).

None of these are worth fixing on one box. All of them are worth knowing about.
