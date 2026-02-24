## 1. Context and Decision Criteria

I approached this problem by first clarifying the constraints and priorities.

- Cost is an important factor, but not the primary driver.
- Stability and reliability are non-negotiable.
- The application is being published for the first time.
- We already understand its resource consumption profile.
- The team is aligned with automation and GitOps practices.
- A single cluster is acceptable initially to enable a timely go-live.

I chose **Amazon EKS** as the hosting platform based on my expertise.

---

## 2. Decision

I evaluated the solution across four main dimensions: reliability, scalability, governance and operational maturity.

### 2.1 Reliability

Reliability was the primary decision driver.

EKS provides:

- Managed control plane with built-in high availability
- Native Kubernetes self-healing (pod restarts, replica reconciliation)
- Multi-AZ worker node distribution
- Clear separation between control plane and data plane

This significantly reduces operational risk while preserving flexibility.

By leveraging Kubernetes primitives (Deployments, ReplicaSets, PDBs, readiness/liveness probes), we gain predictable recovery behavior and controlled rollout strategies.

---

### 2.2 Scalability Strategy

Rather than relying only on basic HPA, I would design scaling at two layers:

- **Karpenter** for intelligent node provisioning based on real workload needs
- **KEDA** where event-driven or external-metric scaling is required

Additionally:

- All workloads must define CPU and memory requests/limits.
- Subnets are sized intentionally to prevent future IP exhaustion.
- Node groups can be segmented per workload profile (e.g., compute-intensive vs. general-purpose).

This ensures:

- Predictable scaling behavior
- Controlled cost growth
- Reduced risk of noisy neighbor issues
- Better bin-packing and infrastructure efficiency

---

### 2.3 Governance and Change Management

From day one, I would implement GitOps using **ArgoCD**.

The reasoning is strategic:

- Declarative desired state stored in Git
- Full auditability of infrastructure and application changes
- Deterministic deployments
- Fast and safe rollback capability

This shifts operational control from imperative actions to a controlled, versioned workflow.

Additionally, I would modify the CI pipeline to:

1. Build and push images to ECR
2. Automatically update the image tag in the Git repository
3. Sync manually the new state in ArgoCD UI in the first moment.

This removes manual manifest edits and reduces human error.

---

### 2.4 Networking and Infrastructure Design

Infrastructure would be provisioned via Terraform:

- VPC spanning at least 2–3 Availability Zones
- Public subnets for the load balancer
- Private subnets for nodes and pods
- EKS cluster with required add-ons (CNI, CoreDNS)
- IAM roles via IRSA
- DNS management using **Amazon Route 53**

Traffic flow would be intentionally segmented:

Route53 → Public ALB → Ingress Controller → ClusterIP Service → Pods (private subnets)

Security principles applied:

- No public exposure of worker nodes
- Security groups limiting inbound access
- Clear separation between public entry point and private workloads

This minimizes attack surface and enforces layered network boundaries.

---

## 3. Operational Readiness and Recovery

From an operational standpoint, I would ensure:

- Health endpoints are clearly defined:
  - `/healthz` for liveness
  - `/readyz` for readiness
- Rolling updates configured with:
  - maxUnavailable = 0
  - maxSurge = 1
- PodDisruptionBudgets to protect minimum availability

Rollback strategy:

- Immediate rollback via ArgoCD if regression is detected
- Incremental deploys to limit blast radius

As maturity increases, this can evolve into progressive delivery (e.g., canary deployments with automated metric evaluation).

---

## 4. Trade-Off Analysis

I explicitly considered the trade-offs:

- Kubernetes introduces operational complexity.
- EKS may have higher baseline cost compared to simpler orchestration services.
- A single cluster increases blast radius.

However, in this context, the benefits outweigh the downsides:

- Long-term scalability
- Portability
- Standardization across teams
- Strong governance model
- Mature ecosystem for observability and automation

The decision prioritizes reliability and long-term architectural soundness over short-term simplicity.

---