# Azure Private Endpoint & Private DNS — Secure Azure Blob Storage

## Project Overview

This lab demonstrates how to secure an Azure Storage Account using **Azure Private Endpoint, Azure Private Link, and Azure Private DNS**.

The goal was to remove public network access to Azure Blob Storage while allowing workloads inside an Azure Virtual Network to continue accessing the storage service through a private IP address.

The lab also validates the design from both sides:

- An Azure VM inside the VNet successfully accessed a private blob through the Private Endpoint.
- A workstation outside Azure was denied access after public network access was disabled.
- The same Blob SAS URL was used for both tests so that authentication remained consistent while the network path changed.

---

## Architecture

```text
                         MICROSOFT AZURE
┌───────────────────────────────────────────────────────────────────┐
│                                                                   │
│              vnet-private-endpoint-lab                            │
│                                                                   │
│   ┌──────────────────────┐                                        │
│   │   vm-private-test    │                                        │
│   │                      │                                        │
│   │ Azure VM             │                                        │
│   │ 10.50.1.x            │                                        │
│   └──────────┬───────────┘                                        │
│              │                                                    │
│              │ DNS query                                          │
│              ▼                                                    │
│   ┌──────────────────────────────────────────────┐                 │
│   │ Azure Private DNS Zone                       │                 │
│   │                                              │                 │
│   │ privatelink.blob.core.windows.net            │                 │
│   │                                              │                 │
│   │ stprivatelab2026 → 10.50.1.5                 │                 │
│   └──────────────────────┬───────────────────────┘                 │
│                          │                                        │
│                          ▼                                        │
│                ┌────────────────────┐                              │
│                │ Private Endpoint   │                              │
│                │                    │                              │
│                │ 10.50.1.5          │                              │
│                └─────────┬──────────┘                              │
│                          │                                        │
│                          │ Azure Private Link                      │
│                          ▼                                        │
│                ┌────────────────────┐                              │
│                │ Azure Blob Storage │                              │
│                │                    │                              │
│                │ stprivatelab2026   │                              │
│                │                    │                              │
│                │ Public Network     │                              │
│                │ Access: DISABLED   │                              │
│                └────────────────────┘                              │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘


                 PUBLIC INTERNET

┌──────────────────────┐
│ Local Workstation    │
│                      │
│ Public DNS           │
└──────────┬───────────┘
           │
           │ HTTPS
           ▼
   Public Storage Endpoint
           │
           X
      BLOCKED
   HTTP 403 Forbidden
```

---

## Traffic Flow

### Private Azure VM

```text
vm-private-test
      |
      | DNS lookup
      v
stprivatelab2026.blob.core.windows.net
      |
      | CNAME
      v
stprivatelab2026.privatelink.blob.core.windows.net
      |
      | Private DNS A record
      v
10.50.1.5
      |
      v
Private Endpoint
      |
      | Azure Private Link
      v
Azure Blob Storage
      |
      v
HTTP 200 OK
```

### External Workstation

```text
Local Workstation
      |
      | Public DNS
      v
Public Azure Storage IP
      |
      | HTTPS
      v
Azure Storage Public Endpoint
      |
      X
Public Network Access Disabled
      |
      v
HTTP 403
```

---

# Technologies Used

- Microsoft Azure
- Azure Virtual Network
- Azure Virtual Machines
- Azure Storage Account
- Azure Blob Storage
- Azure Private Endpoint
- Azure Private Link
- Azure Private DNS
- Azure DNS
- SAS (Shared Access Signature)
- Linux
- `nslookup`
- `curl`

---

# Lab Objectives

The objectives of this project were to:

1. Deploy an Azure Storage Account.
2. Establish baseline public DNS resolution.
3. Deploy an Azure VNet and test VM.
4. Create a Private Endpoint for Azure Blob Storage.
5. Assign the Storage service a private endpoint IP inside the VNet.
6. Configure Azure Private DNS.
7. Link the Private DNS Zone to the VNet.
8. Verify split DNS behavior between Azure and an external workstation.
9. Disable public network access to the Storage Account.
10. Validate HTTPS connectivity through the Private Endpoint.
11. Create a private Blob container and test file.
12. Generate a restricted SAS URL.
13. Test the same authenticated Blob request from inside and outside Azure.
14. Confirm private access succeeds while public access is denied.

