# Supporting Artifacts

# 1. Infrastructure as Code

The Terraform layer establishes a repeatable, production-aligned baseline. I intentionally keep it modular and aligned with community-supported modules to reduce undifferentiated operational burden.

## providers.tf

```hcl
provider "aws" {
  region = var.aws_region
}
```

This keeps region configurable and promotes environment portability.

---

## vpc.tf

```hcl
module "vpc" {
  source = "terraform-aws-modules/vpc/aws"

  name = "drafty-vpc"
  azs  = var.azs

  public_subnets  = var.public_subnets
  private_subnets = var.private_subnets
}
```

Design decisions:

- Multi-AZ by default for high availability.
- Public subnets reserved strictly for load balancers.
- Private subnets for worker nodes, pods and virtual machines (if needed).
- Bigger subnet sizing planned for future growth to avoid IP exhaustion.

---

## eks.tf (Skeleton)

```hcl
module "eks" {
  source          = "terraform-aws-modules/eks/aws"
  cluster_name    = "drafty-eks"
  cluster_version = "1.35"

  subnets = module.vpc.private_subnets

  node_groups = {
    app = {
      desired_capacity = 2
      instance_types   = ["t3.medium"] - Exemple
    }
  }

  manage_aws_auth = true
}
```

Rationale:

- Managed control plane reduces operational overhead.
- Initial desired capacity = 2 for baseline HA.
- Instance types chosen for moderate traffic profile.
- Node groups can later be segmented by workload class.

This skeleton establishes a reliable, evolvable baseline without over-engineering early scale.

---

# 2. Kubernetes Manifests

These manifests demonstrate safe rollout configuration, health signaling, and observability integration.

---

## deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: drafty-bird
  namespace: drafty-bird
spec:
  replicas: 2
  selector:
    matchLabels:
      app: drafty-bird
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  template:
    metadata:
      labels:
        app: drafty-bird
    spec:
      serviceAccountName: drafty-bird-sa
      containers:
      - name: drafty-bird
        image: <ECR_REPO>/drafty-bird:{{ .Values.image.tag }}
        ports:
        - containerPort: 8080
        readinessProbe:
          httpGet:
            path: /readyz
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 20
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
```

Design considerations:

- `maxUnavailable = 0` prevents availability drops during rollout.
- `maxSurge = 1` enables controlled incremental updates.
- Readiness probe gates traffic.
- Liveness probe enables self-healing.
- Explicit resource requests/limits ensure predictable scheduling and autoscaling behavior.

This configuration prioritizes stability over deployment speed.

---

## service-monitor.yaml (Prometheus Operator)

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: drafty-bird-sm
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: drafty-bird
  endpoints:
  - port: http
    path: /metrics
    interval: 1s
```

This enables Prometheus scraping without modifying application code.

---

# 3. PromQL Queries & Alert Rules

The focus is on service health, traffic, latency, and error rates — aligned with SRE golden signals.

---

## PromQL Examples

### Throughput

```promql
sum(rate(http_requests_total{job="drafty-bird"}[1m])) by (status)
```

Used to understand traffic patterns and status code distribution.

---

### p95 Latency

```promql
histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket{job="drafty-bird"}[5m])) by (le))
```

Primary latency SLI.

---

### Error Rate

```promql
sum(rate(http_requests_total{job="drafty-bird",status=~"5.."}[5m])) 
/ 
sum(rate(http_requests_total{job="drafty-bird"}[5m]))
```

Core reliability indicator.

---

## Alert Rule Examples

```yaml
groups:
- name: drafty-bird.rules
  rules:
  - alert: DraftyBirdHighErrorRate
    expr: increase(http_requests_total{job="drafty-bird",status=~"5.."}[5m]) 
          / increase(http_requests_total{job="drafty-bird"}[5m]) > 0.05
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "High error rate for Drafty Bird"
      runbook: "Check ArgoCD for recent deploys; inspect pods and traces; rollback if needed."

  - alert: DraftyBirdHighLatencyP95
    expr: histogram_quantile(0.95, 
          sum(rate(http_request_duration_seconds_bucket{job="drafty-bird"}[5m])) by (le)) > 1.5
    for: 10m
    labels:
      severity: warning
```

Operational thinking:

- Error rate is critical because it directly impacts user experience.
- Latency degradation is warning-level initially.
- Alerts include actionable runbook context.
- First triage step: check recent deployments.

---

# 4. Architecture Diagram (ASCII)

This diagram reflects layered network boundaries and operational tooling.

```
Internet
  |
  v
Route53 -> ALB (public subnets)
                 |
                 v
           Ingress Controller
                 |
                 v
        EKS Cluster (private subnets)
        ├─ Namespace: drafty-bird
        │   ├─ Deployment: drafty-bird (pods)
        │   └─ Service: ClusterIP
        ├─ Observability: Prometheus, OTel Collector, Loki/CloudWatch
        └─ GitOps: ArgoCD (manages app manifests)
```

Design characteristics:

- Clear public/private network boundary.
- No direct exposure of worker nodes.
- Observability components co-located but logically separated.
- GitOps controller as control plane for application state.

---