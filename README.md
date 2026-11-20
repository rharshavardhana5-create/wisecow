# Wisecow on Kubernetes

Containerised and deployed the Wisecow app (a small Bash web server that serves
`fortune | cowsay` output on port 4499) to Kubernetes with an automated CI/CD
pipeline and TLS.

## What's included

| Path | Purpose |
|---|---|
| `wisecow.sh` | Application source |
| `Dockerfile` | Container image (Ubuntu 22.04 + fortune, cowsay, netcat) |
| `k8s/deployment.yaml` | Deployment, 2 replicas, image from GHCR |
| `k8s/service.yaml` | ClusterIP Service (80 -> 4499) |
| `k8s/ingress.yaml` | Traefik Ingress with TLS for `wisecow.local` |
| `.github/workflows/ci.yml` | Build, push and deploy pipeline |

## Run locally with Docker

```
docker build -t wisecow:local .
docker run --rm -p 8080:4499 wisecow:local
```
Open http://localhost:8080

## Deploy to Kubernetes (kind)

```
kind create cluster --name wisecow
kubectl apply -f k8s/deployment.yaml -f k8s/service.yaml
kubectl port-forward service/wisecow 9090:80
```
Open http://localhost:9090

## CI/CD (GitHub Actions)

On every push to `main` the workflow:
1. **build-and-push**: builds the image and pushes it to
   `ghcr.io/rharshavardhana5-create/wisecow` (tags: `latest` and the commit SHA).
2. **deploy**: creates a temporary kind cluster, applies `k8s/`, sets the new
   image, waits for the rollout and runs a smoke test with `curl`.

## TLS

TLS is terminated at a Traefik Ingress using a self-signed certificate stored
as the Kubernetes secret `wisecow-tls`. The key and certificate are never
committed (see `.gitignore`).

```
helm repo add traefik https://traefik.github.io/charts
helm install traefik traefik/traefik --namespace traefik --create-namespace
# create tls.crt / tls.key for wisecow.local, then:
kubectl create secret tls wisecow-tls --cert=tls.crt --key=tls.key
kubectl apply -f k8s/ingress.yaml
kubectl port-forward -n traefik service/traefik 8443:443
curl -k --resolve wisecow.local:8443:127.0.0.1 https://wisecow.local:8443
```
`-k` is needed only because the certificate is self-signed.

## Screenshots

(Add: green GitHub Actions run, deploy job log, cow served over HTTPS.)