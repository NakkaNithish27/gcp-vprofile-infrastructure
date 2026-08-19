# Architecture

[← Back to README](../README.md)

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/971cf055-d624-4828-abd8-700bb17b5343" />


## 1. Architecture Overview

This project deploys the existing VProfile application workload on Google Cloud Platform using a layered, multi-tier architecture.

The architecture is built around four major layers:

```text
                    ┌──────────────────────┐
                    │       Internet       │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │   Public DNS / A     │
                    │       Record         │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ Static LB Public IP  │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ Global HTTP(S)       │
                    │ Load Balancer        │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ Application Tier     │
                    │ Managed Instance     │
                    │ Group                │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ Private Application  │
                    │ Instances            │
                    │ Tomcat / VProfile    │
                    └──────────┬───────────┘
                               │
                 ┌─────────────┴─────────────┐
                 │                           │
                 ▼                           ▼
        ┌─────────────────┐        ┌─────────────────┐
        │   Cloud SQL     │        │   Memorystore   │
        │    MySQL 8      │        │    Memcached    │
        └─────────────────┘        └─────────────────┘
```

The application tier is placed in private subnets, while the public entry point is the global HTTP(S) load balancer. Cloud SQL and Memorystore use private connectivity rather than being placed directly inside the VPC subnets.

---

## 2. Architectural Layers

The project follows a dependency-driven sequence:

```text
Layer 1: Network Foundation
        ↓
Layer 2: Administrative Access
        ↓
Layer 3: Managed Backend Services
        ↓
Layer 4: Private Service Discovery
        ↓
Layer 5: Application Image
        ↓
Layer 6: Scalable Application Fleet
        ↓
Layer 7: Public HTTPS Entry Point
```

Each layer depends on infrastructure created by the previous layer.

---

# 3. Network Architecture

The VPC is the foundation of the deployment.

The design uses:

- One custom VPC.
- Two public subnets.
- Two private subnets.
- Subnets distributed across two zones.
- Cloud Firewall rules.
- Cloud Router.
- Cloud NAT.
- Bastion host.

Conceptually:

```text
┌──────────────────────────────────────────────────────────────┐
│                         GCP VPC                              │
│                                                              │
│  ┌──────────────────┐      ┌──────────────────────────────┐ │
│  │ Public Subnet 1  │      │ Private Subnet 1             │ │
│  │                  │      │                              │ │
│  │ Bastion Host     │─────►│ Application Instances       │ │
│  │                  │      │                              │ │
│  └──────────────────┘      └──────────────────────────────┘ │
│                                                              │
│  ┌──────────────────┐      ┌──────────────────────────────┐ │
│  │ Public Subnet 2  │      │ Private Subnet 2             │ │
│  │                  │      │                              │ │
│  │ Redundancy       │      │ Application Redundancy      │ │
│  │                  │      │                              │ │
│  └──────────────────┘      └──────────────────────────────┘ │
│                                                              │
│             Cloud Router + Cloud NAT                         │
│                                                              │
│             Cloud Firewall Rules                             │
└──────────────────────────────────────────────────────────────┘
```

---

## 3.1 Public and Private Subnets

The application instances are deliberately placed in private subnets.

The public subnet contains the bastion host used for administrative access.

This produces the following access model:

```text
Internet
   │
   ▼
Bastion Host
   │
   │ SSH
   ▼
Private Application Instance
```

The application instances do not require public IP addresses for normal operation.

---

## 3.2 Cloud Router and Cloud NAT

Cloud Router and Cloud NAT provide outbound Internet connectivity for private instances.

```text
Private Instance
      │
      ▼
   Cloud NAT
      │
      ▼
 Cloud Router
      │
      ▼
   Internet
```

Cloud NAT provides outbound connectivity without exposing private application instances directly to inbound Internet traffic.

---

# 4. Security Boundaries

The architecture applies layered network boundaries:

```text
Internet
   │
   ▼
Global Load Balancer
   │
   ▼
Private Application Tier
   │
   ▼
Private Managed Backend Services
```

Administrative access follows a separate path:

```text
Administrator
     │
     ▼
Public Bastion
     │
     ▼
Private Resources
```

Firewall rules control the allowed traffic paths between these components.

---

# 5. Backend Services Architecture

The application uses two managed backend services:

```text
Application Tier
      │
      ├──────────────► Cloud SQL
      │                  MySQL 8
      │
      └──────────────► Memorystore
                         Memcached
```

