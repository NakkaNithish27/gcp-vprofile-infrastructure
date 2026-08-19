# Limitations and Future Work

[← Back to README](../README.md)

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/10933beb-277d-4efa-a827-2c3cda1a613a" />


## 1. Purpose

This document defines the boundaries of the GCP VProfile infrastructure project and identifies logical areas for future improvement.

The project demonstrates a complete cloud infrastructure deployment using GCP services, but it should not be presented as a production-ready platform or as a fully automated Infrastructure-as-Code implementation.

The purpose of this document is to clearly distinguish:

- What was implemented.
- What was validated.
- What was intentionally outside the project scope.
- What could be implemented in a future iteration.

---

# 2. Current Project Scope

The completed project focuses on deploying the VProfile application workload on Google Cloud Platform.

The demonstrated infrastructure includes:

```text
GCP VPC
   ↓
Public / Private Subnets
   ↓
Firewall + Bastion
   ↓
Cloud NAT + Cloud Router
   ↓
Cloud SQL + Memorystore
   ↓
Private DNS
   ↓
Application VM
   ↓
Custom Image
   ↓
Instance Template
   ↓
Managed Instance Group
   ↓
Global HTTP(S) Load Balancer
   ↓
HTTPS / TLS
   ↓
Public DNS
```

The project therefore demonstrates cloud infrastructure fundamentals, private networking, managed services, scalable compute, load balancing, HTTPS, DNS, validation, and dependency-aware cleanup.

---

# 3. Application Ownership Boundary

The VProfile application was used as the workload deployed onto the infrastructure.

The project does **not** represent development of the VProfile application itself.

The infrastructure contribution covers:

- GCP networking.
- Compute resources.
- Managed backend services.
- Private connectivity.
- DNS.
- Load balancing.
- HTTPS.
- Instance scaling configuration.
- Validation.
- Infrastructure cleanup.

The following are outside the ownership boundary:

- VProfile business logic.
- VProfile application source development.
- Application feature development.
- Application-level product design.

This distinction should remain explicit when presenting the project in a portfolio or interview.

---

# 4. Infrastructure-as-Code Limitation

## Current State

The infrastructure was built using the GCP CLI / `gcloud` workflow.

The project intentionally emphasizes understanding the underlying GCP resources and their dependencies before automating them.

The current flow is:

```text
Understand GCP Resources
        ↓
Build with gcloud
        ↓
Understand Dependencies
        ↓
Validate
        ↓
Terraform
```

Terraform is therefore **not implemented as part of the current project**.

---

## Future Improvement

The next logical step would be to reproduce the infrastructure using Terraform.

Potential structure:

```text
terraform/
├── providers.tf
├── variables.tf
├── networking.tf
├── firewall.tf
├── nat.tf
├── sql.tf
├── memorystore.tf
├── dns.tf
├── compute.tf
├── mig.tf
├── load-balancer.tf
├── certificate.tf
└── outputs.tf
```

The infrastructure could then evolve toward:

```text
Manual gcloud
      ↓
Terraform
      ↓
Reusable Modules
      ↓
Terraform CI/CD
```

This would improve repeatability, version control, and environment reconstruction.

---

# 5. Manual Configuration Limitation

Several configuration steps remain dependent on manually executed CLI commands and environment-specific values.

Examples include:

- Project configuration.
- Network naming.
- IP ranges.
- Service configuration.
- DNS values.
- Certificate configuration.
- Application configuration.
- Cleanup.

This means that reproducing the environment requires knowledge of the implementation sequence.

## Future Improvement

Create reusable infrastructure modules and configuration variables so that the environment can be recreated with a small number of Terraform commands.

For example:

```text
terraform init
terraform plan
terraform apply
```

and later:

```text
terraform destroy
```

The objective would be to convert procedural knowledge into declarative infrastructure.

---

# 6. CI/CD Limitation

The project does not implement a complete CI/CD pipeline.

There is no demonstrated pipeline for:

```text
Git Push
   ↓
Build
   ↓
Test
   ↓
Infrastructure Validation
   ↓
Deployment
   ↓
Application Verification
```

## Future Improvement

A future iteration could introduce a CI/CD system capable of:

