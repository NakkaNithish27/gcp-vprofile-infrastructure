# Validation

[← Back to README](../README.md)

## 1. Validation Objective

Validation confirms that the GCP VProfile infrastructure works as an integrated system rather than merely confirming that individual GCP resources were created successfully.

The validation follows the same dependency chain as the deployment:

```text
Network
   ↓
Administrative Access
   ↓
Backend Services
   ↓
Private DNS
   ↓
Application VM
   ↓
Managed Instance Group
   ↓
Load Balancer
   ↓
HTTPS / TLS
   ↓
Public DNS
   ↓
End-to-End Application
```

---

# 2. Validation Strategy

Validation is divided into five levels:

| Level | Validation Area | Primary Question |
|---|---|---|
| 1 | Infrastructure | Were the required resources created correctly? |
| 2 | Connectivity | Can the required components communicate? |
| 3 | Service Health | Are application instances and backend services healthy? |
| 4 | Traffic Flow | Can the load balancer route requests correctly? |
| 5 | Application | Does the application actually use the database and cache? |

The strongest validation is the final end-to-end test because it confirms several infrastructure layers simultaneously.

---

# 3. Phase-by-Phase Validation

## 3.1 VPC Validation

### Objective

Confirm that the network foundation exists and is configured correctly.

### Verify

- Custom VPC exists.
- Two public subnets exist.
- Two private subnets exist.
- Subnets are associated with the intended zones.
- Cloud Router exists.
- Cloud NAT is configured.
- Firewall rules exist.

### Expected architecture

```text
GCP VPC
│
├── Public Subnet 1
├── Public Subnet 2
├── Private Subnet 1
├── Private Subnet 2
│
├── Cloud Router
└── Cloud NAT
```

### Evidence

Recommended evidence:

- VPC/subnet listing.
- Cloud Router/NAT configuration.
- Firewall rule listing.

### What this proves

> The foundational network required by the remaining project components is available.

---

# 4. Bastion Validation

## Objective

Confirm that administrative access into the private environment works.

The expected path is:

```text
Administrator
     │
     ▼
Bastion Host
     │
     │ SSH
     ▼
Private Instance
```

### Verify

1. SSH access to the bastion works.
2. From the bastion, private resources can be reached as intended.
3. Private application instances do not require public IP addresses for administrative access.

### What this proves

> The administrative access path into the private environment is functional.

---

# 5. Backend Service Validation

## Objective

Confirm that Cloud SQL and Memorystore are operational and privately connected.

The backend architecture is:

```text
Private Application VPC
        │
        ▼
PSA / VPC Peering
        │
        ├──────────────► Cloud SQL
        │
        └──────────────► Memorystore
```

---

## 5.1 Cloud SQL

### Verify

- Cloud SQL instance exists.
- Instance is active/running.
- Private IP is assigned.
- Required `accounts` database exists.
- Database credentials are configured consistently with the application configuration.

### What this proves

> The managed database layer is available for the application.

---

## 5.2 Memorystore

### Verify

- Memorystore Memcached instance exists.
- Instance is active.
- Authorized VPC/network is correct.
- Private connectivity is available.

### What this proves

> The managed caching layer is available to the application.

---

# 6. Private DNS Validation

## Objective

Confirm that backend service hostnames resolve to their private IP addresses.

Expected flow:

```text
Application
     │
     │ DNS query
     ▼
Private Cloud DNS
     │
     ├──────────► Cloud SQL private IP
     │
     └──────────► Memorystore private IP
```

### Verify

- Cloud SQL hostname resolves.
- Memorystore hostname resolves.
- Returned addresses are private addresses.
- Resolution works from the appropriate private network context.

### What this proves

> Service discovery is functioning without requiring the application to depend directly on hard-coded backend IP addresses.

---

# 7. Application VM Validation

## Objective

Before creating the reusable image and scalable fleet, confirm that the application works on the configured VM.

Expected path:

```text
Application VM
      │
      ├──────► Cloud SQL
      │
      └──────► Memorystore
```

### Verify

- Tomcat is running.
- Application is listening on port `8080`.
- VProfile application loads.
- Database connectivity works.
- Cache connectivity works.

### What this proves

> The application image source is functional before it is converted into reusable compute infrastructure.

---

# 8. Golden Image Validation

## Objective

Confirm that the custom image represents a usable application-server state.

The intended lifecycle is:

```text
Configured VM
     │
     ▼
Application Verified
     │
     ▼
VM Stopped
     │
     ▼
Custom Image
```

