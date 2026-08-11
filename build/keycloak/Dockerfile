# Custom Keycloak image: official base + the email-or-phone authenticator JAR.
#
# Air-gap friendly by construction: the base image is the ALREADY-MIRRORED
# stock image in the internal registry, so this builds on any machine inside
# the gap that can reach the registry — no internet involved.
#
#   docker build --build-arg REGISTRY=<REGISTRY> \
#     -t <REGISTRY>/keycloak/keycloak-custom:26.5.2-1 build/keycloak
#   docker push <REGISTRY>/keycloak/keycloak-custom:26.5.2-1
#
# Tag scheme: <keycloak-version>-<build-number>. The Keycloak version MUST
# match the operator version running on the clusters.
#
# After pushing, point spec.image at this tag in staging.yaml/production.yaml
# and flip startOptimized to true — this image IS augmented (kc.sh build below
# bakes the build-time options), so pods skip re-augmentation at startup.
ARG REGISTRY
FROM ${REGISTRY}/keycloak/keycloak:26.5.2 AS builder

# Build-time options baked into the optimized image. Keep in sync with what
# the CRs expect: postgres driver, health endpoints (operator probes),
# metrics endpoint.
ENV KC_DB=postgres \
    KC_HEALTH_ENABLED=true \
    KC_METRICS_ENABLED=true

# The authenticator JAR (built offline with Maven, copied here before build).
COPY providers/ /opt/keycloak/providers/

RUN /opt/keycloak/bin/kc.sh build

ARG REGISTRY
FROM ${REGISTRY}/keycloak/keycloak:26.5.2
COPY --from=builder /opt/keycloak/ /opt/keycloak/
ENTRYPOINT ["/opt/keycloak/bin/kc.sh"]
