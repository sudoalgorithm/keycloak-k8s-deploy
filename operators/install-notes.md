# Operator Install & Deploy Sequence

Everything here is `kubectl` against pinned, official upstream manifests — no
Helm, no Kustomize. Run the steps yourself in order; each step says which
cluster it applies to.

> **Version pins.** Keycloak operator `26.5.2`, CNPG `1.25.1`. Before a fresh
> install, check for newer patch releases of the same minor
> (https://github.com/keycloak/keycloak/releases,
> https://github.com/cloudnative-pg/cloudnative-pg/releases) and bump the pin
> — staging and production must always use the **same** Keycloak version.

## 1. Keycloak operator — BOTH clusters

The operator manifests are namespaced; the namespace must exist first.

```bash
KC_VER=26.5.2

kubectl create namespace keycloak

# CRDs (cluster-scoped)
kubectl apply -f "https://raw.githubusercontent.com/keycloak/keycloak-k8s-resources/${KC_VER}/kubernetes/keycloaks.k8s.keycloak.org-v1.yml"
kubectl apply -f "https://raw.githubusercontent.com/keycloak/keycloak-k8s-resources/${KC_VER}/kubernetes/keycloakrealmimports.k8s.keycloak.org-v1.yml"

# the operator itself (into the keycloak namespace)
kubectl -n keycloak apply -f "https://raw.githubusercontent.com/keycloak/keycloak-k8s-resources/${KC_VER}/kubernetes/kubernetes.yml"

kubectl -n keycloak rollout status deployment/keycloak-operator
```

## 2. CNPG operator — STAGING cluster only

```bash
kubectl apply --server-side -f \
  "https://raw.githubusercontent.com/cloudnative-pg/cloudnative-pg/release-1.25/releases/cnpg-1.25.1.yaml"
kubectl -n cnpg-system rollout status deployment/cnpg-controller-manager
```

## 3. Secrets — created by hand, never committed

Both clusters — the bootstrap admin (pick a strong password, store it in the
team vault):

```bash
kubectl -n keycloak create secret generic keycloak-bootstrap-admin \
  --from-literal=username=admin \
  --from-literal=password='<CHOOSE-STRONG-PASSWORD>'
```

Production cluster only — credentials for the `keycloak` database owner on the
client's external PostgreSQL (client provides these; the database and role
must exist per the parent spec §2.3):

```bash
kubectl -n keycloak create secret generic keycloak-db \
  --from-literal=username=keycloak \
  --from-literal=password='<CLIENT-PROVIDED>'
```

Production cluster only — the CA certificate of the external PostgreSQL (PEM
file from the DBA team; Keycloak verifies the DB connection against it via
`db-tls-mode: verify-server`):

```bash
kubectl -n keycloak create secret generic keycloak-db-ca \
  --from-file=ca.crt=<PATH-TO-CLIENT-PG-CA>.pem
```

If the client's PostgreSQL turns out not to serve TLS, deploying without it
means removing `db-tls-mode` and the `truststores` block from
`production.yaml` — record that as an accepted risk with the client, don't do
it silently.

Staging needs neither — CNPG generates `keycloak-pg-app` (credentials) and
`keycloak-pg-ca` (CA) automatically.

## 4. Deploy staging

Replace `<CLIENT-DOMAIN>` in `staging.yaml`, then:

```bash
kubectl apply -f staging.yaml

# PostgreSQL first (the Keycloak CR just retries until the DB is up):
kubectl -n keycloak get cluster keycloak-pg -w        # wait: "Cluster in healthy state"

# then Keycloak:
kubectl -n keycloak wait --for=condition=Ready keycloak/keycloak --timeout=15m
```

Validation checklist:

```bash
# 3 pods, spread across 3 different nodes (operator default anti-affinity):
kubectl -n keycloak get pods -o wide

# PDB active, ALLOWED DISRUPTIONS = 1:
kubectl -n keycloak get pdb keycloak

# DB connection is TLS-verified (look for no TLS/SSL errors at startup and
# structured JSON log lines confirming the pool is up):
kubectl -n keycloak logs statefulset/keycloak --tail=50 | grep -i -E 'ssl|tls|pool' || true

# token endpoint answers (through a port-forward; realm exists after the
# migration rehearsal imports it — until then use the master realm's
# well-known endpoint as the liveness check):
kubectl -n keycloak port-forward svc/keycloak-service 8080:8080 &
curl -s http://localhost:8080/realms/master/.well-known/openid-configuration | head -c 200; echo
kill %1
```

## 5. Deploy production (after staging validates)

Replace `<CLIENT-DOMAIN>` and `<CLIENT-PG-RW-ENDPOINT>` in
`production.yaml`, confirm step 3's secrets exist on the prod cluster,
then:

```bash
kubectl apply -f production.yaml
kubectl -n keycloak wait --for=condition=Ready keycloak/keycloak --timeout=15m
kubectl -n keycloak get pods -o wide   # 3 pods on 3 nodes
```

Kong is pointed at `keycloak-service.keycloak.svc:8080` (how Kong reaches
cluster services is open item #2 in the design spec — resolve with the client
before cutover).

## Keeping the two files in sync

`staging.yaml` and `production.yaml` are hand-maintained parallels. After any
edit (e.g. a Keycloak version bump), diff them and confirm the only deltas are
the intended ones (DB source, hostname, resources, CNPG block):

```bash
diff staging.yaml production.yaml
```
