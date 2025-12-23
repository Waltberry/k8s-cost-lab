# k8s-cost-lab

OpenCost on a local **Kind** cluster with **kube-prometheus-stack** (Prometheus + Grafana) and **Kyverno** label enforcement. Includes a tiny 3-service microapp and a separate **cost-report-api** service (FastAPI) that reads OpenCost.

## What this shows
- Enforced ownership/cost labels on **Namespaces/Deployments/Services** via Kyverno.
- **Prometheus/Grafana** via kube-prometheus-stack.
- **OpenCost** wired to Prometheus for cost allocation.
- 3-service microapp (web/api/worker) with **Kustomize** overlays (dev/staging/prod).
- Optional **FastAPI** proxy (`cost-report-api`) that exposes `/cost-report`.

## Repo layout
```

k8s-cost-lab/
├─ namespaces/                 # monitoring, opencost, platform, apps-* namespaces
├─ policies/kyverno/           # require owner/team/cost-center labels
├─ monitoring/                 # Helm values for kube-prom-stack + OpenCost
├─ microapp/
│  ├─ base/                    # web/api/worker Deployments + Services
│  └─ overlays/{dev,staging,prod}/
├─ kustomize/                  # (optional older scaffolding)
├─ README.md
└─ kustomization.yaml          # applies namespaces (base infra)

```

## Architecture (high level)
````

Kind cluster
│
├─ monitoring (kube-prom-stack)
│    ├─ Prometheus
│    └─ Grafana
├─ opencost
│    └─ OpenCost (reads Prometheus)
├─ platform
│    └─ cost-report-api (FastAPI)  <-- proxies OpenCost
└─ apps-dev/staging/prod
└─ microapp (web/api/worker)

````

## Quickstart

1) **Create Kind cluster** (example)
```bash
kind create cluster --config kind-config.yaml
```

2. **Install Kyverno + policy**

```bash
helm repo add kyverno https://kyverno.github.io/kyverno/
helm repo update
kubectl create ns kyverno
helm upgrade --install kyverno kyverno/kyverno -n kyverno
kubectl wait -n kyverno deploy --all --for=condition=available --timeout=120s
kubectl apply -f policies/kyverno/require-labels.yaml
```

3. **Namespaces**

```bash
kubectl apply -k .
kubectl get ns
```

4. **Install kube-prometheus-stack + OpenCost**

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add opencost https://opencost.github.io/opencost-helm-chart
helm repo update

kubectl create ns monitoring
helm upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  -n monitoring -f monitoring/values-kps.yaml
kubectl wait -n monitoring deploy --all --for=condition=available --timeout=300s

kubectl create ns opencost
helm upgrade --install opencost opencost/opencost \
  -n opencost -f monitoring/values-opencost.yaml
kubectl wait -n opencost deploy --all --for=condition=available --timeout=300s
```

5. **Deploy microapp (dev)**

```bash
kubectl apply -k microapp/overlays/dev
kubectl -n apps-dev get deploy,svc,pod
```

## Port-forwards (use separate terminals)

* **Grafana** → [http://localhost:3000](http://localhost:3000)

  ```powershell
  kubectl -n monitoring port-forward svc/kube-prometheus-stack-grafana 3000:80
  ```

  **Password** (Windows PowerShell):

  ```powershell
  $pwd64 = kubectl -n monitoring get secret kube-prometheus-stack-grafana -o jsonpath="{.data.admin-password}"
  [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($pwd64))
  ```

* **OpenCost API** → [http://localhost:9003](http://localhost:9003)

  ```powershell
  kubectl -n opencost port-forward svc/opencost 9003:9003
  ```

  Test:

  ```powershell
  curl.exe "http://localhost:9003/allocation/compute?window=24h&aggregate=namespace"
  ```

* **microapp frontend** → [http://localhost:8080](http://localhost:8080)

  ```powershell
  kubectl -n apps-dev port-forward svc/frontend 8080:80
  ```

* **cost-report-api** (if deployed in `platform`) → [http://localhost:8082](http://localhost:8082)

  ```powershell
  kubectl -n platform port-forward svc/cost-report-api 8082:8080
  curl.exe http://127.0.0.1:8082/health
  curl.exe "http://127.0.0.1:8082/cost-report?window=24h&aggregate=namespace"
  ```

## OpenCost ↔ Prometheus wiring (values excerpt)

```yaml
opencost:
  exporter:
    defaultModelPricing: true
    cloudProvider: generic
    prometheus:
      internal:
        enabled: true
        serviceName: kube-prometheus-stack-prometheus
        namespaceName: monitoring
        port: 9090
```

## Troubleshooting

* **Port already in use (e.g., 3000/9003/8082):**

  ```powershell
  Get-NetTCPConnection -LocalPort 3000 | Select-Object -ExpandProperty OwningProcess | %{ taskkill /PID $_ /F }
  ```

  Then re-run `port-forward`.
* **OpenCost not returning data:** make sure kube-prom-stack is up; check OpenCost logs
  `kubectl -n opencost logs deploy/opencost -c opencost --tail=200`
* **Kyverno blocking unlabeled objects:** add `owner/team/cost-center` labels to metadata, re-apply.
* **Private image pulls:** use public images or create an imagePullSecret and reference it in the Deployment.