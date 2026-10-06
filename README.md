# Kubernetes Deployment Strategy Labs

These labs are Kubernetes-YAML driven and designed for Windows + Docker Desktop + WSL2 + Minikube.

## Prerequisites

- Docker Desktop running with WSL2 integration enabled
- WSL2 Ubuntu
- kubectl
- minikube

## Start cluster

```bash
minikube start --driver=docker
minikube addons enable ingress
kubectl apply -f 00-setup/namespace.yaml
kubectl get nodes
kubectl get pods -A
```

## Lab 1: Rolling Update

```bash
kubectl apply -f 01-rolling-update/deployment-v1.yaml
kubectl apply -f 01-rolling-update/service.yaml
kubectl get pods -n deployment-lab -w
```

Open a second terminal and upgrade:

```bash
kubectl apply -f 01-rolling-update/deployment-v2.yaml
kubectl rollout status deployment/demo-app -n deployment-lab
kubectl rollout history deployment/demo-app -n deployment-lab
```

Rollback:

```bash
kubectl rollout undo deployment/demo-app -n deployment-lab
kubectl rollout status deployment/demo-app -n deployment-lab
```

## Lab 2: Rolling Update Failure

First restore V1 if needed, then deliberately apply the bad image:

```bash
kubectl apply -f 01-rolling-update/deployment-v1.yaml
kubectl apply -f 01-rolling-update/service.yaml
kubectl apply -f 02-rolling-update-failure/bad-deployment.yaml
kubectl get pods -n deployment-lab -w
kubectl get rs -n deployment-lab
kubectl rollout history deployment/demo-app -n deployment-lab
```

Recover:

```bash
kubectl rollout undo deployment/demo-app -n deployment-lab
```

## Lab 3: Canary

```bash
kubectl apply -f 03-canary/stable-deployment.yaml
kubectl apply -f 03-canary/canary-deployment.yaml
kubectl apply -f 03-canary/stable-service.yaml
kubectl apply -f 03-canary/canary-service.yaml
kubectl apply -f 03-canary/stable-ingress.yaml
kubectl apply -f 03-canary/canary-ingress.yaml
kubectl get ingress -n deployment-lab
kubectl get pods -n deployment-lab -o wide
```

From another terminal:

```bash
kubectl port-forward -n ingress-nginx service/ingress-nginx-controller 8080:80
```

Then from Windows PowerShell:

```powershell
curl.exe -H "Host: demo.local" http://127.0.0.1:8080/
```

For repeated traffic in WSL2:

```bash
for i in {1..100}; do curl -s -H 'Host: demo.local' http://127.0.0.1:8080/ | grep -o 'V[12]-CANARY\|V1'; done
```

Change Canary weight from 10 to 30 by editing the annotation in `03-canary/canary-ingress.yaml`:

```yaml
nginx.ingress.kubernetes.io/canary-weight: "30"
```

Apply again:

```bash
kubectl apply -f 03-canary/canary-ingress.yaml
```

Progressively test 10 -> 30 -> 50 -> 100.

Rollback by setting:

```yaml
nginx.ingress.kubernetes.io/canary-weight: "0"
```

## Lab 4: Blue-Green

```bash
kubectl apply -f 04-blue-green/blue-deployment.yaml
kubectl apply -f 04-blue-green/green-deployment.yaml
kubectl apply -f 04-blue-green/blue-service.yaml
kubectl apply -f 04-blue-green/green-service.yaml
kubectl apply -f 04-blue-green/ingress-blue.yaml
kubectl get pods -n deployment-lab -o wide
```

Test blue:

```powershell
curl.exe -H "Host: bluegreen.local" http://127.0.0.1:8080/
```

Switch production traffic to green by changing the service name in `ingress-blue.yaml` from `demo-app-blue` to `demo-app-green`, then:

```bash
kubectl apply -f 04-blue-green/ingress-blue.yaml
```

Rollback by changing it back to `demo-app-blue` and applying again.

## Cleanup

```bash
kubectl delete namespace deployment-lab
minikube delete
```
