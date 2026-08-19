# Production Readiness Gate

Production is deployed **only** when every line below is checked. Nothing
here is new — each item is either the client-approved design or a lesson
staging already taught us. The gate exists so prod-day is a checklist run,
not a judgement call.

Staging validated 2026-08-19 with interim settings; production runs the
**design** settings. The two are deliberately different (see the table at
the end) and the interim shortcuts **do not** carry over.

---

## A. Inputs from the client (block until all are in hand)

- [ ] **PostgreSQL read-write endpoint** (`<CLIENT-PG-RW-ENDPOINT>`) —
      VIP / Patroni proxy / PgBouncer; confirm it follows failover.
- [ ] Database `keycloak` + owner role `keycloak` created on that cluster;
      credentials delivered (→ secret `keycloak-db`).
- [ ] **PostgreSQL server CA** (PEM) for `db-tls-mode: verify-server`
      (→ secret `keycloak-db-ca`). If their PG does not serve TLS, that is a
      **recorded risk decision with the client** before go-live, not a
      silent removal.
- [ ] PostgreSQL sized per the requirement sheet (4 vCPU / 16 GB / 50 GB SSD
      / ≥ PG 15) and the **70-connection budget** approved.
- [ ] Production auth domain (`auth.<CLIENT-DOMAIN>`) decided: retained from
      vendor (invisible cutover) or new (apps update base URL).
- [ ] **≥ 3 schedulable worker nodes** on the prod cluster
      (`kubectl get nodes`, check taints) — otherwise 3 pods cannot spread.
- [ ] How Kong reaches cluster services (NodePort / routable pods / other).
- [ ] GitLab registry reachable **from the prod cluster nodes**; deploy token
      with `read_registry`.

## B. Artifacts (must exist before prod day)

- [ ] **Pre-optimized image** `…/keycloak:26.5.2-optimized` pushed to the
      registry and **proven on staging first** (`startOptimized: true`, pod
      Ready in < 60 s, no "Updating the configuration…" line in logs).
      Non-negotiable: production never runs the in-pod build.
- [ ] Keycloak operator image `26.5.2` and the vendored manifests in this
      repo at the same version as staging.
- [ ] `production.yaml` placeholders filled: `<REGISTRY>`, `<CLIENT-DOMAIN>`,
      `<CLIENT-PG-RW-ENDPOINT>`; `imagePullSecrets` name matches the secret created (`registry-creds`).
- [ ] `diff staging.yaml production.yaml` reviewed — only the intended deltas
      (instances, resources, image/startOptimized, hostname, DB source, PDB,
      CNPG block).

## C. Pre-flight on the production cluster

- [ ] Namespace `keycloak` created.
- [ ] Secrets present: `registry-creds`, `keycloak-bootstrap-admin`,
      `keycloak-db`, `keycloak-db-ca`.
- [ ] Keycloak operator installed from `operators/keycloak-operator/` (image
      rewritten to the registry), SA patched with the pull secret, operator
      pod Running. **No CNPG on prod.**
- [ ] Connectivity proven **before** applying the CR — from a throwaway pod in
      the namespace, TLS to the DB endpoint with the CA:
      `psql "host=<ENDPOINT> dbname=keycloak user=keycloak sslmode=verify-full sslrootcert=/ca.crt"`
      — catches wrong endpoint / wrong CA / firewall in seconds instead of
      in a restart loop.
- [ ] Database `keycloak` is **empty** (fresh schema) — Keycloak creates it.
      Never point prod at a DB another Keycloak version has touched.

## D. Deploy

- [ ] `kubectl apply -f production.yaml`
- [ ] `kubectl -n keycloak wait --for=condition=Ready keycloak/keycloak --timeout=15m`
- [ ] First pod logs show *"Initializing database schema"* then
      *"started in"* **in seconds-to-low-minutes** (schema creation on a cold
      DB is the only slow part; the server start itself must be fast).

## E. Validate (all must pass)

- [ ] 3 pods `1/1`, **3 different nodes** (`get pods -o wide`).
- [ ] `get pdb keycloak` → ALLOWED DISRUPTIONS **1**.
- [ ] `Received new cluster view … (3)` in a pod log — one cluster of three.
- [ ] No TLS errors; truststore loaded; DB pool up.
- [ ] `.well-known/openid-configuration` issuer = `https://auth.<CLIENT-DOMAIN>/realms/master`.
- [ ] Restart drill: `kubectl delete pod keycloak-1` → back `1/1` in < 90 s,
      cluster view returns to (3). Proves the optimized image and that a
      node loss is a blip, not an outage.
- [ ] Metrics scraped by the client's Prometheus; login-failure alert wired.

## F. Hand-over hygiene

- [ ] Log in once as the bootstrap admin → create named admin accounts →
      disable/rotate the bootstrap user. Store credentials in the client
      vault.
- [ ] Kong: only the public OIDC paths routed; `/admin` and `/realms/master`
      return 403 through Kong (test it).
- [ ] `production.yaml` committed exactly as applied (minus secrets).

---

## Staging vs production — what does NOT carry over

| Setting | Staging (interim, validated 2026-08-19) | Production (design) |
|---|---|---|
| Image | stock `26.5.2`, `startOptimized: false` | **`26.5.2-optimized`, `startOptimized: true`** |
| Startup probe | 5 s × 240 (covers in-pod build) | operator default |
| CPU limit | 2 (build headroom) | 3 (design) |
| Instances / PDB | 2 / `minAvailable: 1` | **3 / `minAvailable: 2`** |
| PostgreSQL | CNPG in-cluster, 1 instance | client's external HA cluster |
| Superuser access | enabled (rehearsal needs it) | n/a — not our DB |

If any production item is tempted toward the staging column "just to get it
up", stop: that is the restart loop and half-applied schema from staging
waiting to happen on a system users depend on.

## Note on what staging could and could not prove

Staging PostgreSQL runs on the cluster's only storage — an S3/FUSE CSI — so
staging validated **behaviour** (clustering, DB TLS, probes, realm import,
login flows, operator lifecycle) but **not** throughput or timing. The k6
load test (2× design peak) and the migration-import timing must be measured
against a real PostgreSQL — ideally a staging database on the client's
VM + NFS PG stack, which is also the most faithful rehearsal of production.
Do not quote any staging-cluster performance numbers as evidence for the
production sizing.
