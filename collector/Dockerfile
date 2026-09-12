# Builds an OpenTelemetry Collector that ships the Hippocampus logs exporter, assembled with the
# OpenTelemetry Collector Builder (OCB) from builder-config.yaml. The build context must be the
# repository root, because the builder manifest's `replaces:` points at the sibling exporter module
# and it must be present in the context. Build from the repo root with:
#
#   docker build -f collector/Dockerfile -t hippocampus-otel-collector .

# Build on the native BUILDPLATFORM and cross-compile the collector to TARGETOS/TARGETARCH (CGO off),
# so a multi-arch build never emulates the OCB toolchain under QEMU. TARGETOS/TARGETARCH come from
# buildx and must be re-declared as ARGs in this stage to be referenced. OCB itself (go install
# below) is left native to the build host - only the collector it compiles is cross-targeted.
FROM --platform=$BUILDPLATFORM golang:1.27-alpine AS build

ARG TARGETOS
ARG TARGETARCH

# Must track otelcol_version in builder-config.yaml.
ARG OCB_VERSION=0.160.0

# git: the go toolchain needs it to resolve some collector-contrib module dependencies. The git
# version is left unpinned to track the alpine repo, matching the unpinned tooling used elsewhere.
# hadolint ignore=DL3018
RUN apk add --no-cache git \
    && go install go.opentelemetry.io/collector/cmd/builder@v${OCB_VERSION}

# The release tag, used both to stamp the binary below and to label the image in the next stage.
# Declared here as well because an ARG is scoped to the stage that declares it.
ARG VERSION=dev

WORKDIR /src
COPY . .

# OCB generates sources under _build/ and compiles them; the relative replace in the manifest
# resolves against that generated go.mod, so it must run from the collector directory. CGO is off so
# the binary runs on a minimal base image with no C library.
#
# dist.version is what `hippocampus-otelcol --version` and its `components` buildinfo report, and it
# is stamped from the release tag rather than committed. Left committed it was 0.0.1, so every image
# ever published - at the service's version, on the service's tag - shipped a collector that
# reported 0.0.1 when asked. Stamping it here means there is no second version to remember to bump.
# The leading v is stripped: the field is semver, and the tags are vX.Y.Z.
WORKDIR /src/collector
RUN version="${VERSION#v}" \
    && sed -i "s|^  version: .*|  version: ${version}|" builder-config.yaml \
    && CGO_ENABLED=0 GOOS=${TARGETOS} GOARCH=${TARGETARCH} builder --config builder-config.yaml

FROM alpine:3.24 AS otel-collector

# checkov:skip=CKV_DOCKER_2: the bundled config enables no health_check extension, so there is no
# endpoint to probe; liveness is left to the orchestrator (mirrors the root Dockerfile's mcp stage).
ARG VERSION=dev
LABEL org.opencontainers.image.title="hippocampus-otel-collector" \
    org.opencontainers.image.version="${VERSION}" \
    org.opencontainers.image.source="https://github.com/fastbean-au/hippocampus-otel-collector"

RUN adduser -D -H -u 1000 otel

COPY --from=build /src/collector/_build/hippocampus-otelcol /usr/local/bin/hippocampus-otelcol
# The bundled config tails sample.log and ships to a local Hippocampus - a runnable demo default.
# Mount your own config over /etc/otelcol/config.yaml (and adjust the CMD) for a real deployment.
COPY collector/config.yaml collector/sample.log /etc/otelcol/

USER otel

# 4317 OTLP gRPC (the bundled config opens an otlp receiver alongside the filelog tail).
EXPOSE 4317

WORKDIR /etc/otelcol
ENTRYPOINT ["hippocampus-otelcol"]
CMD ["--config", "/etc/otelcol/config.yaml"]