### Verify

- Custom image exists.
- Image was created from the configured application VM.
- Expected application state is captured.
- Instance Template references the intended custom image.

### What this proves

> The application configuration can be reproduced through an image-based deployment model.

---

# 9. Instance Template Validation

## Objective

Confirm that the Instance Template contains the expected application-instance configuration.

### Verify

- Correct custom image.
- Correct machine type.
- Correct VPC/subnet.
- Private application network placement.
- Expected instance configuration.
- Required instance tags/named-port configuration.

### Expected relationship

```text
Custom Image
     │
     ▼
Instance Template
     │
     ▼
MIG
```

### What this proves

> The application instance configuration has been converted into a reusable deployment blueprint.

---

# 10. Managed Instance Group Validation

## Objective

Confirm that the MIG can create and manage the application instances.

### Verify

- MIG exists.
- Instance Template is attached.
- Expected instances are created.
- Instances are healthy.
- Named port configuration maps HTTP traffic to port `8080`.
- Minimum and maximum instance limits are configured.
- Auto-scaling policy is configured.

Expected relationship:

```text
Instance Template
        │
        ▼
      MIG
        │
   ┌────┴────┐
   ▼         ▼
 App VM    App VM
```

### What this proves

> The application tier can be managed as a repeatable instance fleet.

---

## 10.1 Auto-Scaling Validation Boundary

Configured auto scaling is not the same as experimentally proving scale-out under production load.

Unless load testing was actually performed and evidence was captured, the repository should claim:

> "Configured auto scaling."

rather than:

> "Validated production-scale auto scaling."

This distinction keeps the project evidence-based.

---

# 11. Health Check Validation

## Objective

Confirm that the load balancer can determine whether application instances are healthy.

Expected flow:

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

### Verify

- Health check exists.
- Health check uses HTTP.
- Port is `8080`.
- Expected path is `/`.
- MIG instances become healthy.

### What this proves

> The load-balancing layer can distinguish healthy application instances from unhealthy ones.

---

# 12. Backend Service Validation

## Objective

Confirm that the backend service correctly connects the load balancer to the MIG.

Expected relationship:

```text
Backend Service
      │
      ├── Health Check
      │
      └── MIG
```

### Verify

- Backend service exists.
- Correct MIG is attached.
- Named port is configured.
- Health check is associated.
- Backend instances become healthy.

### What this proves

> The load balancer has a valid application backend.

---

# 13. HTTP Load Balancer Validation

## Objective

Confirm that the HTTP load-balancing chain works before HTTPS is introduced.

Expected chain:

```text
Internet
   ↓
Static IP
   ↓
Forwarding Rule
   ↓
HTTP Proxy
   ↓
URL Map
   ↓
Backend Service
   ↓
MIG
   ↓
Healthy Instance
   ↓
Tomcat :8080
   ↓
VProfile
```

### Verify

- Static public IP exists.
- Forwarding rule exists on port 80.
- Target HTTP proxy is configured.
- URL map points to the backend service.
- Backend service reports healthy instances.
- Browser request reaches the VProfile application.

### What this proves

> The complete HTTP request path from the Internet to the private application tier is functional.

---

# 14. Application-Level Validation

The application itself provides stronger validation than simply checking whether a web page loads.

## Test 1 — Application page

Access the load balancer endpoint.

Expected:

```text
VProfile Login Page
```

This confirms:

```text
Load Balancer
      ↓
MIG
      ↓
Application Instance
      ↓
Tomcat
```

---

## Test 2 — Application Login

Log into the application using the configured application credentials.

Successful login provides evidence that the application can access the database.

Conceptually:

```text
User
 ↓
VProfile
 ↓
Cloud SQL
 ↓
User Data
```

### What this proves

> The application-to-Cloud-SQL path is functioning.

---

## Test 3 — User Data

Open the application's user list.

Expected:

```text
User List
```

This provides another database-read validation.

---

## Test 4 — Cache Population

Open a user for the first time.

The expected behavior is that the application indicates that data was retrieved from the database and inserted into cache.

Conceptually:

```text
Application
    │
    ├──────► Cloud SQL
    │           │
    │           ▼
    │       User Data
    │
    └──────► Memorystore
                │
                ▼
             Cache
```

### What this proves

> Database access and cache population are functioning.

---

## Test 5 — Cache Retrieval

Open the same user again.

The expected behavior is that the application indicates the data came from cache.

Conceptually:

