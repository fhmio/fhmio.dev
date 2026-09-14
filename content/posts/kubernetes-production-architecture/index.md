---
title: "Kubernetes in Production: Architecture, Hardening, and Observability"
summary: "A battle-tested blueprint for running containerized microservices in production Kubernetes clusters: Helm packaging, security contexts, ingress routing, and Prometheus metrics."
categories: ["DevOps", "Kubernetes"]
tags: ["kubernetes", "k8s", "devops", "containers", "observability"]
date: 2026-06-10
draft: false
showTableOfContents: true
---

Running **Kubernetes (k8s)** in development with minikube or kind is vastly different from managing distributed microservices under real traffic in production. Ensuring high availability, rapid incident detection, and multi-tenant security requires implementing standardized operational patterns.

In this deep dive, we walk through the architectural pillars every DevOps engineer should establish when operating production Kubernetes clusters.

---

## 🛡️ 1. Pod Hardening & Security Standards

Never allow containers to run with root privileges or unrestricted system calls. Always declare an explicit `securityContext`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway
  namespace: production
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: api-gateway
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
        runAsGroup: 10001
        fsGroup: 10001
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: gateway
          image: ghcr.io/fhmio/gateway:v1.4.0
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
```

---

## 📈 2. Automated Scaling with HPA & PDB

Under unpredictable traffic spikes, manual scaling is too slow. Combine **Horizontal Pod Autoscaling (HPA)** with **Pod Disruption Budgets (PDB)** to guarantee availability even during node drains:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-gateway-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-gateway
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 75
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-gateway-pdb
  namespace: production
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: api-gateway
```

---

## 🔭 3. The Modern Observability Stack

You cannot manage what you cannot observe. A reliable Kubernetes production setup requires:

1. **Metrics Collection:** Prometheus Agent scraping application `/metrics` endpoints and cluster metrics with `kube-state-metrics`.
2. **Visualization:** Grafana dashboards showing p95/p99 latency, error rates, and CPU/Memory saturation.
3. **Log Aggregation:** Fluent Bit forwarding structured JSON logs to OpenSearch / Grafana Loki.
4. **Distributed Tracing:** OpenTelemetry collectors providing end-to-end trace context across distributed service calls.

Combining these operational guardrails allows teams to deploy multiple times a day with high confidence and minimal operational toil.