- Validating Terraform.
- Running security checks.
- Creating a Terraform plan.
- Requiring approval for infrastructure changes.
- Applying infrastructure changes.
- Running post-deployment validation.
- Publishing deployment results.

The long-term model could become:

```text
Developer
    ↓
Git
    ↓
CI
    ↓
Terraform Plan
    ↓
Approval
    ↓
Terraform Apply
    ↓
Validation
```

---

# 7. Containerization Limitation

The current application deployment is VM-based.

The architecture follows:

```text
Custom Image
     ↓
Instance Template
     ↓
Managed Instance Group
     ↓
VM Instances
```

The application is therefore tied to an operating-system-level VM image.

## Future Improvement

The application could eventually be containerized:

```text
VProfile
   ↓
Docker Image
   ↓
Container Registry
   ↓
Container Platform
```

This would separate application packaging from the underlying VM image.

---

# 8. Kubernetes Limitation

Kubernetes is outside the current implementation scope.

The project does not demonstrate:

- GKE cluster creation.
- Kubernetes Deployments.
- Services.
- ConfigMaps.
- Secrets.
- Ingress.
- Horizontal Pod Autoscaling.
- Kubernetes networking.

## Future Improvement

A future containerized architecture could evolve toward:

```text
Docker Image
      ↓
Artifact Registry
      ↓
GKE
      ↓
Kubernetes Deployment
      ↓
Service
      ↓
Ingress / Load Balancer
```

This would represent a separate engineering iteration rather than an incremental configuration change to the current VM-based deployment.

---

# 9. Observability Limitation

The current project does not implement a comprehensive observability platform.

The project validates infrastructure and application behavior, but does not build a complete monitoring stack covering:

```text
Metrics
Logs
Traces
Alerts
Dashboards
SLOs
```

## Future Improvement

A future iteration could introduce:

- Cloud Monitoring dashboards.
- Cloud Logging.
- Application logs.
- Load balancer logs.
- VM metrics.
- Database metrics.
- Alert policies.
- Uptime checks.
- SLO/SLA-oriented monitoring.

The goal would be to move from:

```text
"Is the application working?"
```

toward:

```text
"How is the application behaving,
and will we know when it begins to fail?"
```

---

# 10. Security Hardening Limitation

The project establishes private application networking and firewall boundaries, but it should not be represented as a complete production security architecture.

Areas that could be strengthened include:

- More granular firewall rules.
- IAM least-privilege design.
- Service accounts with narrowly scoped permissions.
- Secret management.
- Automated credential rotation.
- Security scanning.
- Organization policies.
- VPC Service Controls where appropriate.
- Centralized audit logging.
- Security monitoring.

## Future Improvement

A mature security model could evolve toward:

```text
Identity
   +
Least Privilege
   +
Network Segmentation
   +
Secrets Management
   +
Security Monitoring
   +
Policy Enforcement
```

---

# 11. Secrets Management Limitation

Application credentials and backend configuration require secure handling.

The current project does not implement a complete centralized secrets-management workflow.

## Future Improvement

Google Secret Manager could be introduced for:

- Database credentials.
- Application secrets.
- API credentials.
- Other sensitive configuration.

The application architecture could then become:

```text
Application
     │
     ▼
Secret Manager
     │
     ▼
Runtime Credentials
```

This would avoid embedding sensitive values directly into application configuration or deployment commands.

---

# 12. High-Availability Validation Limitation

The architecture uses multiple zones and a Managed Instance Group, but functional validation does not automatically prove high availability under real failure conditions.

The project does not demonstrate controlled:

- VM failure.
- Zone failure.
- Backend failure.
- Load-balancer failure.
- Database failure.
- Regional failure.

## Future Improvement

A future validation exercise could intentionally introduce controlled failures and verify recovery.

For example:

```text
Healthy Application
        ↓
Terminate Instance
        ↓
MIG Detects Failure
        ↓
New Instance Created
        ↓
Health Check Passes
        ↓
Traffic Resumes
```

This would provide stronger evidence of operational resilience.

---

# 13. Auto-Scaling Validation Limitation

Auto-scaling is configured, but configuration alone does not prove that the application scales correctly under sustained production-like load.

The project should therefore distinguish between:

```text
Auto Scaling Configured
```