These services operate within GCP's service-producer network and are connected privately to the application VPC.

---

## 5.1 Private Service Access and VPC Peering

The private connectivity model is:

```text
┌─────────────────────────┐
│       Application VPC   │
│                         │
│ Private App Instances   │
└────────────┬────────────┘
             │
             │ Private Service Access
             │
             ▼
      ┌───────────────┐
      │ VPC Peering   │
      └───────┬───────┘
              │
              ▼
┌──────────────────────────────┐
│ GCP Service Producer Network │
│                              │
│  ┌───────────┐ ┌──────────┐ │
│  │ Cloud SQL │ │Memorystore│ │
│  │  MySQL    │ │ Memcached │ │
│  └───────────┘ └──────────┘ │
└──────────────────────────────┘
```

A Private Service Access IP range is allocated from the VPC address space and used for private connectivity to the managed services.

---

# 6. Private DNS Architecture

The application does not need to hard-code the backend service IP addresses.

Private DNS provides name-to-IP resolution:

```text
Application
     │
     │ db hostname
     ▼
Private Cloud DNS
     │
     ▼
Cloud SQL Private IP
```

and:

```text
Application
     │
     │ cache hostname
     ▼
Private Cloud DNS
     │
     ▼
Memorystore Private IP
```

This provides a useful separation between:

```text
Application Configuration
          ↓
       Hostname
          ↓
      DNS Record
          ↓
     Private IP
```

---

# 7. Application Compute Architecture

The application tier follows an image-based deployment model.

The construction flow is:

```text
Temporary Application VM
          │
          │ Configure Tomcat
          │ Deploy VProfile
          │ Validate
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

The configured VM is a temporary construction resource. Once the custom image is created, the reusable image becomes the basis for subsequent instances.

---

# 8. Instance Template

The Instance Template acts as the application-instance blueprint.

Conceptually:

```text
Instance Template
│
├── Machine type
├── Custom application image
├── Private subnet
├── No public IP
├── Network configuration
└── Instance tags
```

The project uses the custom application image as the template's source image and places the resulting instances in the private subnet.

The template separates:

> **What an application instance should look like**

from:

> **How many instances should exist.**

---

# 9. Managed Instance Group

The Managed Instance Group uses the Instance Template to create and manage the application fleet.

```text
              Instance Template
                      │
                      ▼
              Managed Instance
                   Group
                ┌─────┼─────┐
                ▼     ▼     ▼
              VM-1  VM-2  VM-N
```

The project configures the MIG with minimum and maximum instance limits and scaling triggers based on CPU utilization and load-balancing utilization.

The conceptual relationship is:

```text
Custom Image
     ↓
Instance Template
     ↓
MIG
     ↓
Application Instances
     ↓
Cloud SQL + Memorystore
```

The important engineering pattern is:

```text
Blueprint → Fleet → Policy
```

where:

- Instance Template = blueprint.
- MIG = fleet controller.
- Auto Scaling = dynamic capacity policy.

---

# 10. Load Balancer Architecture

The GCP load balancer is not represented as one monolithic resource.

It is a chain of components:

```text
Forwarding Rule
      ↓
Target Proxy
      ↓
Certificate Map / TLS
      ↓
URL Map
      ↓
Backend Service
      ↓
Health Check
      ↓
MIG
      ↓
Application Instances
```

This decomposed architecture is a significant GCP-specific characteristic of the project.

---

## 10.1 HTTPS Request Flow

The complete HTTPS path is:

```text
User
 │
 ▼
Public DNS
 │
 ▼
Static Load Balancer IP
 │
 ▼
HTTPS Forwarding Rule :443
 │
 ▼
Target HTTPS Proxy
 │
 ▼
Certificate Map Entry
 │
 ▼
URL Map
 │
 ▼
Backend Service
 │
 ├── Health Check
 │
 ▼
Managed Instance Group
 │
 ▼
Private Application Instance
 │
 ▼
VProfile Application
```

---

# 11. HTTP Request Flow

An HTTP listener can also be configured:

```text
HTTP :80
   │
   ▼
HTTP Forwarding Rule
   │
   ▼
Target HTTP Proxy
   │
   ▼
HTTP → HTTPS Redirect
   │
   ▼
