# ---- shared Go builder: generates proto stubs and builds every binary ----
FROM golang:1.25 AS builder
WORKDIR /src

# Make module downloads resilient to flaky corporate networks: retry via the
# public proxy, then fall straight to source if the proxy is unreachable, and
# don't fail the whole build just because sum.golang.org can't be reached.
ENV GOPROXY=https://proxy.golang.org,direct
ENV GOSUMDB=off
ENV GOFLAGS=-mod=mod
# Let Go fetch a newer toolchain if a dependency demands one.
ENV GOTOOLCHAIN=auto

RUN apt-get update && apt-get install -y --no-install-recommends protobuf-compiler \
    && rm -rf /var/lib/apt/lists/*
RUN go install google.golang.org/protobuf/cmd/protoc-gen-go@v1.35.1 \
 && go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@v1.5.1
COPY . .
RUN bash scripts/genproto.sh
RUN go mod tidy
RUN CGO_ENABLED=0 go build -o /out/gateway  ./cmd/gateway  \
 && CGO_ENABLED=0 go build -o /out/city     ./cmd/city     \
 && CGO_ENABLED=0 go build -o /out/places   ./cmd/places   \
 && CGO_ENABLED=0 go build -o /out/route    ./cmd/route    \
 && CGO_ENABLED=0 go build -o /out/favorite ./cmd/favorite

FROM gcr.io/distroless/static-debian12 AS gateway
COPY --from=builder /out/gateway /app
COPY static /static
ENTRYPOINT ["/app"]

FROM gcr.io/distroless/static-debian12 AS city
COPY --from=builder /out/city /app
ENTRYPOINT ["/app"]

FROM gcr.io/distroless/static-debian12 AS places
COPY --from=builder /out/places /app
ENTRYPOINT ["/app"]

FROM gcr.io/distroless/static-debian12 AS route
COPY --from=builder /out/route /app
ENTRYPOINT ["/app"]

FROM gcr.io/distroless/static-debian12 AS favorite
COPY --from=builder /out/favorite /app
ENTRYPOINT ["/app"]

# ---- combined stage used by managed hosting ----------------------------
# Render always builds the LAST stage of a Dockerfile and cannot select a
# target, so this stage carries every binary and each service picks its own
# via dockerCommand in render.yaml. Docker Compose still uses the named
# targets above, so local behaviour is unchanged.
FROM gcr.io/distroless/static-debian12 AS all
COPY --from=builder /out/gateway  /bin/gateway
COPY --from=builder /out/city     /bin/city
COPY --from=builder /out/places   /bin/places
COPY --from=builder /out/route    /bin/route
COPY --from=builder /out/favorite /bin/favorite
COPY static /static
ENV STATIC_DIR=/static
