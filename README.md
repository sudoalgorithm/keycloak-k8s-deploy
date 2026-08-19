# keycloak-k8s-deploy

Deployment manifests for a highly available **Keycloak 26** on on-prem,
**air-gapped** Kubernetes, managed by the official
[Keycloak Operator](https://www.keycloak.org/operator/installation).
Two environments, one plain YAML file each — no Helm, no Kustomize, nothing to
install beyond `kubectl`. The clusters have no internet access: everything
they need is committed here (operator manifests vendored under `operators/`)
or mirrored into the internal container registry beforehand.

```
├── staging.yaml               # Namespace + PostgreSQL (CloudNativePG) + Keycloak + PDB
├── production.yaml            # Namespace + Keycloak + PDB (uses the external PostgreSQL cluster)
├── operators/
│   ├── install-notes.md       # step-by-step: image mirroring, operator installs, secrets, deploy, validation
│   ├── keycloak-operator/     # vendored official manifests, tag 26.5.2 (unmodified)
│   └── cnpg/                  # vendored official CNPG manifest 1.25.1 (unmodified)
└── docs/
    ├── manifests-explained.md   # field-by-field walkthrough of the two YAML files
    └── production-readiness.md  # the gate: every box checked before production is applied
```

**Images: official only, mirrored — nothing is built.** The four official
images (Keycloak server + operator, CNPG operator + PostgreSQL) are pulled on
a connected machine, saved to tar, carried across the gap through the approved
channel, and pushed to the internal registry (install-notes step 0). Keycloak
runs the **stock image in the operator's standard mode** (`startOptimized:
false`): it performs its `kc.sh build` as part of every pod start, and the CR
sizes the CPU limit and startup probe for that. Deliberate trade-off — a
restarted pod rejoins in ~1–2 min rather than ~30 s — in exchange for zero
custom artifacts to build, sign, or maintain inside the air gap.

## What gets deployed

| | Staging | Production |
|---|---|---|
| Keycloak | 2 instances (operator-managed) | 3 instances (operator-managed) |
| Resources per pod | 500m / 1200Mi (limits 2 CPU / 1500Mi) | 1500m / 2Gi (limits 3 CPU / 3Gi) |
| Image | stock `26.5.2`, `startOptimized: false` | stock `26.5.2`, `startOptimized: false` |
| Startup probe | 5 s × 120 (covers the in-pod build) | 5 s × 120 |
| PostgreSQL | In-cluster CloudNativePG, 1 instance (2 once a second PV exists) | External HA cluster (read-write endpoint) |
| DB pool | 20 connections per pod (40 total) | 20 connections per pod (60 total) |
| Hostname | `https://auth-staging.<CLIENT-DOMAIN>` | `https://auth.<CLIENT-DOMAIN>` |
| DB TLS | `verify-server` against the CNPG-generated CA | `verify-server` against the DBA-provided CA |
| Disruption budget | `minAvailable: 1` | `minAvailable: 2` |

Staging was first brought up 2026-08-19 on this shape (see install-notes
"Lessons from the first staging deploy").

Both environments are deliberately the same shape so staging genuinely
rehearses production behavior (clustering, pod spreading, rolling upgrades,
DB failover). The **only** intended differences are the DB source, hostname,
resource sizing, and the CNPG block — verify after any edit with:

```bash
diff staging.yaml production.yaml
```

## Design notes

- **HA:** anti-affinity is the operator's built-in default (Keycloak ≥ 26.0):
  pods of the same CR won't share a node. Each cluster needs **≥ 3 schedulable
  nodes**. The PDB keeps 2 of 3 pods up through node drains and upgrades.
- **Sessions** are persisted in PostgreSQL (Keycloak 26 default) — they survive
  a full cluster restart, and no sticky sessions are needed.
- **TLS** terminates at the load balancer in front of the API gateway;
  Keycloak runs `http-enabled=true` with `proxy.headers=xforwarded` and a
  pinned public hostname so issued tokens carry the correct issuer URL.
- **Exposure** follows the cluster convention — NodePort behind HAProxy:
  `keycloak-np` (31080) is the public path (nginx → Kong → HAProxy; Kong
  routes only the OIDC endpoints), `keycloak-admin-np` (31180) is the admin
  console on an internal HAProxy frontend reachable from the RCRC jump
  servers only. Access control is enforced at HAProxy (public frontend
  allows only the OIDC paths; admin frontend allows only the jump servers)
  and Kong (403 on `/admin`, `/realms/master`); `hostname.admin` just makes
  the console's own links point at the internal URL — Keycloak itself does
  not restrict `/admin` by hostname.
- **DB TLS:** connections to PostgreSQL use `db-tls-mode: verify-server` with
  the server CA mounted as a Keycloak truststore. Staging gets the CA for free
  from CloudNativePG (`keycloak-pg-ca`); production needs the external
  cluster's CA in a `keycloak-db-ca` secret.
- **Observability:** Prometheus metrics plus user event metrics
  (`event-metrics-user-enabled`) for login/failure alerting, JSON console logs
  for the log stack.
- **Secrets** are never in this repo. They are created by hand
  (`kubectl create secret`, see install notes); staging DB credentials are
  auto-generated by CloudNativePG.
- **Air-gapped delivery:** four images (Keycloak server + operator, CNPG
  operator + PostgreSQL) are pulled `--platform linux/amd64` on a connected
  machine, transferred, and pushed to the internal registry; `<REGISTRY>` in
  the YAML files points at it. Operator manifests are applied from the
  vendored copies in this repo, never from URLs. The Keycloak CR pins the
  mirrored stock `26.5.2` image (`startOptimized: false`, operator standard
  mode).
- **Versions** are pinned (Keycloak operator `26.5.2`, CloudNativePG `1.25.1`).
  Bump patch versions deliberately: re-vendor manifests + re-mirror images on
  a connected machine, then staging first, both files together.

## Deploying

Full command sequence, validation checklist included:
**[`operators/install-notes.md`](operators/install-notes.md)**. In short:

1. Get the four images: download the prebuilt bundle from the repo's
   **Releases** page (tag `images-26.5.2`, ~700 MB, `SHA256SUMS` included) —
   or regenerate it with the pull/save commands in the install notes. Then:
   connected machine → air gap → internal registry push.
2. Install the Keycloak operator (both clusters) and the CloudNativePG
   operator (staging cluster only) from the vendored manifests, images
   rewritten to the internal registry.
3. Create the secrets: `keycloak-bootstrap-admin` (both), `keycloak-db` and
   `keycloak-db-ca` (production only: credentials + CA of the external
   PostgreSQL), `registry-creds` if the registry needs auth.
4. Replace the placeholders — `<CLIENT-DOMAIN>` and `<REGISTRY>` in both
   files, `<CLIENT-PG-RW-ENDPOINT>` in `production.yaml`.
5. `kubectl apply -f staging.yaml`, wait for `Ready`, run the validation
   checklist.
6. Only after staging validates **and every box in
   [`docs/production-readiness.md`](docs/production-readiness.md) is
   checked**: `kubectl apply -f production.yaml`.

## Out of scope

- API gateway (Kong) route configuration — managed outside this repo.
- Realm content and migration (`KeycloakRealmImport` resources are produced by
  the migration pipeline, not stored here).
- The custom authenticator image (`spec.image` moves to a tag carrying the JAR
  when it exists).
