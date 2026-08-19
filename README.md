# GCP V-Profile Infrastructure

Hands-on deployment of the VProfile application workload on Google Cloud Platform using private networking, managed backend services, scalable compute, global HTTPS load balancing, TLS, and DNS.

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/7608af26-7d5e-4191-9e68-fe3a07340a8b" />

---

## Overview

This project demonstrates the deployment of a multi-tier application workload on Google Cloud Platform (GCP).

The infrastructure was built in sequential layers:

```text
VPC Networking
      ↓
Private Backend Services
      ↓
Application Compute
      ↓
Managed Instance Group
      ↓
Global HTTP(S) Load Balancer
      ↓
HTTPS / TLS
      ↓
Public DNS
```

The resulting architecture separates the application tier from backend services, keeps application instances in private subnets, provides private connectivity to managed backend services, and exposes the application through a secure HTTPS load-balancing layer.

---

## Application Ownership Boundary

The VProfile application was used as the existing application workload for this project.

My contribution was focused on the **cloud infrastructure and deployment surrounding the application**, including networking, compute, backend connectivity, load balancing, HTTPS, DNS, validation, and cleanup.

I did **not** develop or author the VProfile application's business logic or application source code.

The repository therefore represents the **DevOps / cloud infrastructure engineering performed around the workload**, rather than application development.

---

## Architecture

The infrastructure consists of four major layers:

```text
                         INTERNET
                            │
                            ▼
                         DNS
                            │
                            ▼
                    Static LB IP
                            │
                            ▼
                 Global HTTP(S) LB
                            │
                    ┌───────┴───────┐
                    │               │
                  HTTPS            HTTP
                    │               │
                    ▼               ▼
               TLS / Proxy      Redirect
                    │
                    ▼
                 URL Map
                    │
                    ▼
              Backend Service
                    │
              Health Check
                    │
                    ▼
             Managed Instance
                 Group
                    │
                    ▼
             Private App VMs
                    │
             ┌──────┴──────┐
             │             │
             ▼             ▼
         Cloud SQL     Memorystore
           MySQL        Memcached
             │             │
             └──────┬──────┘
                    │
             Private Service
                 Access
                    │
               VPC Peering
```

The application instances operate in private subnets. Cloud NAT provides outbound Internet access for private instances without assigning public IP addresses to them.

A bastion host provides an SSH access path to private instances.

For the detailed architecture and traffic flows:

**[Architecture →](docs/architecture.md)**

---

## My Engineering Contribution

I personally performed the infrastructure deployment and configuration required to run the application workload on GCP.

### Networking

- Created the custom VPC.
- Configured public and private subnets across multiple zones.
- Configured Cloud Firewall rules.
- Configured Cloud Router and Cloud NAT.
- Configured a bastion host for controlled SSH access to private instances.

### Backend Services

- Configured Cloud SQL for MySQL.
- Configured Memorystore for Memcached.
- Configured Private Service Access and VPC Peering.
- Configured private connectivity between the application environment and managed backend services.
- Configured private Cloud DNS records for backend service discovery.

### Application Compute

- Prepared the application VM environment.
- Configured the application workload.
- Created a custom machine image from the configured instance.
- Created an Instance Template from the image.
- Created a Managed Instance Group using the template.
- Configured auto scaling for the application tier.

### Load Balancing and HTTPS

- Configured the global HTTP(S) load-balancing architecture.
- Configured the backend service and health check.
- Configured HTTPS forwarding.
- Configured TLS certificate handling through GCP Certificate Manager.
- Configured DNS authorization.
- Configured public DNS to point the application hostname to the load balancer.

### Operations

- Validated the infrastructure and application connectivity.
- Reviewed the complete architecture after deployment.
- Performed dependency-aware cleanup of the deployed environment.

---

## Key Engineering Concepts

### Private Application Tier

Application instances were placed in private subnets rather than directly exposing them to the Internet.

```text
Internet
   │
   ▼
Load Balancer
   │
   ▼
Private Application Instances
```

This separates the public entry point from the application compute layer.

### Private Backend Connectivity

Cloud SQL and Memorystore were accessed through private connectivity using Private Service Access and VPC Peering.

```text
Application VPC
      │
      │ Private Service Access
      ▼
GCP Managed Service Network
      │
 ┌────┴────┐
 ▼         ▼
Cloud SQL  Memorystore
```

### Golden Image → Instance Template → MIG

The application compute tier followed an image-based deployment model:

```text
Configured VM
     │
     ▼
Custom Image
     │
     ▼
Instance Template
     │
     ▼
Managed Instance Group
     │
     ▼
Application Instances
```