HTTPS Flow
```

The project uses port 80 as an optional HTTP-to-HTTPS path while HTTPS operates on port 443.

---

# 12. Health Check Architecture

The backend service uses a health check to determine whether application instances are suitable for receiving traffic.

Conceptually:

```text
Load Balancer
     │
     ▼
Backend Service
     │
     ▼
Health Check
     │
     ▼
Application Instance
     │
     ▼
Tomcat :8080
```

The architectural purpose is:

```text
Healthy instance
      ↓
Eligible for traffic

Unhealthy instance
      ↓
Removed from serving path
```

---

# 13. TLS and Certificate Architecture

HTTPS is terminated at the load-balancing layer.

```text
Client
  │
  │ HTTPS
  ▼
HTTPS Forwarding Rule
  │
  ▼
Target HTTPS Proxy
  │
  ▼
Certificate Map
  │
  ▼
TLS Termination
  │
  ▼
URL Map
  │
  ▼
Backend Service
```

The project uses GCP Certificate Manager with DNS authorization and a certificate map entry as part of the HTTPS architecture.

The application instances therefore sit behind the HTTPS termination layer rather than being directly exposed to the public Internet.

---

# 14. Public DNS Architecture

The public application hostname points to the static public IP assigned to the load balancer.

```text
User
 │
 ▼
Application Domain
 │
 ▼
Public DNS A Record
 │
 ▼
Static Load Balancer IP
 │
 ▼
Global HTTPS Load Balancer
```

This provides the public entry point for the application.

---

# 15. Complete End-to-End Architecture

Putting the individual layers together:

```text
                              INTERNET
                                  │
                                  ▼
                         ┌─────────────────┐
                         │   Public DNS    │
                         │    A Record     │
                         └────────┬────────┘
                                  │
                                  ▼
                         ┌─────────────────┐
                         │ Static Public   │
                         │   LB Address    │
                         └────────┬────────┘
                                  │
                                  ▼
                    ┌──────────────────────────┐
                    │ Global HTTP(S) LB        │
                    │                          │
                    │ Forwarding Rule           │
                    │ Target Proxy              │
                    │ Certificate Map           │
                    │ URL Map                   │
                    │ Backend Service           │
                    │ Health Check               │
                    └────────────┬─────────────┘
                                 │
                                 ▼
                    ┌──────────────────────────┐
                    │ Managed Instance Group   │
                    │                          │
                    │ Instance Template        │
                    │ Auto Scaling              │
                    └────────────┬─────────────┘
                                 │
                                 ▼
              ┌────────────────────────────────────┐
              │             GCP VPC                │
              │                                    │
              │   ┌────────────────────────────┐   │
              │   │ Private Subnets             │   │
              │   │                            │   │
              │   │ Tomcat / VProfile VMs      │   │
              │   │ No public IP               │   │
              │   └─────────────┬──────────────┘   │
              │                 │                  │
              │        ┌────────┴────────┐         │
              │        │                 │         │
              │        ▼                 ▼         │
              │   Private DNS       Private DNS    │
              │        │                 │         │
              └────────┼─────────────────┼─────────┘
                       │                 │
                       │ PSA / Peering   │
                       ▼                 ▼
                ┌────────────┐     ┌──────────────┐
                │ Cloud SQL  │     │ Memorystore  │
                │ MySQL 8    │     │ Memcached    │
                └────────────┘     └──────────────┘


Administrative Path:

Administrator
     │
     ▼
Public Bastion Host
     │
     ▼
Private VPC Resources


Outbound Path:

Private Application VM
     │
     ▼
Cloud NAT
     │
     ▼
Cloud Router
     │
     ▼
Internet
```

---

# 16. Architectural Dependency Chain

The infrastructure follows a strict dependency order:

```text
VPC
 │
 ├── Subnets
 │
 ├── Firewall
 │
 ├── Cloud Router
 │      │
 │      └── Cloud NAT
 │
 └── Bastion
        │
        ▼
Private Service Access
        │
        ▼
VPC Peering
        │
        ├── Cloud SQL
        └── Memorystore
                │
                ▼
           Private DNS
                │
                ▼
        Application VM
                │
                ▼
          Custom Image
                │
                ▼
       Instance Template
                │
                ▼
              MIG
                │
                ▼
        Backend Service
                │
                ▼
          Load Balancer
                │
                ▼
       HTTPS / Certificate
                │
                ▼
          Public DNS