```text
Application
     │
     ▼
Memorystore
     │
     ▼
Cached Data
```

### What this proves

> Memorystore/Memcached connectivity and application caching behavior are functioning.

---

# 15. HTTPS Validation

## Objective

Confirm that the secure public access path works.

Expected flow:

```text
Browser
   │
   │ HTTPS
   ▼
Public DNS
   │
   ▼
Static LB IP
   │
   ▼
HTTPS Forwarding Rule
   │
   ▼
HTTPS Proxy
   │
   ▼
Certificate Map
   │
   ▼
URL Map
   │
   ▼
Backend Service
   │
   ▼
MIG
```

### Verify

- Certificate exists.
- Certificate is associated with the correct certificate map entry.
- DNS authorization is complete.
- HTTPS forwarding rule exists on port `443`.
- HTTPS proxy references the correct certificate configuration.
- Domain resolves to the load balancer IP.
- Browser establishes HTTPS successfully.
- Certificate is valid for the configured domain.

---

# 16. Public DNS Validation

The final public DNS path is:

```text
Application Domain
       │
       ▼
A Record
       │
       ▼
Static Load Balancer IP
       │
       ▼
Global HTTPS Load Balancer
```

### Verify

```bash
nslookup <application-domain>
```

Expected:

```text
<application-domain>
        ↓
Load Balancer Public IP
```

### What this proves

> Public DNS correctly maps the application hostname to the GCP load-balancing entry point.

---

# 17. End-to-End Validation

The strongest validation is the complete request and dependency path:

```text
                         USER
                           │
                           ▼
                    Public DNS
                           │
                           ▼
                 Static LB Public IP
                           │
                           ▼
                Global HTTPS LB
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
                        MIG
                           │
                           ▼
                 Private App Instance
                           │
                ┌──────────┴──────────┐
                ▼                     ▼
           Cloud SQL             Memorystore
             MySQL                 Memcached
```

A successful end-to-end validation should demonstrate:

1. Domain resolves.
2. HTTPS connection succeeds.
3. Certificate is valid.
4. Load balancer accepts the request.
5. Backend is healthy.
6. MIG provides the application instance.
7. VProfile loads.
8. Application login succeeds.
9. Database-backed data loads.
10. Cache behavior works.

---

# 18. Troubleshooting Decision Tree

When the application does not load:

```text
Application unavailable
        │
        ▼
Check DNS
        │
        ├── FAIL → Fix DNS
        │
        ▼
Check Load Balancer
        │
        ├── FAIL → Check forwarding rule / proxy / URL map
        │
        ▼
Check Backend Service
        │
        ├── FAIL → Check MIG attachment
        │
        ▼
Check MIG Health
        │
        ├── UNHEALTHY → Check Tomcat :8080
        │
        ▼
Check Application
        │
        ├── FAIL → Check application configuration
        │
        ▼
Check Cloud SQL
        │
        ├── FAIL → Check private connectivity / credentials
        │
        ▼
Check Memorystore
        │
        ├── FAIL → Check private connectivity / cache configuration
        │
        ▼
End-to-End Path Healthy
```

---

# 19. Evidence Strategy

Evidence should prove the engineering claims without turning the repository into a screenshot collection.

Recommended high-signal evidence:

| Evidence | What It Proves |
|---|---|
| VPC/subnet configuration | Network foundation |
| Bastion → private instance access | Private administrative path |
| Cloud SQL / Memorystore status | Backend services |
| Private DNS resolution | Service discovery |
| MIG health | Application fleet health |
| Load balancer configuration | Traffic routing |
| Certificate / HTTPS | TLS configuration |
| Final application over HTTPS | End-to-end deployment |
| Database-backed user data | Cloud SQL connectivity |
| First/second cache behavior | Memorystore connectivity |

Only personal execution evidence should be included in the repository.

---

# 20. Validation Evidence Matrix