---

# Step 1 — Create Azure Storage Account

An Azure Storage Account was deployed for the lab.

Example:

```text
Storage Account:
stprivatelab2026

Service:
Azure Blob Storage
```

Initially, public network access was enabled so that baseline connectivity and DNS behavior could be observed.

---

# Step 2 — Establish Public DNS Baseline

Before deploying the Private Endpoint, DNS resolution was tested from the Azure VM.

```bash
nslookup stprivatelab2026.blob.core.windows.net
```

The Storage hostname resolved through Microsoft's public Storage infrastructure.

Example result:

```text
stprivatelab2026.blob.core.windows.net
        |
        v
Microsoft Azure Storage public endpoint
        |
        v
57.150.26.33
```

This established the **before Private Endpoint** baseline.

---

# Step 3 — Deploy the Virtual Network

A dedicated VNet was created for the Private Endpoint lab.

Example architecture:

```text
vnet-private-endpoint-lab
        |
        +--- snet-client
                |
                +--- vm-private-test
                |
                +--- Private Endpoint
```

The test VM and Private Endpoint were placed inside the Azure network.

---

# Step 4 — Deploy Test VM

A Linux VM was deployed into the VNet.

The VM was used to simulate an internal Azure workload that needed private access to the Storage Account.

Connectivity tools used during validation included:

```bash
nslookup
curl
```

---

# Step 5 — Create the Storage Private Endpoint

A Private Endpoint was created for the Storage Account.

Configuration:

```text
Resource:
stprivatelab2026

Target sub-resource:
blob

Virtual Network:
vnet-private-endpoint-lab

Subnet:
snet-client

Private IP allocation:
Dynamic
```

Azure assigned the Private Endpoint:

```text
10.50.1.5
```

This IP exists inside the Azure VNet.

The Storage Account itself was not moved into the VNet.

Instead, **Private Link exposed the Storage service through a private endpoint NIC inside the VNet**.

---

# Step 6 — Configure Azure Private DNS

During Private Endpoint deployment, Private DNS integration was enabled.

Azure created:

```text
privatelink.blob.core.windows.net
```

This Private DNS Zone is specifically used for Azure Blob Storage Private Link endpoints.

---

# Step 7 — Private DNS A Record

Azure automatically created an A record:

```text
stprivatelab2026
        |
        v
10.50.1.5
```

Full private hostname:

```text
stprivatelab2026.privatelink.blob.core.windows.net
```

The resulting DNS relationship became:

```text
stprivatelab2026.blob.core.windows.net
        |
        | CNAME
        v
stprivatelab2026.privatelink.blob.core.windows.net
        |
        | A record
        v
10.50.1.5
```

---

# Step 8 — Link Private DNS Zone to the VNet

The Private DNS Zone was linked to:

```text
vnet-private-endpoint-lab
```

Configuration:

```text
Link Status:
Completed

Auto-registration:
Disabled
```

Auto-registration was not required because the zone was being used for a Private Link service rather than automatic VM hostname registration.

The VNet link allows resources inside the VNet to resolve records contained in the Private DNS Zone.

---

# Step 9 — Validate Private DNS Resolution

From `vm-private-test`:

```bash
nslookup stprivatelab2026.blob.core.windows.net
```

The result changed from the original public IP to:

```text
stprivatelab2026.blob.core.windows.net
        |
        | CNAME
        v
stprivatelab2026.privatelink.blob.core.windows.net
        |
        v
10.50.1.5
```

This confirmed that the Azure VM was now resolving the normal Storage hostname to the Private Endpoint.

---

# Step 10 — Compare DNS from Outside Azure

The same lookup was performed from a local workstation:

```powershell
nslookup stprivatelab2026.blob.core.windows.net
```

The workstation resolved the hostname through public Azure DNS infrastructure rather than to:

```text
10.50.1.5
```

This demonstrated the difference between DNS resolution inside the linked Azure VNet and DNS resolution from the public Internet.

