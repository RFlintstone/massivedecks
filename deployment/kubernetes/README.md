# Kubernetes Deployment

This directory contains Kubernetes manifests for deploying Massive Decks. It deploys:

- **client** — nginx serving the compiled frontend, proxying `/api` to the server
- **server** — Node.js game server
- **storage** — PostgreSQL 14 StatefulSet for persistent game data and caching
- **ingress** — Traefik ingress routing external HTTP traffic to the client

## Prerequisites

- A Kubernetes cluster with [Traefik](https://traefik.io/) installed as the ingress controller
- `kubectl` configured to talk to the cluster

## Quick Start

1. **Edit secrets** — open `secrets.yaml` and replace both `CHANGE ME` values with secure
   random strings. To generate a suitable server secret:
   ```sh
   cd ../../server && npm run generate-secret
   ```

2. **Edit the server config** — open `server.yaml` and update the `server-config` Secret:
   - Set the postgres `password` fields to the same value as `postgres-password` in `secrets.yaml`.
   - The `server-secret` is injected automatically from `secrets.yaml` via the `MD_SECRET`
     environment variable; you do not need to set it inside the config file.
   - Adjust any other settings as needed (deck sources, timeouts, etc.).

3. **Set your hostname** — the ingress is already configured for `cah.rubenflinterman.com`.
   Ensure that DNS for this domain points to your cluster's ingress IP.

4. **Apply all manifests**:
   ```sh
   kubectl apply -f namespace.yaml
   kubectl apply -f cert.yaml
   kubectl apply -f secrets.yaml
   kubectl apply -f postgres.yaml
   kubectl apply -f server.yaml
   kubectl apply -f client.yaml
   kubectl apply -f ingress.yaml
   ```
   Or apply the entire directory at once:
   ```sh
   kubectl apply -f .
   ```

5. **Verify the rollout**:
   ```sh
   kubectl -n massivedecks-rubenflinterman-com get pods
   kubectl -n massivedecks-rubenflinterman-com get ingressroutes
   ```

## Notes

### TLS / HTTPS

TLS is managed by cert-manager via `cert.yaml`. A `Certificate` resource requests a
Let's Encrypt certificate for `cah.rubenflinterman.com`, storing it in the
`cah-rubenflinterman-com-tls` secret referenced by `ingress.yaml`.

### Image versions

The manifests use `ghcr.io/rflintstone/massivedecks/{server,client}:latest` and
`imagePullPolicy: Always`. These images are built and pushed automatically by the
[publish workflow](../../.github/workflows/publish.yml) whenever a commit is pushed to
`main` or a `v*.*.*` tag is created.

For production stability, pin both the `server` and `client` images to the **same** exact
commit SHA tag (e.g. `ghcr.io/rflintstone/massivedecks/server:abc1234`) and update them
together when you want to upgrade.

### In-memory storage

If you do not need PostgreSQL, remove or skip `postgres.yaml` and replace the `storage` and
`cache` blocks in the server `config.json5` ConfigMap with:

```json5
storage: {
  type: "InMemory",
  abandonedTime: "PT15M",
  garbageCollectionFrequency: "PT5M",
},
cache: {
  type: "InMemory",
  checkAfter: "PT5M",
},
```

### Traefik IngressRoute (CRD)

The `ingress.yaml` already uses the Traefik-native `IngressRoute` CRD and routes traffic
for `cah.rubenflinterman.com` to the `client` service on port 8080.
