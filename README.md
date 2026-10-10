# k8s-flux

GitOps configuration for the homelab Kubernetes cluster(s), reconciled by
[Flux](https://fluxcd.io).

## Repository layout

```
clusters/          One directory per cluster, one subdirectory per environment (Flux entrypoint).
apps/              Application workloads.
vms/               KubeVirt virtual machines.
infrastructure/    Cluster infrastructure, one directory per deployable unit.
```

Each unit is a Kustomize `base/`; an env directory (`development/`, `production/`) exists only
when that environment diverges. `clusters/<cluster>/<env>/` is bootstrapped with
`flux bootstrap --path=clusters/<cluster>/<env>` and contains the generated `flux-system/`
(do not edit). `infrastructure.yaml`, `apps.yaml` and `vms.yaml` hold one Flux `Kustomization`
per unit; `dependsOn` (with `wait: true`) is the dependency graph.

## Clusters

- **bee** — active cluster.
- **oldbee** — archived, no longer reconciled; kept for reference.

## bee

Dependencies between units:

```mermaid
flowchart TD
  gateway-api-crds --> cert-manager
  gateway-api-crds --> nginx-gateway-fabric
  cert-manager --> external-secrets
  cert-manager --> external-secrets-certs
  external-secrets --> external-secrets-certs
  external-secrets --> external-secrets-store
  external-secrets-certs --> bitwarden-sdk-server
  bitwarden-sdk-server --> external-secrets-store
  cert-manager --> cert-manager-issuers
  external-secrets-store --> cert-manager-issuers
  nginx-gateway-fabric --> gateways
  cert-manager-issuers --> gateways
  piraeus-operator --> linstor-cluster
  prometheus-operator --> linstor-cluster
  kubevirt-operator --> kubevirt
  gateways --> degoog
  external-secrets-store --> degoog
  linstor-cluster --> degoog
  gateways --> kanidm
  linstor-cluster --> kanidm
  gateways --> openwebui
  external-secrets-store --> openwebui
  linstor-cluster --> openwebui
  gateways --> outline
  external-secrets-store --> outline
  linstor-cluster --> outline
  gateways --> vaultwarden
  external-secrets-store --> vaultwarden
  linstor-cluster --> vaultwarden
  gateways --> hermes-agent
  linstor-cluster --> hermes-agent
  kubevirt --> vms
  linstor-cluster --> vms
  prometheus-operator --> node-exporter
  prometheus-operator --> kube-state-metrics
  prometheus-operator --> alertmanager
  node-exporter --> prometheus
  kube-state-metrics --> prometheus
  alertmanager --> prometheus
  linstor-cluster --> prometheus
  prometheus --> grafana
  gateways --> grafana
  external-secrets-store --> grafana
  linstor-cluster --> grafana
```

### Infrastructure

- `gateway-api-crds` — upstream Gateway API CRDs.
- `cert-manager` — cert-manager.
- `external-secrets`, `external-secrets-certs` — external-secrets operator and its certificates.
- `bitwarden-sdk-server` — Bitwarden Secrets Manager SDK server.
- `external-secrets-store` — `ClusterSecretStore` backed by Bitwarden.
- `cert-manager-issuers` — `letsencrypt` ClusterIssuer (DNS-01 via Cloudflare).
- `nginx-gateway-fabric` — Gateway API data plane (hostPort 80/443 on every node).
- `gateways` — shared `bee-gateway` for `*.xtinto.com`, HTTP redirected to HTTPS.
- `piraeus-operator`, `linstor-cluster` — LINSTOR storage (`local`, `replicated`).
- `prometheus-operator`, `node-exporter`, `kube-state-metrics`, `alertmanager`, `prometheus`,
  `grafana` — monitoring.
- `kubevirt-operator`, `kubevirt` — virtualization.

### Apps

- `degoog` — `search.xtinto.com`; depends on `gateways`, `external-secrets-store`, `linstor-cluster`.
- `kanidm` — `idm.xtinto.com`; depends on `gateways`, `linstor-cluster`.
- `openwebui` — `chat.xtinto.com`; depends on `gateways`, `external-secrets-store`, `linstor-cluster`.
  Official Helm chart, SQLite on a `replicated` PVC, Kanidm OIDC. LLM connections are added later in
  the admin panel.
- `outline` — `outline.xtinto.com`; depends on `gateways`, `external-secrets-store`,
  `linstor-cluster`. Raw manifests with a bundled PostgreSQL 17 and Valkey 8 on `replicated` PVCs,
  local file storage and Kanidm OIDC. The Kanidm OAuth2 client `outline` already exists; its secret
  is stored manually in Bitwarden as `outline/oidc-client-secret`. Workspace data is imported in the
  admin settings from a JSON export of the old instance.
- `vaultwarden` — `passwords.xtinto.com`; depends on `gateways`, `external-secrets-store`,
  `linstor-cluster`. Official Helm chart, SQLite on a `replicated` PVC. Credentials are imported
  from the old instance via the web vault's export/import.
- `hermes-agent` — `hermes.xtinto.com`; depends on `gateways`, `linstor-cluster`. Nous Research's
  self-improving agent, run in gateway mode from the official `nousresearch/hermes-agent` image (no
  Helm chart). The dashboard UI/management API is served under `/dashboard` (Kanidm OIDC) and the
  OpenAI-compatible API under `/api/v1`; all state lives on a `replicated` PVC. NetworkPolicies
  default-deny both directions (cluster CIDRs are Flux `postBuild` variables) so the agent can only
  reach DNS, the internet and the gateway VIP (Kanidm), never other pods or the Kubernetes API. The
  model, the OpenRouter key and `API_SERVER_KEY` (generated on first boot) are all managed in the
  dashboard, not in Git.

### VMs

A VM lives in `vms/<cluster>/<vm>/base` with its own Kustomization in `vms.yaml`, depending on
`kubevirt` and `linstor-cluster` (plus `external-secrets-store` when cloud-init data comes from
an `ExternalSecret`).

## Secrets

No plaintext secrets. All secrets go through external-secrets backed by Bitwarden
(`ClusterSecretStore: bitwarden-secrets-manager`). Anything committed here is compromised:
remove and rotate it. `external-secrets-store` becomes Ready only after the
`bitwarden-access-token` Secret exists in `external-secrets` (a manual bootstrap step).