Conceptually:

```text
Azure VM
   |
   +--> Private DNS --> 10.50.1.5


Local PC
   |
   +--> Public DNS --> Azure Storage public endpoint
```

---

# Step 11 — Disable Storage Public Network Access

After the Private Endpoint and DNS configuration were validated, public network access was disabled on the Storage Account.

Configuration:

```text
Public network access:
Disabled
```

The intended security model became:

```text
Internet
   |
   X
Public Storage access


Azure VNet
   |
   v
Private Endpoint
   |
   v
Storage Account
```

---

# Step 12 — Test HTTPS Connectivity

HTTPS connectivity was tested from the Azure VM:

```bash
curl -I https://stprivatelab2026.blob.core.windows.net
```

Azure Storage returned an HTTP response.

Receiving a Storage service response confirmed that TCP/HTTPS connectivity to the Storage service existed.

However, an HTTP response alone was not considered sufficient proof of the complete design because application authorization and network connectivity are separate controls.

A stronger validation test was therefore performed.

---

# Step 13 — Create Private Blob

A Blob container was created:

```text
private-test
```

A test file was uploaded:

```text
private-test.txt
```

Example contents:

```text
test lab
```

The Blob remained private.

---

# Step 14 — Generate Read-Only SAS

A Shared Access Signature (SAS) was generated for the test Blob.

Permissions were restricted to:

```text
Read
```

HTTPS was required and the SAS was configured with a short expiration period.

A SAS provides temporary authorization to a Storage resource without distributing the Storage Account key.

Conceptually:

```text
Blob URL
+
Temporary SAS credential
=
Authorized Blob request
```

> **Security Note:** SAS tokens are credentials and must never be committed to GitHub, included in screenshots, or shared publicly.

---

# Step 15 — Final Private vs Public Validation

The exact same Blob, SAS authorization, and HTTPS operation were tested from both machines.

This was important because it removed authentication as the variable.

## Test A — Azure VM

From the Azure VM:

```bash
curl -i "<BLOB-SAS-URL>"
```

Result:

```text
HTTP/1.1 200 OK
Content-Type: text/plain

test lab
```

The VM's network path was:

```text
VM
 |
 v
Private DNS
 |
 v
10.50.1.5
 |
 v
Private Endpoint
 |
 v
Azure Private Link
 |
 v
Blob Storage
 |
 v
200 OK
```

**Result: SUCCESS**

---

## Test B — External Workstation

The same SAS URL was tested from the local Windows workstation:

```powershell
curl.exe -i "<SAME-BLOB-SAS-URL>"
```

Result:

```text
HTTP/1.1 403
x-ms-error-code: AuthorizationFailure
```

The workstation was attempting to reach the Storage service through its public endpoint while public network access was disabled.

**Result: DENIED**

---

# Final Validation Results

| Test | Azure VM | External PC |
|---|---|---|
| DNS path | Private | Public |
| Storage destination | `10.50.1.5` | Public Azure endpoint |
| Private Endpoint | Yes | No |
| Same SAS credential | Yes | Yes |
| HTTPS request | Allowed | Denied |
| Result | `200 OK` | `403` |

The test demonstrated that authorization alone does not bypass Storage network controls.

The same valid SAS credential behaved differently depending on the network path.

---

# Key Concepts Learned

## Private Endpoint

A Private Endpoint creates a network interface with a private IP address inside an Azure VNet for an Azure PaaS service.

In this lab:

```text
Azure Blob Storage
        |
        | Private Link
        v
Private Endpoint
10.50.1.5
```

---

## Azure Private Link

Private Link provides the underlying Azure platform connectivity between the Private Endpoint and the Azure PaaS service.

The Storage service does not need to be deployed directly inside the VNet.

---

## Private DNS Zone

The Private DNS Zone maps the Private Link hostname to the Private Endpoint IP.

```text
privatelink.blob.core.windows.net
```

contained:

```text
stprivatelab2026 → 10.50.1.5
```

---

## VNet Link

The VNet link makes the Private DNS Zone available to resources inside:

```text
vnet-private-endpoint-lab
```

