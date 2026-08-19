# Pre-optimized Keycloak image — REQUIRED for the air-gapped clusters.
#
# Why this exists: the stock image is un-augmented. With `startOptimized:
# false` Keycloak re-runs `kc.sh build` on EVERY pod start, before it opens
# the health port. On the staging nodes (1 CPU limit) that build takes
# minutes, the startup probe gives up, the kubelet kills the pod (exit 143),
# and the pod never becomes Ready. Baking the build into the image once moves
# that cost out of the start path: pods start in seconds.
#
# Builds entirely INSIDE the air gap — the base is the already-mirrored stock
# image in the internal registry. No internet involved.
#
#   docker build --build-arg REGISTRY=<REGISTRY> \
#     -t <REGISTRY>/keycloak/keycloak:26.5.2-optimized build/keycloak-optimized
#   docker push <REGISTRY>/keycloak/keycloak:26.5.2-optimized
#
# The Keycloak version MUST match the operator version on the clusters.
# Later, the email-or-phone authenticator JAR is added by dropping it into
# providers/ before the build — same Dockerfile, new tag.
ARG REGISTRY
FROM ${REGISTRY}/keycloak/keycloak:26.5.2 AS builder

# Build-time options baked in. Must stay consistent with what the Keycloak CR
# sets: postgres DB, health endpoints (operator probes on :9000), metrics.
ENV KC_DB=postgres \
    KC_HEALTH_ENABLED=true \
    KC_METRICS_ENABLED=true

# Optional custom providers (authenticator JAR) — empty today.
COPY providers/ /opt/keycloak/providers/

RUN /opt/keycloak/bin/kc.sh build

ARG REGISTRY
FROM ${REGISTRY}/keycloak/keycloak:26.5.2
COPY --from=builder /opt/keycloak/ /opt/keycloak/
ENTRYPOINT ["/opt/keycloak/bin/kc.sh"]