and:

```text
Auto Scaling Behavior Experimentally Validated
```

## Future Improvement

A future project iteration could introduce controlled load testing:

```text
Baseline
   ↓
Generate Load
   ↓
Observe Metrics
   ↓
Trigger Scaling
   ↓
Verify New Instances
   ↓
Reduce Load
   ↓
Verify Scale-In
```

Evidence could include:

- CPU metrics.
- MIG instance count.
- Load-balancing utilization.
- Scaling events.
- Application response behavior.

---

# 14. Performance Testing Limitation

The project validates functionality but does not establish formal performance characteristics.

It does not provide production-grade measurements for:

- Requests per second.
- Latency.
- Throughput.
- Concurrent users.
- Database performance.
- Cache hit ratio.
- Load-balancer performance.

## Future Improvement

A controlled performance test could establish a baseline:

```text
Load
 ↓
Latency
 ↓
Throughput
 ↓
Resource Utilization
 ↓
Scaling Behavior
```

Results could then be used to tune:

- MIG scaling policies.
- Machine types.
- Database configuration.
- Cache sizing.
- Load-balancing configuration.

---

# 15. Disaster Recovery Limitation

The project does not implement a complete disaster recovery strategy.

It does not demonstrate:

- Cross-region application recovery.
- Database disaster recovery.
- Backup restoration testing.
- Recovery Time Objective (RTO).
- Recovery Point Objective (RPO).

## Future Improvement

A future architecture could introduce:

```text
Primary Region
      │
      ├── Application
      ├── Database
      └── Cache
             │
             ▼
      Recovery Strategy
             │
             ▼
     Secondary Region
```

The important next step would not simply be creating backups, but actually testing restoration and measuring recovery.

---

# 16. Multi-Region Limitation

The current deployment is not a demonstrated multi-region architecture.

The project focuses on a single GCP environment.

## Future Improvement

A multi-region design could introduce:

```text
                    Global DNS / LB
                         │
             ┌───────────┴───────────┐
             ▼                       ▼
        Region A                  Region B
             │                       │
        Application              Application
             │                       │
        Backend                  Backend
```

This would introduce additional complexity around:

- Data replication.
- Database consistency.
- Cache behavior.
- Failover.
- DNS.
- Regional health checks.
- Deployment coordination.

---

# 17. Production DNS Management Limitation

The project uses public DNS to expose the application through the load balancer.

The current project does not demonstrate a full production DNS management lifecycle.

## Future Improvement

A future implementation could automate DNS records through Terraform and integrate DNS changes into the infrastructure deployment process.

The desired model would be:

```text
Terraform
   ↓
Static IP
   ↓
Load Balancer
   ↓
DNS Record
   ↓
HTTPS Domain
```

This would remove manual DNS configuration from the deployment workflow.

---

# 18. Certificate Lifecycle Limitation

HTTPS and certificate configuration are demonstrated, but the project does not implement a broader certificate-management operating model.

## Future Improvement

Automate:

- Certificate creation.
- DNS authorization.
- Certificate-map configuration.
- Certificate rotation.
- Certificate validation.
- Expiration monitoring.

The objective would be to make certificate management part of the infrastructure lifecycle rather than a separate manual activity.

---

# 19. Environment Separation Limitation

The project is focused on a single practical deployment environment.

It does not demonstrate a complete multi-environment structure such as:

```text
Development
    ↓
Staging
    ↓
Production
```

## Future Improvement

Terraform workspaces or separate environment configurations could be used to create:

```text
environments/
├── dev/
├── staging/
└── prod/
```

with reusable modules:

```text
modules/
├── network/
├── compute/
├── database/
├── cache/
├── load-balancer/
└── dns/
```

---

# 20. State Management Limitation

Because Terraform is not implemented in the current project, Terraform state management is outside the current scope.

## Future Improvement

A production-style Terraform implementation should consider:

- Remote state.
- State locking.
- State access control.
- State encryption.
- Environment isolation.
- CI/CD integration.

This becomes particularly important once multiple engineers or automated pipelines begin modifying infrastructure.

---

# 21. Testing Automation Limitation

The current validation process is primarily manual and command-driven.

A future iteration could automate infrastructure tests.

