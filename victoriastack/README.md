Needs its own plugin for grafana, run:

```
kubectl exec -n opentelemetry grafana-5bcb4b9b4f-8s2df \
  -- grafana-cli plugins install victoriametrics-logs-datasource
```

Then:

```
kubectl set env deployment/grafana \
  -n opentelemetry \
  GF_INSTALL_PLUGINS=victoriametrics-logs-datasource
```

Finally:

```
kubectl rollout restart deployment/grafana -n opentelemetry
```