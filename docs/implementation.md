# Implementation

[← Back to README](../README.md)

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/e69b1833-7589-4c89-9234-9ac6135e3249" />


## 1. Implementation Overview

This project was implemented using **Google Cloud Shell and the `gcloud` CLI** rather than the GCP web console.

The implementation follows the dependency order established by the architecture:

```text
GCP Project Preparation
        ↓
VPC + Subnets
        ↓
Cloud Router + Cloud NAT
        ↓
Firewall Rules
        ↓
Bastion Host
        ↓
Private Service Access
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
Health Check + Backend Service
        ↓
Global HTTP(S) Load Balancer
        ↓
Certificate Manager
        ↓
Public DNS
        ↓
End-to-End Validation
        ↓
Cleanup
```

The implementation follows a dependency-driven sequence: the network must exist before dependent resources, backend services must exist before the application is fully configured, and the load balancer can only route to an available application backend.

---

# 2. Tooling and Execution Model

The deployment uses **GCloud Shell and the `gcloud` CLI**.

The intended learning model is:

```text
Understand GCP Resource
        ↓
Execute gcloud Command
        ↓
Observe Result
        ↓
Understand Dependency
        ↓
Validate
        ↓
Later Convert to Terraform
```

The project deliberately does not treat memorizing commands as the primary objective.

The important implementation knowledge is:

- Which resource is being created.
- Why it is required.
- What it depends on.
- What depends on it.
- How it connects to other resources.
- How to verify it.

---

# 3. GCP Environment Preparation

Before creating infrastructure, establish the GCP execution context.

The required environment is:

```text
GCP Project
     ↓
Selected Project Context
     ↓
Required APIs / Services
     ↓
gcloud CLI
     ↓
Infrastructure Deployment
```

Typical project-level preparation includes:

```bash
gcloud config set project <PROJECT_ID>
```

The actual project ID, region, zones, CIDR ranges, credentials, and environment-specific values should be supplied from the practical environment rather than hard-coded into this documentation.

---

# 4. Part 1 — VPC Infrastructure

The first implementation stage creates the network foundation.

The required resources are:

| Resource | Purpose |
|---|---|
| Custom VPC | Isolated cloud network |
| Public Subnet 1 | Bastion host |
| Public Subnet 2 | Redundancy |
| Private Subnet 1 | Application instances |
| Private Subnet 2 | Redundancy |
| Cloud Router | Routing support for Cloud NAT |
| Cloud NAT | Private-instance outbound Internet access |
| Firewall Rules | Traffic control |
| Bastion Host | Administrative access |

Every other project component depends on the VPC.

---

## 4.1 Create the VPC

Create a custom-mode VPC.

Conceptually:

```text
VPC
 │
 ├── Public Subnet 1
 ├── Public Subnet 2
 ├── Private Subnet 1
 └── Private Subnet 2
```

A custom VPC is used so that subnet ranges and regional placement are explicitly controlled.

---

## 4.2 Create Public Subnets

Create two public subnets across the selected zones/regions.

The first public subnet hosts the bastion.

```text
Public Subnet 1
     │
     └── Bastion Host

Public Subnet 2
     │
     └── Redundancy
```

The second public subnet provides redundancy across zones.

---

## 4.3 Create Private Subnets

Create two private subnets.

```text
Private Subnet 1
     │
     └── Application Instances

Private Subnet 2
     │
     └── Application Instances
```

The application instances are intentionally placed in private subnets and do not receive public IP addresses.

---

# 5. Cloud Router and Cloud NAT

Cloud Router and Cloud NAT provide outbound Internet connectivity for private instances.

The relationship is:

```text
Private Subnets
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

# 6. Firewall Rules

Firewall rules control the traffic paths through the environment.

The main security model is:

```text
Internet
   │
   ▼
Bastion
   │
   ▼
Private Application Instances
```

and:

```text
Internet
   │
   ▼
Global Load Balancer
   │
   ▼
Private Application Instances
```

The implementation should define rules for the required traffic rather than broadly exposing the private application tier.

---

# 7. Bastion Host

Create a Compute Engine VM in the public subnet.

Conceptually:

```text
Public Subnet
      │
      ▼
Bastion Host
      │
      │ SSH
      ▼
Private Resources
```

The bastion provides a controlled administrative path into the VPC.

The project does not require direct Internet exposure of private application instances.

---

# 8. Part 2 — Private Backend Services

The second major implementation stage establishes private connectivity to the managed backend services.

The required services are:

```text
Cloud SQL
   │
   └── MySQL 8

Memorystore
   │
   └── Memcached
```

These services are not placed directly into the VPC private subnets.

Instead:

```text
Application VPC
       │
       ▼
Private Service Access
       │
       ▼
VPC Peering
       │
       ▼
