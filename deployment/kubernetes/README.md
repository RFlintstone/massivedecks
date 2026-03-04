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

3. **Set your hostname** — open `ingress.yaml` and replace `massivedecks.example.com` with
   your actual domain name. Ensure that DNS for this domain points to your cluster's ingress IP.

4. **Apply all manifests**:
   ```sh
   kubectl apply -f namespace.yaml
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
   kubectl -n massivedecks get pods
   kubectl -n massivedecks get ingress
   ```

## Notes

### TLS / HTTPS

To add HTTPS, configure a `tls` section in `ingress.yaml` with a certificate secret, or use a
Traefik middleware and a certificate resolver such as Let's Encrypt. For example:

```yaml
spec:
  ingressClassName: traefik
  tls:
    - hosts:
        - massivedecks.example.com
      secretName: massivedecks-tls
  rules:
    ...
```

### Image versions

The manifests reference `latest-release` image tags. For production stability, pin both the
`server` and `client` images to the **same** exact version tag or commit digest, and update
them together.

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

If you prefer to use the Traefik-native `IngressRoute` CRD instead of the standard Ingress
resource, replace `ingress.yaml` with a manifest like:

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: massivedecks
  namespace: massivedecks
spec:
  entryPoints:
    - web
  routes:
    - match: Host(`massivedecks.example.com`)
      kind: Rule
      services:
        - name: client
          port: 8080
```