| Claim | Validation | Evidence | Result |
|---|---|---|---|
| Custom VPC exists | VPC/subnet inspection | GCP CLI or console evidence | Record actual result |
| Private subnets exist | Subnet inspection | Subnet listing | Record actual result |
| Cloud NAT works | Private-instance outbound test | CLI/output evidence | Record actual result |
| Bastion access works | SSH test | Terminal screenshot | Record actual result |
| Cloud SQL is available | Instance status/connectivity | CLI/output evidence | Record actual result |
| Memorystore is available | Instance status/connectivity | CLI/output evidence | Record actual result |
| Private DNS works | `nslookup`/resolution | Terminal screenshot | Record actual result |
| App VM works | Tomcat/application test | Browser/terminal evidence | Record actual result |
| Custom image works | Instance creation from image | GCP evidence | Record actual result |
| MIG works | Instance/health inspection | GCP evidence | Record actual result |
| Health check works | Backend health inspection | GCP evidence | Record actual result |
| Load balancer works | Public endpoint test | Browser evidence | Record actual result |
| HTTPS works | Certificate/browser validation | Browser evidence | Record actual result |
| Public DNS works | DNS resolution | `nslookup`/browser | Record actual result |
| Database integration works | Login/user-list test | Application screenshot | Record actual result |
| Cache integration works | First/second access behavior | Application screenshot | Record actual result |

The **Result** column should only be changed to a confirmed result after the corresponding practical validation has actually been performed.

---

# 21. Validation Boundaries

Validation demonstrates that the configured infrastructure and application path work.

It does **not**, by itself, prove:

- Production-scale performance.
- Long-term reliability.
- Disaster recovery.
- Multi-region failover.
- Zero-downtime deployment.
- Security compliance.
- Enterprise IAM maturity.
- Production-grade observability.
- Proven auto-scaling behavior under sustained load.

Therefore:

> Successful functional validation should be described as **validated functionality**, not automatically as **production readiness**.

---

# 22. Cleanup Validation

After cleanup, verify that the intended resources have been removed.

The dependency-aware cleanup model is:

```text
HTTPS / DNS
     ↓
Load Balancer
     ↓
MIG
     ↓
Application Resources
     ↓
Backend Services
     ↓
PSA / Peering
     ↓
VPC
```

### Verify

- Load balancer resources removed.
- MIG removed.
- Temporary application resources removed.
- Cloud SQL removed if the project cleanup requires it.
- Memorystore removed if the project cleanup requires it.
- PSA resources handled.
- VPC resources removed where appropriate.

### Important GCP consideration

Private Service Access resources may not disappear immediately after deletion. The allocated range can remain reserved temporarily, which can delay deletion of related peering/VPC resources.

Therefore, cleanup validation should distinguish:

```text
Deletion requested
        ≠
Resource immediately gone
```

from:

```text
Resource successfully released
```

---

# 23. Final Validation Checklist

```text
NETWORK
[ ] Custom VPC verified
[ ] Public subnets verified
[ ] Private subnets verified
[ ] Cloud Router verified
[ ] Cloud NAT verified
[ ] Firewall rules verified

ACCESS
[ ] Bastion SSH verified
[ ] Private instance access verified

BACKEND
[ ] Cloud SQL active
[ ] Cloud SQL private IP verified
[ ] accounts database verified
[ ] Memorystore active
[ ] Private connectivity verified

DNS
[ ] Private DNS resolution verified
[ ] Backend names resolve to private addresses

APPLICATION
[ ] Tomcat :8080 verified
[ ] Application loads
[ ] Database connectivity verified
[ ] Cache connectivity verified

IMAGE / COMPUTE
[ ] Custom image verified
[ ] Instance Template verified
[ ] MIG verified
[ ] Instances healthy
[ ] Auto-scaling configuration verified

LOAD BALANCER
[ ] Health check verified
[ ] Backend service verified
[ ] URL map verified
[ ] Target proxy verified
[ ] Forwarding rule verified
[ ] Static IP verified

HTTPS
[ ] Certificate verified
[ ] Certificate map verified
[ ] DNS authorization verified
[ ] HTTPS endpoint verified

PUBLIC DNS
[ ] A record verified
[ ] Domain resolves to LB IP

END-TO-END
[ ] Login works
[ ] User data loads
[ ] First cache access works
[ ] Subsequent cache access works

CLEANUP
[ ] Intended resources deleted
[ ] PSA resources handled
[ ] VPC cleanup verified
```

---

# 24. Validation Conclusion

The project is considered **functionally validated** when the complete path works:

```text
Public DNS
    ↓
HTTPS
    ↓
Global Load Balancer
    ↓
Healthy MIG
    ↓
Private VProfile Instance
    ↓
Cloud SQL
    +
Memorystore
```

The strongest proof is not an individual `gcloud` command returning successfully.

The strongest proof is:

> **A user reaches the application through the public HTTPS domain, the load balancer routes the request to a healthy private application instance, the application successfully reads from Cloud SQL, and the application's subsequent access demonstrates Memcached behavior through Memorystore.**

---

[← Back to README](../README.md)