GCP Service Producer Network
       │
       ├── Cloud SQL
       └── Memorystore
```

This two-network model is an important GCP-specific implementation concept.

---

# 9. Private Service Access

Before creating the managed services, allocate a private IP range for service networking.

The required sequence is:

```text
VPC
 │
 ▼
Allocate PSA IP Range
 │
 ▼
Create Private Service Access
 │
 ▼
VPC Peering
 │
 ▼
Service Producer Network
```

The IP range and peering connection must be established before the managed services are created for this architecture.

---

# 10. Cloud SQL

Create the Cloud SQL MySQL instance.

The project uses:

```text
Cloud SQL
    │
    └── MySQL 8
```

The application requires its database to be available before application validation.

The implementation therefore follows:

```text
PSA / Peering
      ↓
Cloud SQL
      ↓
Private IP
      ↓
Database
      ↓
Application Connectivity
```

The VProfile project uses MySQL 8 on Cloud SQL as the managed relational database layer.

---

# 11. Memorystore

Create the Memorystore Memcached service.

The implementation follows:

```text
PSA / Peering
      ↓
Memorystore
      ↓
Memcached
      ↓
Private IP
      ↓
Application Connectivity
```

Memorystore provides the application's managed caching layer.

---

# 12. Private DNS

After obtaining the private service IP addresses, create private DNS entries.

The application should use service names rather than raw IP addresses.

Conceptually:

```text
db01.vprofile.internal
        │
        ▼
Cloud SQL Private IP
```

and:

```text
mc01.vprofile.internal
        │
        ▼
Memorystore Private IP
```

The implementation sequence is:

```text
Managed Service
      ↓
Private IP
      ↓
Private DNS Record
      ↓
Application Hostname
```

---

# 13. Part 3 — Application VM

Create a Compute Engine VM that will act as the source for the reusable application image.

The intended sequence is:

```text
Application VM
      ↓
Install Tomcat
      ↓
Deploy VProfile
      ↓
Configure Backend Connections
      ↓
Validate Application
      ↓
Create Custom Image
```

The application runs on Tomcat and serves VProfile on port `8080`.

---

# 14. Application Configuration

The application configuration must point to the private backend service names.

Conceptually:

```text
application.properties
        │
        ├── Database → db01
        ├── Cache    → mc01
        └── Other application dependencies
```

The exact environment-specific values should come from the practical configuration.

The important implementation principle is:

> The application connects to backend services through their private service names rather than hard-coded infrastructure IP addresses.

---

# 15. Tomcat Deployment

The application VM runs Tomcat.

The implementation pattern is:

```text
Application VM
      ↓
Java / Runtime
      ↓
Tomcat
      ↓
Port 8080
      ↓
VProfile Application
```

The application instances use the custom image with VProfile pre-deployed on Tomcat and serve the application on port `8080`.

The exact application installation steps are treated as part of the workload preparation rather than as infrastructure ownership.

---

# 16. Application Validation Before Imaging

Before creating the custom image, verify that the configured application VM works.

The expected path is:

```text
Application
    │
    ├──► Cloud SQL
    │
    └──► Memorystore
```

Validate:

- Tomcat is running.
- VProfile loads.
- Database connectivity works.
- Cache connectivity works.

Only after the source VM is functional should it become the image source.

This prevents an invalid application state from being propagated to the entire Managed Instance Group.

---

# 17. Custom Image

Once the application VM is configured and validated:

```text
Configured VM
      │
      ▼
Stop / Prepare VM
      │
      ▼
Create Custom Image
```

The custom image captures the configured application-server state.

The conceptual relationship is:

```text
Custom Image
      │
      ▼
Instance Template
      │
      ▼
MIG
      │
      ▼
Application Instances
```

This is the GCP equivalent of an image-based application deployment workflow.

---

# 18. Part 4 — Instance Template

Create an Instance Template using the custom application image.

The template defines the instance blueprint.

The project specifies:

```text
Machine Type: e2-micro
Image: Custom VProfile Image
Network: Private Subnet
Public IP: None
Application: Tomcat :8080
```

The Instance Template provides a repeatable definition for every application instance launched by the MIG.

---

# 19. Managed Instance Group

Create the Managed Instance Group from the Instance Template.

The implementation model is:

```text
Custom Image
      ↓
Instance Template
      ↓
Managed Instance Group
      ↓
Application Instances
```

The project uses a MIG named using the `vprofile-app01-mig` pattern and configures it to autoscale between **1 and 4 instances**.

The MIG is responsible for maintaining the application fleet.

---

## 19.1 Auto Scaling

Configure the MIG with:

- Minimum instance count.
- Maximum instance count.
- Scaling triggers.

The scaling configuration can use CPU utilization and HTTP request volume.

Important distinction:

```text
MIG
 =
