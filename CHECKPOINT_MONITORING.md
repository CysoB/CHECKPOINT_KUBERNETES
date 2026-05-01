# CHECKPOINT MONITORING — Prometheus + Grafana (kube-prometheus-stack)

Add the Helm repository:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

Create the namespace:

```bash
kubectl create namespace monitoring
```

Install the stack:

```bash
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring
```

Verify:

```bash
kubectl get pods -n monitoring
```

## 2) Access Grafana

Port-forward Grafana:

```bash
kubectl port-forward svc/monitoring-grafana -n monitoring 3000:80
```

Get the admin password:

```bash
kubectl get secret monitoring-grafana -n monitoring -o jsonpath="{.data.admin-password}" | base64 --decode
```

Sign in:
- URL: `http://localhost:3000`
- User: `admin`
- Password: output of the command above

## 3) Access Prometheus

```bash
kubectl port-forward svc/monitoring-kube-prometheus-prometheus -n monitoring 9090
```

## 4) Verify metrics collection

The core components are deployed with the stack. Check:

```bash
kubectl get pods -n monitoring
```

You should see pods for:
- Prometheus
- Grafana
- Alertmanager
- Node Exporter
- Kube-State-Metrics

## 5) Grafana dashboards

In Grafana:
- Go to **Dashboards → Import**
- Import these dashboard IDs:
  - `315` (cluster)
  - `1860` (nodes)

These dashboards cover:
- CPU
- RAM
- Pods
- Nodes

## 6) PromQL (essential queries)

Total CPU:

```promql
sum(rate(container_cpu_usage_seconds_total[5m]))
```

CPU by pod:

```promql
sum(rate(container_cpu_usage_seconds_total[5m])) by (pod)
```

Memory:

```promql
container_memory_usage_bytes
```

Running pods:

```promql
kube_pod_status_phase{phase="Running"}
```

## 7) Prometheus alerts

Create `alerts.yaml`:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: basic-alerts
  namespace: monitoring
spec:
  groups:
  - name: basic
    rules:

    - alert: HighCPU
      expr: sum(rate(container_cpu_usage_seconds_total[5m])) > 0.8
      for: 2m
      labels:
        severity: warning
      annotations:
        summary: "CPU usage high"

    - alert: HighMemory
      expr: container_memory_usage_bytes > 500000000
      for: 2m
      labels:
        severity: warning
      annotations:
        summary: "Memory usage high"

    - alert: PodRestart
      expr: kube_pod_container_status_restarts_total > 5
      for: 1m
      labels:
        severity: critical
      annotations:
        summary: "Pod restarting too much"
```

Apply:

```bash
kubectl apply -f alerts.yaml
```

## 8) Alertmanager (basic configuration)

Edit the Alertmanager secret:

```bash
kubectl edit secret monitoring-kube-prometheus-alertmanager -n monitoring
```

Replace it with a simple config (no external integration required):

```yaml
route:
  receiver: default

receivers:
- name: default
```

Restart Alertmanager:

```bash
kubectl rollout restart statefulset alertmanager-monitoring-kube-prometheus-alertmanager -n monitoring
```

## 9) Test an alert (CPU)

Run a CPU test pod:

```bash
kubectl run cpu-test --image=busybox -it -- /bin/sh
```

Inside the pod:

```sh
while true; do :; done
```

Then check:

```bash
kubectl get pods -n monitoring
```

And in Prometheus UI → **Alerts**.

## 10) Monitoring commands

```bash
kubectl top pods -n monitoring
kubectl logs -n monitoring <pod>
kubectl describe pod -n monitoring <pod>
kubectl get events -n monitoring
```