Without the correct DNS architecture, clients may continue resolving the Storage service to its public endpoint even though a Private Endpoint exists.

---

## SAS

A Shared Access Signature provides delegated, time-limited access to Azure Storage.

For the final test, SAS ensured that both machines had equivalent Blob authorization.

This allowed the lab to isolate **network location** as the primary difference between the successful and unsuccessful requests.

---

# Security Architecture

The completed architecture followed this model:

```text
                    AUTHORIZED PRIVATE PATH

Azure VM
   |
   | HTTPS
   v
Private Endpoint
10.50.1.5
   |
   | Private Link
   v
Azure Blob Storage
   |
   v
200 OK


                    PUBLIC PATH

External Workstation
   |
   | Internet
   v
Azure Storage Public Endpoint
   |
   X
Public Network Access Disabled
   |
   v
403
```

---

# Troubleshooting Performed

Several validation steps were intentionally used rather than assuming the Private Endpoint was working.

### Public DNS still returned a public IP

Before Private DNS integration, the Storage hostname resolved to Azure's public Storage infrastructure.

Resolution:

- Created the Blob Private Endpoint.
- Created `privatelink.blob.core.windows.net`.
- Verified the Private DNS A record.
- Verified the VNet link.

---

### HTTP 400 during initial curl test

An initial request was made against the root Blob endpoint:

```bash
curl -I https://stprivatelab2026.blob.core.windows.net
```

The Storage service returned an HTTP error because the request was not a complete Blob operation.

This was useful for connectivity troubleshooting but was not treated as final application-level validation.

The lab therefore progressed to an actual Blob request using a valid SAS.

---

### Azure Portal unable to browse Blob container

After public network access was disabled, the local workstation could no longer browse Blob data through the Azure Portal.

This was expected because the browser was outside the Azure VNet and could not use the Private Endpoint.

The portal displayed a `403` data-plane access failure.

---

# Real-World Use Case

This architecture is commonly used when workloads such as:

```text
Application VMs
AKS workloads
Azure Functions
App Services
Database servers
Backup systems
Internal enterprise applications
```

need access to Azure PaaS services without exposing those services through public network access.

The same Private Endpoint design can be applied to services such as:

- Azure Storage
- Azure SQL
- Azure Key Vault
- Azure Container Registry
- Azure Cosmos DB
- Other Private Link-supported Azure services

---

# Important Design Lesson

Creating a Private Endpoint alone is not enough.

A complete design requires understanding:

```text
Private Endpoint
      +
Private Link
      +
Private DNS Zone
      +
VNet Link
      +
Correct DNS resolution
      +
Public network restrictions
      +
Application-level validation
```

The DNS component is especially important because applications normally connect using the service's hostname rather than manually connecting to the Private Endpoint IP.

---

# Project Outcome

The completed lab successfully demonstrated:

- Azure Blob Storage Private Endpoint deployment
- Private IP assignment (`10.50.1.5`)
- Azure Private DNS integration
- Private DNS A-record creation
- VNet-to-Private-DNS linking
- Public vs private DNS behavior
- Storage public network access restriction
- Private HTTPS connectivity
- SAS-based Blob authorization
- Real Blob data-plane testing
- Successful `200 OK` from inside Azure
- Failed `403` access from outside the private network

The result was a Storage architecture where workloads inside the Azure VNet could securely access Blob Storage through Azure Private Link while public network access remained disabled.

---

## Validation Summary

```text
               SAME STORAGE ACCOUNT
                    SAME BLOB
                    SAME SAS
                       |
             +---------+---------+
             |                   |
             v                   v

         AZURE VM           LOCAL WORKSTATION
             |                   |
        Private DNS          Public DNS
             |                   |
             v                   v
         10.50.1.5        Public Storage IP
             |                   |
             v                   v
     Private Endpoint       Public Endpoint
             |                   |
             v                   X
       Private Link        Network Restricted
             |                   |
             v                   v
        Blob Storage          DENIED
             |
             v
          200 OK

          SUCCESS              403
```

---



---

## Author

Hands-on Azure networking project focused on understanding and validating secure private connectivity to Azure PaaS services.