Managed instance lifecycle

Auto Scaling
 =
Dynamic capacity adjustment
```

A MIG can therefore be understood as the fleet-management layer while the autoscaling policy controls capacity.

---

# 20. Health Check

Create the HTTP health check.

The project uses:

```text
Health Check
     │
     ▼
HTTP
     │
     ▼
Port 8080
     │
     ▼
Tomcat
```

The health check uses the application port and determines whether an instance is eligible to receive traffic.

The purpose is to ensure that only healthy application instances receive traffic.

---

# 21. Backend Service

Create the backend service that connects the load balancer to the MIG.

Conceptually:

```text
Backend Service
      │
      ├── Health Check
      │
      └── Managed Instance Group
```

The project uses the `vprofile-app01-backend` naming pattern.

The backend service represents the application fleet from the load balancer's perspective.

---

# 22. Global HTTP(S) Load Balancer

The load balancer is assembled from several individual GCP resources.

The architecture is:

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
```

The project uses the GCP global HTTP(S) load-balancing model.

---

# 23. Static Public IP

Reserve a static public IP for the load balancer.

The relationship is:

```text
Public DNS
      ↓
Static Public IP
      ↓
Load Balancer
```

The DNS record points to the static IP rather than directly to a load-balancer DNS endpoint.

---

# 24. URL Map

Create the URL map used by the load balancer.

The project uses the `vprofile-app01-url-map` naming pattern.

The URL map determines where incoming HTTP requests should be routed.

For this project, the main route is:

```text
/
 ↓
vprofile-app01-backend
```

The URL map can support multiple path-based backends even though this project uses a single application backend.

---

# 25. HTTPS Forwarding Rule

Create the HTTPS forwarding rule on port `443`.

```text
Internet
   │
   │ HTTPS :443
   ▼
HTTPS Forwarding Rule
```

The forwarding rule sends traffic to the target HTTPS proxy.

---

# 26. Target HTTPS Proxy

Create the target HTTPS proxy.

Conceptually:

```text
HTTPS Forwarding Rule
        ↓
Target HTTPS Proxy
        ↓
Certificate Map
        ↓
URL Map
```

The target HTTPS proxy connects the public HTTPS frontend to the certificate and routing configuration.

---

# 27. Certificate Manager

Configure the TLS certificate using GCP Certificate Manager.

The implementation includes:

```text
Certificate
     ↓
DNS Authorization
     ↓
Certificate Map
     ↓
Certificate Map Entry
     ↓
HTTPS Proxy
```

The project uses certificate and certificate-map naming patterns based on the VProfile application.

Certificate validation requires domain verification through the configured DNS provider.

---

# 28. HTTP Forwarding Rule

An HTTP listener can also be configured on port `80`.

The architecture identifies:

```text
HTTP Forwarding Rule
        ↓
Target HTTP Proxy
        ↓
URL Map
        ↓
Backend
```

The HTTP path can then be used for HTTP-to-HTTPS redirection.

---

# 29. Public DNS

The final public DNS record maps the application domain to the static load-balancer IP.

The project uses an A record pattern similar to:

```text
vprogcp.example-domain
        │
        ▼
Static Load Balancer IP
```

The complete public path becomes:

```text
User
  ↓
Application Domain
  ↓
DNS A Record
  ↓
Static LB IP
  ↓
Global HTTPS Load Balancer
```

---

# 30. Complete Implementation Flow

The complete build can be mentally reconstructed as:

```text
1. Configure GCP Project
        ↓
2. Create Custom VPC
        ↓
3. Create Public + Private Subnets
        ↓
4. Create Cloud Router
        ↓
5. Configure Cloud NAT
        ↓
6. Create Firewall Rules
        ↓
7. Create Bastion
        ↓
8. Allocate PSA Range
        ↓
9. Create PSA / VPC Peering
        ↓
10. Create Cloud SQL
        ↓
11. Create Memorystore
        ↓
12. Create Private DNS
        ↓
13. Create Application VM
        ↓
14. Configure Tomcat / VProfile
        ↓
15. Validate Application
        ↓
16. Create Custom Image
        ↓
17. Create Instance Template
        ↓
18. Create MIG
        ↓
19. Configure Auto Scaling
        ↓
20. Create Health Check
        ↓
21. Create Backend Service
        ↓
22. Reserve Static LB IP
        ↓
23. Create URL Map
        ↓
24. Create Certificate
        ↓
25. Create Certificate Map Entry
        ↓
26. Create HTTPS Proxy
        ↓
27. Create HTTPS Forwarding Rule
        ↓
28. Configure HTTP Redirect Path
        ↓
29. Configure Public DNS A Record
        ↓
30. Validate End-to-End
```

This order reflects the dependency-driven implementation strategy.

---

# 31. End-to-End Traffic Flow

