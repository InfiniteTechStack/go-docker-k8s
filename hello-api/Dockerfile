# -------- Stage 1: Build --------
FROM golang:1.26-alpine AS builder

WORKDIR /app

# Enable go modules
COPY go.mod ./
RUN go mod download

# Copy source code
COPY . .

# Build static binary
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o app

# -------- Stage 2: Runtime (Distroless) --------
FROM gcr.io/distroless/base-debian12

WORKDIR /app

# Copy binary from builder
COPY --from=builder /app/app .

# Use non-root user (best practice for EKS/K8s)
USER nonroot:nonroot

EXPOSE 8080

ENTRYPOINT ["/app/app"]
