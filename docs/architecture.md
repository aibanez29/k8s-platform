# K8s Platform Architecture

One VPS, one node, no cloud. Every decision below follows from those three facts.

## Flow

```
GitHub (this repo — single source of truth)
  │  push to main
  ▼
ArgoCD ── root Application (app-of-apps) ── k8s/platform/*
  │
  ▼
k3s (systemd unit, survives reboot)
  │
  ├── ingress-nginx ─── DaemonSet + hostPort :80/:443 ─── the VPS public IP
  │     └── cert-manager ─── Let's Encrypt HTTP-01
  │
  ├── Kyverno (admission)
  │     └── require resource limits  (enforced)
  │
  ├── Sealed Secrets ─── boot-critical secrets, decrypt unattended
  │
  ├── OpenBao ─── dynamic secrets, sealed on boot (manual unseal)
  │
  └── dev / staging / prod namespaces
```

## Decisions

**k3s, not kind.** kind runs the cluster inside Docker containers: no systemd, so it does
not come back after a reboot, and its storage is ephemeral by design. Upstream says it is
for testing Kubernetes, not for serving traffic. k3s is the same Kubernetes, packaged as a
service that starts on boot.

**No `type: LoadBalancer`, anywhere.** A LoadBalancer Service asks the infrastructure to
provision an NLB/ALB. On a VPS there is no such infrastructure, so the Service sits in
`<pending>` forever. ServiceLB and MetalLB paper over this by handing back the node's own
IP — satisfying the manifest without balancing anything. This platform is honest about it
instead: ingress-nginx runs as a DaemonSet with `hostPort`, binding :80/:443 on the node.
The VPS public IP *is* the ingress address. k3s ships Traefik and ServiceLB by default and
the bootstrap disables both, so nothing fights for those ports.

**Helm charts as ArgoCD sources, never `helm install`.** A chart installed from a terminal
is state that exists nowhere in Git. Consumed as an ArgoCD `source` with its values
committed here, the same chart becomes a reviewable diff — and `selfHeal` reverts anyone
who edits the live cluster by hand.

**Sync waves.** CRDs must exist before the resources that use them. cert-manager and
Kyverno land in wave 0; the ClusterIssuers and ClusterPolicies that depend on their CRDs
land in wave 1. Without the waves this is a race, and the policies lose it.

**Kyverno excludes the platform namespaces.** `require-resource-limits` is enforced, and
several upstream platform charts ship without limits. Enforcing against them deadlocks the
cluster: the admission webhook rejects the ingress, cert and secret plumbing that
everything else is waiting on. Workload namespaces get no such exemption.

**Two secret managers, on purpose.** Sealed Secrets for anything an app needs in order to
boot — it decrypts unattended, so a reboot recovers with no human. OpenBao for dynamic,
short-lived credentials tied to a pod's ServiceAccount identity. OpenBao boots sealed and
cannot be auto-unsealed on a single node without either a cloud KMS or leaving the master
key on the disk it protects, so it is kept off the boot path entirely. See
[openbao-runbook.md](./openbao-runbook.md).

## Workload identity

```
[Pod] ── ServiceAccount JWT (already mounted, no static secret) ──▶ [OpenBao K8s auth]
                                                                          │
                                                       validates JWT against the K8s API
                                                                          │
                                                          short-lived token, TTL 1h
                                                                          ▼
                                                                       [Pod]
```

Identical in shape to IRSA on EKS or Workload Identity on GKE. The only difference is who
validates the JWT: there it is the cloud provider, operated for you; here it is OpenBao,
operated by you. Running it yourself is the point — the same mechanism with the magic
removed.

## What this deliberately does not have

No HA — one node. No replicated storage — `local-path` on this node's disk. No cluster
autoscaler — nothing to scale onto. No external-dns — with a handful of hostnames, a
wildcard A record is one DNS entry instead of a controller plus API credentials.

Each is a real gap on a real cluster. On one box, every one of them would be complexity
paid for and never used.
