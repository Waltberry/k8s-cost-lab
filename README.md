# k8s-cost-lab

OpenCost on Kind + kube-prometheus-stack + Kyverno label enforcement + sample microapp.

## What this shows
- Ownership/cost-center labels enforced on Namespaces/Deployments/Services (Kyverno)
- Prometheus/Grafana (kube-prometheus-stack)
- OpenCost cost allocation wired to Prometheus
- 3-service microapp (web/api/worker) across dev/staging/prod overlays (Kustomize)

## Quickstart
1. `kind create cluster --config kind-config.yaml`
2. `helm upgrade --install kube-prometheus-stack ...`
3. `helm upgrade --install opencost ...`
4. `kubectl apply -k ./` (namespaces)
5. `kubectl apply -k microapp/overlays/dev`
6. Port-forward Grafana: `kubectl -n monitoring port-forward svc/kube-prometheus-stack-grafana 3000:80`
7. Port-forward OpenCost: `kubectl -n opencost port-forward svc/opencost 9003:9003`
