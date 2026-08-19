# Operator Install & Deploy Sequence (Air-Gapped)

The client clusters have **no internet access**. Everything they need is
either committed in this repo (operator manifests, vendored under
`operators/`) or mirrored into the client's internal container registry ahead
of time. Nothing is fetched from the internet at install time. Run the steps
yourself in order; each step says where it runs (connected machine, staging
cluster, production cluster).

> **Version pins.** Keycloak operator `26.5.2`, CNPG `1.25.1`. Bumping a
> version now means, on a connected machine: re-download the manifests into
> `operators/`, re-mirror the images (step 0), commit — then roll staging
> first. Staging and production must always run the **same** Keycloak version,
> and `spec.image` in both YAML files must match the operator version exactly.

## 0. Prepare the air-gap bundle — CONNECTED machine

**Shortcut: a prebuilt bundle is attached to the repo's Releases page**
(tag `images-26.5.2`) — four gzipped image archives + `SHA256SUMS`, all
`linux/amd64`:

```bash
gh release download images-26.5.2 --dir bundle
# or download the assets from
# https://github.com/sudoalgorithm/keycloak-k8s-deploy/releases/tag/images-26.5.2
shasum -a 256 -c bundle/SHA256SUMS   # verify before AND after crossing the gap
```

With the bundle downloaded, skip straight to the transfer/load/push part
below. The pull/save commands that follow are how the bundle is
**regenerated** — needed on a version bump, or if you prefer not to trust a
prebuilt artifact.

Four images make the whole system run:

| Image | Used by |
|---|---|
| `quay.io/keycloak/keycloak-operator:26.5.2` | operator Deployment (both clusters) |
| `quay.io/keycloak/keycloak:26.5.2` | Keycloak pods + realm-import jobs (both clusters) |
| `ghcr.io/cloudnative-pg/cloudnative-pg:1.25.1` | CNPG operator (staging cluster) |
| `ghcr.io/cloudnative-pg/postgresql:16` | staging PostgreSQL pods |

Pull for the **cluster's architecture** — on an Apple-silicon laptop a plain
`docker pull` grabs arm64 and the pods will crash-loop on the x86 cluster:

```bash
mkdir -p bundle
for IMG in \
  quay.io/keycloak/keycloak-operator:26.5.2 \
  quay.io/keycloak/keycloak:26.5.2 \
  ghcr.io/cloudnative-pg/cloudnative-pg:1.25.1 \
  ghcr.io/cloudnative-pg/postgresql:16; do
  docker pull --platform linux/amd64 "$IMG"
  docker save -o "bundle/$(echo "$IMG" | tr '/:' '__').tar" "$IMG"
done
```

Transfer `bundle/` (plus a clone of this repo) across the air gap by the
client's approved method, then load and push into the internal registry from a
machine that reaches it:

```bash
REGISTRY=<REGISTRY>       # the client's internal registry, e.g. registry.internal:5000

for TAR in bundle/*.tar; do docker load -i "$TAR"; done

for IMG in \
  keycloak/keycloak-operator:26.5.2 \
  keycloak/keycloak:26.5.2 \
  cloudnative-pg/cloudnative-pg:1.25.1 \
  cloudnative-pg/postgresql:16; do
  SRC=$IMG
  case $IMG in keycloak/*) SRC="quay.io/$IMG";; *) SRC="ghcr.io/$IMG";; esac
  docker tag "$SRC" "$REGISTRY/$IMG"
  docker push "$REGISTRY/$IMG"
done
```

The path layout in the registry mirrors upstream minus the host
(`<REGISTRY>/keycloak/keycloak:26.5.2` etc.) — the YAML files and the `sed`
substitutions below assume exactly this layout.

> **Why images are mirrored, not built.** There are deliberately no
> Dockerfiles in this repo: a `docker build` would itself pull a base image
> from the internet, and hand-built images lose the official release's
> testing and signing. The air gap is crossed by *transferring* the official
> artifacts through the approved channel — the same trust model as the vendor
> dump.

If the registry requires authentication, additionally create a pull secret on
each cluster (and uncomment the `imagePullSecrets` blocks in the YAML files):

```bash
kubectl -n keycloak create secret docker-registry registry-credentials \
  --docker-server=<REGISTRY-HOST> --docker-username=<USER> --docker-password=<PASS>
# same again in cnpg-system on the staging cluster after step 2 creates it,
# then: kubectl -n cnpg-system patch serviceaccount cnpg-manager \
#         -p '{"imagePullSecrets":[{"name":"registry-credentials"}]}'
#       kubectl -n keycloak patch serviceaccount keycloak-operator \
#         -p '{"imagePullSecrets":[{"name":"registry-credentials"}]}'
```

### The client registry is GitLab — specifics

The internal registry is the **GitLab container registry**, which changes two
assumptions:

