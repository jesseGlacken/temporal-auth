# ---- Build stage -----------------------------------------------------------
FROM golang:1.26.10-bookworm AS builder

WORKDIR /src

# CGO is required; gcc/libc headers ship with the golang image.
ENV CGO_ENABLED=1

# Copy module files first for layer caching
COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN go build -trimpath -ldflags="-s -w" -o /out/temporal-auth-server main.go

# ---- Runtime stage ---------------------------------------------------------
# distroless base (glibc, no shell/package manager), runs as uid 65532.
FROM gcr.io/distroless/base-debian12:nonroot

WORKDIR /app

COPY --from=builder /out/temporal-auth-server /app/temporal-auth-server
# Templated config: contains no secrets; values come from env at startup.
# Docker Compose overrides this by mounting docker/temporal-config.yaml.
COPY --chown=nonroot:nonroot config/ /app/config/

USER nonroot:nonroot

ENTRYPOINT ["/app/temporal-auth-server"]
