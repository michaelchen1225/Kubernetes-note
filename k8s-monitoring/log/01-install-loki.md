# Install kube-prometheus-stack + Loki

## Install kube-prometheus-stack

```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

```
helm upgrade --install -f  /tmp/values-yaml/prom-values.yaml prom prometheus-community/kube-prometheus-stack -n monitoring
```

```
kubectl -n monitoring edit svc prom-grafana
```
> Change `type: ClusterIP` to `type: LoadBalancer` & `port: 80` to `port: 3001`

```
cat <<EOF > self-monitoring.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: prometheus-self
  labels:
    app: kube-prometheus-stack-prometheus
spec:
  endpoints:
  - interval: 30s
    port: web
  selector:
    matchLabels:
      app: kube-prometheus-stack-prometheus
EOF
```

```
kubectl apply -f self-monitoring.yaml -n monitoring
```

```
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

```
helm upgrade --install loki grafana/loki-distributed -n monitoring
```

```
helm upgrade --install -f /tmp/values-yaml/promtail-values.yaml promtail grafana/promtail -n monitoring
```