The temporary configured instance served as the source for the reusable image and was not intended to remain as permanent infrastructure.

### Global HTTPS Load Balancing

The public request path is:

```text
User
  ↓
DNS
  ↓
Static Load Balancer IP
  ↓
HTTPS Forwarding Rule
  ↓
Target HTTPS Proxy
  ↓
Certificate / TLS
  ↓
URL Map
  ↓
Backend Service
  ↓
Health Check
  ↓
Managed Instance Group
  ↓
Private Application Instance
```

---

## Validation

Validation covered the major infrastructure layers rather than relying only on successful resource creation.

The validation approach included:

- Network configuration checks.
- Private instance access through the bastion.
- Backend service connectivity.
- Private DNS resolution.
- Application availability.
- Managed Instance Group and health-check state.
- Load balancer configuration.
- HTTPS and certificate validation.
- End-to-end application access.

Detailed validation methodology and evidence mapping are documented here:

**[Validation →](docs/validation.md)**

---

## Project Boundaries

This project demonstrates **GCP infrastructure and application deployment**, not development of the VProfile application itself.

The following capabilities are outside the demonstrated scope of this project:

- Terraform-based infrastructure provisioning.
- CI/CD pipeline implementation.
- Kubernetes deployment.
- GitOps.
- Production-scale load testing.
- Disaster recovery implementation.
- Enterprise observability implementation.
- Zero-downtime production deployment validation.
- Development of the VProfile application or its business logic.

These are treated as potential future improvements rather than completed capabilities.

---

## Technologies

### Cloud Platform

- Google Cloud Platform (GCP)

### Networking

- VPC
- Subnets
- Cloud Firewall
- Cloud Router
- Cloud NAT
- VPC Peering
- Private Service Access
- Cloud DNS

### Compute

- Compute Engine
- Custom Images
- Instance Templates
- Managed Instance Groups
- Auto Scaling

### Managed Services

- Cloud SQL
- Memorystore

### Load Balancing & Security

- Global HTTP(S) Load Balancer
- Backend Service
- Health Checks
- HTTPS
- Certificate Manager
- DNS Authorization

### Administration

- Google Cloud Shell
- `gcloud` CLI
- Linux / SSH

---

## Implementation

The implementation documentation explains how the environment was assembled, including:

- GCP environment preparation.
- VPC and subnet provisioning.
- Firewall and bastion configuration.
- Managed backend services.
- Private DNS.
- Application VM preparation.
- Golden image creation.
- Instance Template and MIG configuration.
- Load balancer configuration.
- HTTPS and certificate configuration.
- Public DNS.
- Cleanup.

**[Implementation →](docs/implementation.md)**

---

## Architecture Documentation

For the detailed component relationships, network boundaries, service connectivity, request flow, scaling architecture, and major architectural decisions:

**[Architecture →](docs/architecture.md)**

---

## Validation and Evidence

Validation details and the mapping between technical claims and supporting evidence are documented separately.

**[Validation →](docs/validation.md)**

Evidence should represent only the completed environment and work personally performed during the practical.

---

## Limitations and Future Work

The project intentionally represents the infrastructure capabilities demonstrated during the GCP VProfile deployment.

Potential future evolution includes:

```text
Manual / gcloud CLI
        ↓
Terraform
        ↓
Reusable Infrastructure Modules
        ↓
Infrastructure CI/CD
        ↓
Observability
        ↓
Containerized Deployment
        ↓
Kubernetes
        ↓
GitOps
```

The current project should not be interpreted as having already implemented these future capabilities.

**[Limitations & Future Work →](docs/limitations-and-future-work.md)**

---

## Repository Navigation

| Document | Purpose |
|---|---|
| [Architecture](docs/architecture.md) | System architecture, relationships, traffic flows, security boundaries, and design decisions |
| [Implementation](docs/implementation.md) | How the infrastructure was assembled and configured |
| [Validation](docs/validation.md) | Validation strategy, checks, results, and evidence mapping |
| [Limitations & Future Work](docs/limitations-and-future-work.md) | Project boundaries and potential future evolution |

---

## Engineering Takeaway

The project reinforced a CLI-first approach to cloud infrastructure:

```text
Understand the resources
        ↓
Build using the CLI
        ↓
Understand the relationships
        ↓
Validate the complete flow
        ↓
Understand the dependency chain
        ↓
Automate with Infrastructure as Code
```

The important outcome is not memorizing individual `gcloud` commands.

The important outcome is understanding **what each resource does, why it exists, how resources depend on one another, how traffic moves through the system, and how the infrastructure can later be automated**.
