# Infrastructure & DevOps

Production-grade infrastructure-as-code using Kubernetes, ArgoCD, Terraform, and Helm.

## Architecture Layers

### Compute (Kubernetes)
- Multi-region cluster management
- Node pools (GPU, CPU, memory-optimized)
- Network policies and ingress rules
- Pod security standards and admission control

### Storage
- PersistentVolumes for stateful services
- Distributed object storage (S3-compatible)
- Database clusters (PlanetScale, Postgres)
- Backup and disaster recovery

### Networking
- Reverse proxy with authentication (Nginx, Traefik)
- Service mesh for observability (Istio, Linkerd)
- DNS and certificate management (cert-manager)
- Load balancing and traffic shaping

### Observability
- Metrics collection (Prometheus)
- Log aggregation (Loki, ELK)
- Distributed tracing (Jaeger, Tempo)
- Alerting and incident response

## Tools & Technologies

- Terraform: Infrastructure as Code
- Helm: Kubernetes package management
- ArgoCD: GitOps continuous deployment
- Cloudflare Workers: Edge computing
- GitHub Actions: CI/CD pipelines
- Secrets management (sealed-secrets, external-secrets)

## Security Posture

- Non-obvious username patterns
- No public IPFS APIs (authenticated proxies only)
- TLS everywhere, mTLS for service-to-service
- RBAC and pod security policies
- Network segmentation and zero-trust
- Regular security scanning (Semgrep, Trivy)
- Compliance auditing (CIS benchmarks)

## Directory Structure

```
infra/
├── terraform/       # Infrastructure definitions
├── helm/           # Chart values and customizations
├── k8s/            # Raw Kubernetes manifests
├── argocd/         # GitOps application definitions
├── monitoring/     # Prometheus, Grafana configs
├── networking/     # Istio, network policies
└── security/       # RBAC, policies, secrets
```

---
Managed via GitOps (ArgoCD)
