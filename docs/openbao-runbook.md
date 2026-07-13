# OpenBao runbook

## Why there is no auto-unseal

OpenBao encrypts its storage with a master key that it does not keep. On boot it comes
up *sealed* and cannot serve secrets until the key is reassembled from unseal shares.

Every auto-unseal option available on a single VPS is either impossible or a lie:

| Option | Why not |
|--------|---------|
| Cloud KMS seal | Reintroduces the cloud dependency this platform exists to avoid |
| Transit seal | Needs a *second* OpenBao to unseal the first. On one box it is circular — the unsealer boots sealed too |
| TPM seal | Requires a TPM. A typical VPS does not expose one |
| Static key seal | Parks the master key on the same disk it is meant to protect. This is a `.env` file with extra steps |

So: **manual unseal, consciously.** The mitigation is architectural, not operational —
nothing boot-critical depends on OpenBao. App startup secrets live in Sealed Secrets,
which self-decrypt with no human. If OpenBao is sealed at 03:00, the cluster is still
serving traffic and it can wait until morning.

**Upgrade path:** a second small box running the transit unsealer. That is the only
thing that makes auto-unseal real, and it needs hardware, not config.

## Initialise (once, ever)

```bash
kubectl -n openbao exec -it openbao-0 -- bao operator init -key-shares=3 -key-threshold=2
```

Prints 3 unseal keys and a root token.

**Store them in a password manager, not on this server.** An unseal key sitting on the
box it unseals protects nothing. Any two of the three keys reconstruct the master key.

## Unseal (after every reboot)

```bash
kubectl -n openbao exec -it openbao-0 -- bao operator unseal   # run twice, one key each
kubectl -n openbao exec -it openbao-0 -- bao status            # Sealed: false
```

## Kubernetes auth — pods authenticate as themselves

The point of the whole exercise: a pod proves its identity with the ServiceAccount JWT
it already has mounted. No static credential is ever stored in the cluster.

```bash
kubectl -n openbao exec -it openbao-0 -- sh
bao login <root-token>
bao auth enable kubernetes

# OpenBao runs inside the cluster, so it can reach the API server and read its own SA
# token — no CA cert or reviewer JWT needs to be copied in by hand.
bao write auth/kubernetes/config \
  kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443"

# Bind one ServiceAccount in one namespace to one policy. Not "*".
bao write auth/kubernetes/role/my-app \
  bound_service_account_names=my-app \
  bound_service_account_namespaces=prod \
  policies=my-app \
  ttl=1h
```

The pod then exchanges its JWT for a token that expires in an hour:

```bash
curl -s --request POST --data '{"jwt":"'"$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)"'","role":"my-app"}' \
  http://openbao.openbao.svc:8200/v1/auth/kubernetes/login
```

This is the same mechanism as IRSA on EKS. There, AWS is the validator and Amazon
operates it for free. Here, OpenBao is the validator and you operate it yourself.

## Disaster recovery

Storage is a `local-path` PVC on this node's disk. It is not replicated, and it is not
backed up by anything in this repo.

```bash
kubectl -n openbao exec openbao-0 -- tar czf - /openbao/data > openbao-data-$(date +%F).tar.gz
```

Without the unseal keys, that snapshot is undecryptable ciphertext. Back up both, and
keep them in different places.