```

The important principle is:

> **Each layer is created only after the infrastructure it depends on exists.**

The same principle applies during cleanup in reverse.

---

# 17. Cleanup Architecture

The project is removed in reverse dependency order.

```text
BUILD
─────────────────────────────────────►

VPC
 ↓
Backend
 ↓
MIG / Load Balancer
 ↓
HTTPS / Frontend


ROLLBACK
◄─────────────────────────────────────

HTTPS / Frontend
 ↓
MIG / Load Balancer
 ↓
Backend
 ↓
VPC
```

A notable GCP operational consideration is the Private Service Access range.

After the range is deleted, GCP may retain it for a period, preventing immediate deletion of the VPC peering connection and VPC. Those resources may therefore require later/manual deletion after the range is released.

---

# 18. Major Architectural Decisions

## Private application instances

**Decision:** Keep application instances in private subnets.

**Reason:** The public entry point is the load balancer; application compute does not require direct public exposure.

---

## Managed backend services

**Decision:** Use Cloud SQL and Memorystore rather than self-managed database/cache VMs.

**Reason:** The project uses GCP-managed services for the database and caching layers.

---

## Private Service Access

**Decision:** Connect managed backend services privately using PSA/VPC Peering.

**Reason:** Cloud SQL and Memorystore operate in the GCP service-producer network rather than directly inside the project's VPC subnets.

---

## Golden image

**Decision:** Capture the configured application VM as a custom image.

**Reason:** The image provides a repeatable application-server state for the Instance Template and subsequent MIG instances.

---

## Managed Instance Group

**Decision:** Use an Instance Template + MIG rather than individually managed application VMs.

**Reason:** The application tier requires repeatable instance creation and auto-scaling.

---

## Global HTTP(S) Load Balancer

**Decision:** Put the application fleet behind a global Layer-7 load balancer.

**Reason:** It provides the public application entry point, request routing, health checking, and HTTPS termination.

---

## DNS-based service discovery

**Decision:** Use hostnames for backend service connectivity.

**Reason:** Application configuration can refer to stable names rather than directly depending on private service IP addresses.

---

# 19. GCP-Specific Architecture Patterns

### Pattern 1 — Private managed services

```text
Application VPC
      │
      ▼
PSA / VPC Peering
      │
      ▼
GCP Service Producer Network
      │
      ├── Cloud SQL
      └── Memorystore
```

### Pattern 2 — Blueprint → Fleet → Policy

```text
Custom Image
     ↓
Instance Template
     ↓
MIG
     ↓
Auto Scaling
```

### Pattern 3 — Decomposed Load Balancer

```text
Forwarding Rule
     ↓
Target Proxy
     ↓
Certificate Map
     ↓
URL Map
     ↓
Backend Service
     ↓
Health Check
     ↓
MIG
```

### Pattern 4 — Private application tier

```text
Internet
    ↓
Load Balancer
    ↓
Private Instances
```

### Pattern 5 — CLI-first infrastructure

```text
Understand resources
        ↓
Configure through gcloud
        ↓
Understand dependencies
        ↓
Validate
        ↓
Automate later with Terraform
```

---

# 20. AWS → GCP Concept Mapping

| AWS Concept | GCP Concept Used Here |
|---|---|
| VPC | VPC |
| Security Groups | Cloud Firewall Rules |
| NAT Gateway | Cloud NAT + Cloud Router |
| EC2 | Compute Engine |
| AMI | Custom Image |
| Launch Template | Instance Template |
| Auto Scaling Group | Managed Instance Group |
| ALB | Global HTTP(S) Load Balancer |
| Target Group | Backend Service |
| Listener / Rules | Forwarding Rule + Target Proxy + URL Map |
| ACM | Certificate Manager |
| RDS | Cloud SQL |
| ElastiCache | Memorystore |
| Route 53 | Cloud DNS / external DNS |
| AWS CLI | `gcloud` / Cloud Shell |

---

# 21. Architecture Boundary

This document describes the **cloud infrastructure architecture surrounding the VProfile workload**.

It does not claim ownership of:

- VProfile application business logic.
- VProfile application source development.
- Application-level architecture beyond what is required to deploy and connect the workload.

The architectural focus is:

```text
Cloud Network
    +
Cloud Services
    +
Application Deployment
    +
Scalable Compute
    +
Traffic Management
    +
HTTPS / DNS
```

---

[← Back to README](../README.md)
