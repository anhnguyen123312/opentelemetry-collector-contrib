# Stage 1: Clone the OpenTelemetry Collector Contrib repository
FROM alpine:3.19 AS git-clone
RUN apk --update add ca-certificates git curl bash

WORKDIR /src
# Clone the DucHungGithub repository containing the StarRocks exporter
RUN git clone --branch main https://github.com/DucHungGithub/opentelemetry-collector-contrib.git

# Stage 2: Build the custom collector using OCB
FROM golang:1.24 AS build-stage
WORKDIR /build

# Copy the cloned repository from Stage 1
COPY --from=git-clone /src/opentelemetry-collector-contrib ./opentelemetry-collector-contrib

# Copy builder configuration
COPY ./builder-config.yaml ./builder-config.yaml

# Install OpenTelemetry Collector Builder (OCB)
RUN --mount=type=cache,target=/root/.cache/go-build \
    --mount=type=cache,target=/go/pkg/mod \
    GO111MODULE=on go install go.opentelemetry.io/collector/cmd/builder@v0.142.0

# Build the custom collector with StarRocks exporter
RUN --mount=type=cache,target=/root/.cache/go-build \
    --mount=type=cache,target=/go/pkg/mod \
    builder --config builder-config.yaml

# Stage 3: Production runtime image
FROM gcr.io/distroless/base:latest

ARG USER_UID=10001
USER ${USER_UID}

# Copy collector configuration
COPY ./collector-config.yaml /otelcol/collector-config.yaml

# Copy CA certificates from git-clone stage
COPY --from=git-clone /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/ca-certificates.crt

# Copy the built collector binary
COPY --chmod=755 --from=build-stage /build/otelcol-dev/otelcol-dev /otelcol/otelcol-dev

ENTRYPOINT ["/otelcol/otelcol-dev"]
CMD ["--config", "/otelcol/collector-config.yaml"]

# Expose ports for OTLP receivers
# 4317: gRPC
# 4318: HTTP
# 12001: StarRocks custom port (if needed)
EXPOSE 4317 4318 12001