Once implementation is complete, the request path is:

```text
User
 │
 ▼
Application Domain
 │
 ▼
DNS A Record
 │
 ▼
Static Load Balancer IP
 │
 ▼
HTTPS Forwarding Rule
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
 ▼
Health Check
 │
 ▼
Managed Instance Group
 │
 ▼
Private Application Instance
 │
 ▼
Tomcat :8080
 │
 ├──────────────► Cloud SQL
 │
 └──────────────► Memorystore
```

The application instances remain private while public traffic enters through the global load balancer.

---

# 32. Security Implementation Model

The resulting security boundary is:

```text
Internet
    │
    ▼
Global HTTPS Load Balancer
    │
    ▼
Private Application Instances
    │
    ├──────────► Cloud SQL
    │
    └──────────► Memorystore
```

Administrative access is separate:

```text
Administrator
      │
      ▼
Bastion
      │
      ▼
Private Resources
```

Outbound private-instance connectivity is:

```text
Private Instance
      │
      ▼
Cloud NAT
      │
      ▼
Internet
```

The architecture therefore avoids direct public exposure of the application and data tiers.

---

# 33. Cleanup Implementation

Cleanup follows the reverse dependency chain.

The general model is:

```text
Public DNS / HTTPS
        ↓
Load Balancer
        ↓
Backend Service / MIG
        ↓
Application Resources
        ↓
Cloud SQL / Memorystore
        ↓
Private Service Access
        ↓
VPC Networking
```

The important principle is:

> **Delete dependents before dependencies.**

For example, the load balancer should not be removed after its underlying backend dependencies have already been destroyed in an arbitrary order.

Private Service Access may require additional time or cleanup handling because allocated service ranges can remain reserved temporarily.

---

# 34. Implementation Verification Points

At the end of each major implementation stage, perform a focused check before continuing.

| Stage | Verify |
|---|---|
| VPC | Subnets, zones, Router, NAT |
| Bastion | SSH and private-resource reachability |
| Backend | Cloud SQL and Memorystore active/private |
| DNS | Service names resolve privately |
| Application VM | Tomcat/application/backend connectivity |
| Image | Custom image exists |
| Template | Correct image/network configuration |
| MIG | Instances created and healthy |
| Health Check | Tomcat `:8080` responds |
| Backend Service | MIG attached and healthy |
| Load Balancer | Frontend chain configured |
| Certificate | Certificate validated |
| Public DNS | Domain resolves to static LB IP |
| End-to-End | HTTPS application access works |

This keeps errors localized rather than allowing a problem in an earlier layer to appear later as an unrelated load-balancer failure.

---

# 35. Implementation Principles

## Principle 1 — Dependency-Ordered Provisioning

```text
Foundation
    ↓
Dependencies
    ↓
Application
    ↓
Scaling
    ↓
Frontend
```

Do not build the frontend before its backend exists.

---

## Principle 2 — Private by Default

The application instances do not require public IP addresses.

```text
Public
  ↓
Load Balancer

Private
  ↓
Application

Private
  ↓
Managed Backend Services
```

---

## Principle 3 — Managed Services Where Appropriate

Cloud SQL and Memorystore are used instead of self-managed database and cache VMs.

This reduces the infrastructure-management burden for those services.

---

## Principle 4 — Image-Based Application Deployment

The configured application VM becomes a reusable custom image.

```text
Configured
     ↓
Image
     ↓
Template
     ↓
MIG
```

This avoids manually configuring every application instance.

---

## Principle 5 — Validate Before Propagating

The source VM must be working before it becomes the basis for the custom image.

Otherwise:

```text
Broken VM
   ↓
Broken Image
   ↓
Broken Template
   ↓
Broken MIG
   ↓
Multiple Broken Instances
```

The safer model is:

```text
Configure
   ↓
Validate
   ↓
Image
   ↓
Scale
```

---

## Principle 6 — CLI-First, Terraform-Later

The project deliberately establishes manual resource understanding before Infrastructure as Code.

```text
gcloud
  ↓
Understand
  ↓
Validate
  ↓
Terraform
```

This is the intended bridge from operational cloud knowledge to automation.

---

# 36. Implementation Boundary

This document describes the implementation of the **GCP infrastructure surrounding the VProfile workload**.

It does not claim that the VProfile application itself was developed as part of this project.

The implementation focus is:

```text
GCP Resources
     +
Network Connectivity
     +
Private Backend Services
     +
Application Deployment
     +
Scalable Compute
     +
Load Balancing
     +
HTTPS
     +
DNS
     +
Validation
     +
Cleanup
```

Terraform, CI/CD, Kubernetes, GitOps, comprehensive observability, disaster recovery, and production-scale performance testing remain future work rather than current implementation capabilities.

---

[← Back to README](../README.md)