For example:

```text
Terraform Apply
      ↓
Automated Tests
      ↓
DNS Test
      ↓
HTTPS Test
      ↓
Health Check Test
      ↓
Application Test
      ↓
Database Test
      ↓
Cache Test
```

This would transform validation from a manual checklist into a repeatable deployment gate.

---

# 22. Future Evolution Roadmap

The infrastructure can evolve through increasingly automated stages:

```text
Stage 1
Manual gcloud Deployment
        │
        ▼
Stage 2
Terraform
        │
        ▼
Stage 3
Reusable Terraform Modules
        │
        ▼
Stage 4
Terraform CI/CD
        │
        ▼
Stage 5
Observability + Security Automation
        │
        ▼
Stage 6
Containerization
        │
        ▼
Stage 7
GKE / Kubernetes
        │
        ▼
Stage 8
GitOps
```

Each stage should be implemented only after the previous layer is understood and validated.

---

# 23. Proposed Future Architecture

A mature version of this project could evolve toward:

```text
                         Git Repository
                               │
                               ▼
                         CI/CD Pipeline
                               │
                 ┌─────────────┴─────────────┐
                 │                           │
                 ▼                           ▼
          Terraform Plan              Application Build
                 │                           │
                 ▼                           ▼
          Terraform Apply             Container Image
                 │                           │
                 ▼                           ▼
              GCP VPC                 Artifact Registry
                 │                           │
                 │                           ▼
                 │                          GKE
                 │                           │
                 └──────────────┬────────────┘
                                │
                                ▼
                       Global HTTPS LB
                                │
                                ▼
                              DNS
                                │
                                ▼
                              Users

Supporting Services:

        Cloud SQL
        Memorystore
        Secret Manager
        Cloud Monitoring
        Cloud Logging
```

This is a **future architecture**, not a description of what the current project has already implemented.

---

# 24. What This Project Demonstrates

The strongest claims supported by this project are:

- Understanding of GCP networking.
- Private and public subnet design.
- Firewall-based network control.
- Bastion-based administrative access.
- Cloud NAT for private outbound connectivity.
- Managed database deployment using Cloud SQL.
- Managed caching using Memorystore.
- Private Service Access and VPC Peering.
- Private DNS service discovery.
- VM-based application deployment.
- Custom image creation.
- Instance Templates.
- Managed Instance Groups.
- Auto-scaling configuration.
- Health checks.
- Global HTTP(S) load balancing.
- HTTPS and TLS configuration.
- Public DNS.
- End-to-end infrastructure validation.
- Dependency-aware infrastructure cleanup.
- CLI-first cloud infrastructure workflow.

These capabilities provide the foundation for the next stage of the DevOps learning roadmap: Infrastructure as Code and further automation.

---

# 25. What This Project Does Not Demonstrate

The following should not be claimed as completed capabilities unless separately implemented and evidenced:

- Terraform infrastructure.
- Terraform modules.
- Terraform CI/CD.
- Full CI/CD.
- Kubernetes.
- GKE.
- GitOps.
- Production-grade observability.
- Production-scale performance testing.
- Disaster recovery.
- Multi-region failover.
- Enterprise security architecture.
- Production zero-downtime deployment.
- Experimentally proven auto-scaling under sustained load.

Keeping these boundaries explicit makes the project more credible because the portfolio describes what was actually built rather than what could theoretically be done.

---

# 26. Final Takeaway

The project should be viewed as a foundation rather than the final state of a production DevOps platform.

The engineering progression is:

```text
Cloud Fundamentals
        ↓
GCP Infrastructure
        ↓
Networking
        ↓
Managed Services
        ↓
Scalable Compute
        ↓
Load Balancing
        ↓
HTTPS / DNS
        ↓
Validation
        ↓
Infrastructure as Code
        ↓
CI/CD
        ↓
Observability
        ↓
Containers
        ↓
Kubernetes
        ↓
GitOps
```

The immediate next engineering step is **Infrastructure as Code**, specifically converting the manually understood GCP architecture into reproducible Terraform configuration.

The goal is not to skip directly to advanced tooling.

The goal is to preserve the understanding developed here while progressively replacing manual execution with automation.

---

[← Back to README](../README.md)
