---
title: Monitor
sidebar_position: 3
---

# Monitor

Butler controllers expose Prometheus metrics on `:8080/metrics`.

## Built-in Metrics

| Metric | Description |
|--------|-------------|
| `butler_tenantcluster_count` | Total tenant clusters by phase |
| `butler_tenantcluster_phase` | Current phase per cluster |
| `butler_reconcile_duration_seconds` | Reconciliation latency histogram |
| `butler_addon_install_duration_seconds` | Addon installation latency histogram |

## Recommended Stack

Install Victoria Metrics and Grafana as ManagementAddons:

```yaml
apiVersion: butler.butlerlabs.dev/v1alpha1
kind: ManagementAddon
metadata:
  name: victoria-metrics
spec:
  addon: victoria-metrics
---
apiVersion: butler.butlerlabs.dev/v1alpha1
kind: ManagementAddon
metadata:
  name: grafana
spec:
  addon: grafana
```

## Alerting Rules

```yaml
groups:
  - name: butler
    rules:
      - alert: TenantClusterFailed
        expr: butler_tenantcluster_phase{phase="Failed"} > 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Tenant cluster in Failed state"

      - alert: HighReconcileLatency
        expr: histogram_quantile(0.99, butler_reconcile_duration_seconds_bucket) > 60
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Slow reconciliation detected"
```

## Enable Debug Logging

Set the `LOG_LEVEL` environment variable on the controller deployment:

```bash
kubectl set env deploy/butler-controller -n butler-system LOG_LEVEL=debug
```
