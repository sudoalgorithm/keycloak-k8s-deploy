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
each cluster (both YAML files already reference it as `registry-creds`):

```bash
kubectl -n keycloak create secret docker-registry registry-creds \
  --docker-server=<REGISTRY-HOST> --docker-username=<USER> --docker-password=<PASS>
# same again in cnpg-system on the staging cluster after step 2 creates it,
# then: kubectl -n cnpg-system patch serviceaccount cnpg-manager \
#         -p '{"imagePullSecrets":[{"name":"registry-creds"}]}'
#       kubectl -n keycloak patch serviceaccount keycloak-operator \
#         -p '{"imagePullSecrets":[{"name":"registry-creds"}]}'
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
   blocks in both YAML files (`registry-creds`) are live, and the pull-secret +
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
`keycloak-pg-2`); it never adopts a pre-created PVC. A hand-made PVC (e.g.
`keycloak-pg-pvc`) just sits unused — delete it, and if its PV is `Released`,
free it: `kubectl patch pv <pv> -p '{"spec":{"claimRef":null}}'`.

With no dynamic provisioner (bare kubeadm), pre-provision a PV and pin CNPG's
PVC to it — this is what staging runs:

```yaml
  storage:
    size: 50Gi
    pvcTemplate:
      storageClassName: ""          # static binding, no provisioner
      volumeName: keycloak-pg1-pv   # your PV's name
      resources: {requests: {storage: 50Gi}}
```

- PV: ≥ the requested size, `ReadWriteOnce`, `Retain`; backing directory
  owned by the PostgreSQL UID/GID (`26:26` in the CNPG image — `chown -R
  26:26 <dir> && chmod 700 <dir>`), or set `postgresUID`/`postgresGID` to
  what the storage enforces.
- `volumeName` pins **one** PVC to **one** PV, so this form is single-instance
  only. For a replica (`instances: 2`) use a shared `storageClassName` on two
  PVs (different nodes for `local` volumes) instead of `volumeName`.

> **Known limitation — staging PG is on S3/FUSE, by necessity.** The staging
> cluster's only storage driver is `ru.yandex.s3.csi` (geesefs); there is no
> block or file class. PostgreSQL runs on it but: every random write is an
> object round-trip (schema creation ~13 min instead of < 1), and S3 cannot
> give fsync/atomic-rename semantics, so a hard crash may need the
> wipe-and-recreate in "Lessons" §2. **Accepted for staging** — it validates
> behaviour (clustering, TLS, probes, realm import, login flows). It does
> **not** validate load-test numbers or migration import timings; measure
> those against a real PostgreSQL (ideally a staging database on the client's
> VM/NFS PG stack — the most faithful rehearsal of production anyway).
> Production is unaffected: its PostgreSQL is external (VMs + NFS), and prod
> Keycloak pods need no PVs at all.

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

# the operator itself, image rewritten to the internal registry — both its
# own image AND RELATED_IMAGE_KEYCLOAK (the default server image it would use
# for anything not pinned by spec.image), so nothing can ever reference quay.io
sed -e "s|quay.io/keycloak/keycloak-operator|$REGISTRY/keycloak/keycloak-operator|g" \
    -e "s|quay.io/keycloak/keycloak:|$REGISTRY/keycloak/keycloak:|g" \
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

## Lessons from the first staging deploy (2026-08-19)

Recorded so the next person recognises the symptoms in seconds.

**1. Stock image + `startOptimized: false` restart-loops under CPU limits.**
Log stops at *"Updating the configuration and installing your custom
providers… Please wait"*, exit code 143, restarts climb into the hundreds.
The in-pod `kc.sh build` is starved by a 1-CPU limit and the startup probe
kills it first. No-image-build workaround (what staging runs today):

```bash
kubectl -n keycloak patch keycloak keycloak --type merge -p '{
  "spec": {"startupProbe": {"periodSeconds": 5, "failureThreshold": 240},
           "resources": {"limits": {"cpu": "2", "memory": "1500Mi"},
                         "requests": {"cpu": "500m", "memory": "1200Mi"}}}}'
```

Real fix: the pre-optimized image (step 0b). Every pod start then takes
seconds instead of minutes — required before production.

**2. A Keycloak killed mid-first-start leaves a half-applied schema.** Next
start fails with a Liquibase error like *`column "user_setup_allowed" of
relation "authentication_execution" does not exist`* at changeset 1.5.0. The
changelog tracker and the real tables disagree. On a **fresh, empty**
staging DB (never on one holding migrated data) wipe and let Keycloak create
it cleanly:

```bash
kubectl -n keycloak exec -it keycloak-pg-1 -- psql -U postgres -d keycloak -c \
  "DROP SCHEMA public CASCADE; CREATE SCHEMA public; ALTER SCHEMA public OWNER TO keycloak; GRANT ALL ON SCHEMA public TO keycloak;"
kubectl -n keycloak delete pod keycloak-0
```

Healthy first start then logs *"Initializing database schema"* (not
*"Updating"*), and on a cold DB takes ~15 min end to end — the table count
(`SELECT count(*) FROM information_schema.tables WHERE table_schema='public'`)
climbing toward ~95 shows it is progressing.

**3. PDB must track the instance count.** `instances: 2` with
`minAvailable: 2` means zero allowed disruptions — drains block forever.
Staging runs 2/1, production 3/2.

**4. Validation that passed** (the checklist in step 4): pods on distinct
nodes, PDB allows 1 disruption, `Received new cluster view … (2)` in the
logs (one Infinispan cluster, not two singletons), DB TLS `verify-server`
working, `.well-known` issuer equals the pinned hostname.

## Keeping the two files in sync

`staging.yaml` and `production.yaml` are hand-maintained parallels. After any
edit (e.g. a Keycloak version bump), diff them and confirm the only deltas are
the intended ones (DB source, hostname, resources, CNPG block):

```bash
diff staging.yaml production.yaml
```
