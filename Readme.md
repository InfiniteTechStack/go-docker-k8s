# go-app-docker-k8s

- A Go-based Hello API application, containerized with Docker, deployed to a local Kubernetes (kind) cluster using Kustomize.
  
## This project features

- Multi-architecture Docker builds (amd64 + arm64)
- Local testing with docker
- Multi-stage Docker builds for small, secure images
- Kubernetes deployment using Kustomize overlays
- Local testing with kind cluster and port-forwarding

## 1.Dockerfile

- Created a Dockerfile to build the Go-based hello-api application.
- Used a multi-stage build to ensure a small and secure image.
- Dockerfile Path: ./hello-api/Dockerfile

```Dockerfile
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
```

## 2. Build and Push Docker Image

```bash
## build the application
docker buildx build \
    --platform linux/amd64,linux/arm64 \
    -t devops022/hello-api:221fbf9 \
    ./hello-api

## docker login with token
docker login -u devops022

## push the image
docker push devops022/hello-api:221fbf9

## Test Locally:
docker run -itd --name hello-api -p 8080:8080 --platform linux/amd64 devops022/hello-api:221fbf9
curl -G 'http://localhost:8080?name=DevOps'
```

## DockerHub URL

- <https://hub.docker.com/r/devops022/hello-api/tags>

## 3. Kustomize Manifest

- Kustomize manifests are created to deploy the application.
- Base manifests are flexible, allowing developers to change image tags, replicas, resources, or environment variables without modifying the deployment YAML.

### kustomize directory structure

```bash
├── Readme.md
├── hello-api
│   ├── Dockerfile
│   ├── go.mod
│   └── main.go
└── k8s
    ├── base
    │   └── apps
    │       ├── hello-api
    │       │   ├── deployment.yaml
    │       │   ├── kustomization.yaml
    │       │   ├── namespace.yaml
    │       │   └── service.yaml
    │       └── kustomization.yaml
    └── overlays
        └── non-production
            └── test
                └── kustomization.yaml
```

## Create a kind cluster

```bash
brew intall kind # To install kind 
kind create cluster --name project --image kindest/node:v1.29.1
```

### Deploy to kind cluster

```bash
export DOCKERHUB_PAT=<pat-token>
kubectl create ns apps
kubectl create secret -n apps docker-registry image-pull-secret \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=devops022 \
  --docker-password=$DOCKERHUB_PAT
```

```bash
# Client-side dry-run
kustomize build k8s/overlays/non-production/test | kubectl apply -f - --dry-run=client

# Server-side dry-run
kustomize build k8s/overlays/non-production/test | kubectl apply -f - --dry-run=server

# Apply
kustomize build k8s/overlays/non-production/test | kubectl apply -f -
```

## Test the kubernetes app locally

```bash
kubectl port-forward deployment/hello-api 8081:8080 -n apps 
curl -G 'http://localhost:8081?name=DevOps' 
```

## Cleanup the project

```bash
kind delete cluster -n project 
docker stop hello-api:221fbf9
docker rmi devops022/hello-api:221fbf9 -f
```