1. **Paths are project-scoped.** Every image must live under a GitLab
   group/project path. Set the placeholder to the full prefix and nothing
   else changes:

   ```
   REGISTRY=registry.<gitlab-host>/<group>/<project>   # e.g. .../infra/keycloak-images
   ```

   Final refs look like
   `registry.<gitlab-host>/infra/keycloak-images/keycloak/keycloak:26.5.2`
   (GitLab allows extra path levels below the project). Ask the GitLab admin
   to create/designate the project.

2. **Auth is mandatory** (no anonymous pulls) — so the `imagePullSecrets`
   blocks in both YAML files must be uncommented, and the pull-secret +
   service-account patches above are required, not optional. Tokens needed
   from the admin:
   - `write_registry` token — for `docker login` on the machine that pushes
     the bundle;
   - **deploy token** with `read_registry` — for the clusters' pull secret.

   `--docker-server` takes the registry **host only** (no group/project
   path).

3. **Disable the cleanup policy** on that project's container registry —
   GitLab cleanup policies auto-delete tags on a schedule and would silently
   remove pinned images.

## 0b. Build the pre-optimized Keycloak image — machine INSIDE the gap with registry access

**Required, not optional.** The stock image re-runs `kc.sh build` on every
pod start; under the pod CPU limits that takes minutes, the startup probe
gives up and the pod restart-loops forever (symptom: log stops at *"Updating
the configuration and installing your custom providers… Please wait"*, exit
code 143, hundreds of restarts). Baking the build in once fixes it:

```bash
REGISTRY=<REGISTRY>
docker build --build-arg REGISTRY=$REGISTRY \
  -t $REGISTRY/keycloak/keycloak:26.5.2-optimized build/keycloak-optimized
docker push $REGISTRY/keycloak/keycloak:26.5.2-optimized
```

Both YAML files reference this `-optimized` tag with `startOptimized: true`.
The base is the mirrored stock image from step 0, so this needs no internet.

## 0c. Storage for the staging PostgreSQL — read before applying

CloudNativePG **creates its own PVCs**, named `<cluster>-<n>` (`keycloak-pg-1`,
`keycloak-pg-2`); it never adopts a pre-created PVC. If the cluster has no
default StorageClass with dynamic provisioning (typical on bare kubeadm), you
pre-provision **two** PVs (one per instance; for `local` volumes on two
different nodes) and let CNPG's PVCs bind to them:

- each PV: ≥ 20Gi, `ReadWriteOnce`, `storageClassName: keycloak-pg`
  (any name, used consistently), `Retain`;
- the backing directory owned by the PostgreSQL UID/GID — default `26:26`
  in the CNPG image (`chown -R 26:26 <dir> && chmod 700 <dir>`), or set
  `postgresUID`/`postgresGID` in the CNPG spec to whatever the storage
  enforces;
- then uncomment `storageClass: keycloak-pg` in `staging.yaml`.

A hand-made PVC (e.g. `keycloak-pg-pvc`) just sits unused — delete it, and
if its PV is `Released`, free it: `kubectl patch pv <pv> -p
'{"spec":{"claimRef":null}}'`.

## 1. Keycloak operator — BOTH clusters

Applied from the vendored copies in `operators/keycloak-operator/` (downloaded
from `keycloak-k8s-resources` tag `26.5.2`, committed unmodified). The only
change made at apply time is rewriting the operator image to the internal
registry. The namespace must exist first.

```bash
REGISTRY=<REGISTRY>

kubectl create namespace keycloak

# CRDs (cluster-scoped, no images inside)
kubectl apply -f operators/keycloak-operator/keycloaks.k8s.keycloak.org-v1.yml
kubectl apply -f operators/keycloak-operator/keycloakrealmimports.k8s.keycloak.org-v1.yml

# the operator itself, image rewritten to the internal registry
sed "s|quay.io/keycloak/keycloak-operator|$REGISTRY/keycloak/keycloak-operator|g" \
  operators/keycloak-operator/kubernetes.yml | kubectl -n keycloak apply -f -

kubectl -n keycloak rollout status deployment/keycloak-operator
```

## 2. CNPG operator — STAGING cluster only

Applied from the vendored copy in `operators/cnpg/`, image rewritten the same
way:

```bash
sed "s|ghcr.io/cloudnative-pg/cloudnative-pg|$REGISTRY/cloudnative-pg/cloudnative-pg|g" \
  operators/cnpg/cnpg-1.25.1.yaml | kubectl apply --server-side -f -
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

Replace `<CLIENT-DOMAIN>` and `<REGISTRY>` in `staging.yaml`, then:

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

Replace `<CLIENT-DOMAIN>`, `<REGISTRY>`, and `<CLIENT-PG-RW-ENDPOINT>` in
`production.yaml`, confirm step 0's images are in the registry reachable from
the prod cluster and step 3's secrets exist there, then:

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
