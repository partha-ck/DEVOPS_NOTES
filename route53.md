# 🌐 AWS Route 53 — Complete Notes
A comprehensive guide to Amazon Route 53

DNS fundamentals · Hosted zones · Routing policies · Health checks · AWS integrations · Security · Troubleshooting · Interview preparation

---

## 📚 Table of Contents
1. [Introduction to Route 53](#1-introduction-to-route-53)
2. [DNS Fundamentals](#2-dns-fundamentals)
3. [How Route 53 Works](#3-how-route-53-works)
4. [Hosted Zones](#4-hosted-zones)
5. [DNS Record Types](#5-dns-record-types)
6. [Routing Policies](#6-routing-policies)
7. [Health Checks and Failover](#7-health-checks-and-failover)
8. [Route 53 with AWS Services](#8-route-53-with-aws-services)
9. [Private Hosted Zones](#9-private-hosted-zones)
10. [Domain Registration and Delegation](#10-domain-registration-and-delegation)
11. [TTL, Caching, and DNS Propagation](#11-ttl-caching-and-dns-propagation)
12. [Security and Best Practices](#12-security-and-best-practices)
13. [AWS CLI Commands](#13-aws-cli-commands)
14. [Troubleshooting Guide](#14-troubleshooting-guide)
15. [Real-World Architecture Examples](#15-real-world-architecture-examples)
16. [Interview Questions and Answers](#16-interview-questions-and-answers)
17. [Quick Revision Cheat Sheet](#17-quick-revision-cheat-sheet)

---

## 1. Introduction to Route 53

### What is Amazon Route 53?
Amazon Route 53 is a highly available and scalable Domain Name System (DNS) web service provided by AWS.

DNS translates human-readable domain names into IP addresses that computers use to communicate.

**Example:**
$$\text{www.example.com} \longrightarrow \text{203.0.113.10}$$

Instead of remembering an IP address, users can access an application through a domain name.

### Why is it called Route 53?
* **Route:** Directs DNS queries toward appropriate destinations.
* **53:** Refers to the standard DNS port.

DNS commonly uses UDP port 53 and TCP port 53.

### Key Features
* 🌐 Domain registration and DNS management.
* 🗂️ Public and private hosted zones.
* ⚖️ Multiple DNS routing policies.
* ❤️ DNS health checks and failover.
* 🌍 Geographic and latency-aware routing.
* ☁️ Integration with CloudFront, Elastic Load Balancing, S3, and other AWS services.
* 🔐 DNSSEC support for supported configurations.
* 🏢 Private DNS resolution within Amazon VPC environments.

### What Route 53 Is Not
* ❌ A web server.
* ❌ A proxy load balancer.
* ❌ A replacement for an application firewall.
* ❌ A service that automatically replicates application data.

> 💡 **Key takeaway:** Route 53 primarily answers DNS queries. The client then connects to the destination returned by DNS.

---

## 2. DNS Fundamentals

### What is DNS?
DNS (Domain Name System) is a distributed, hierarchical naming system that maps domain names to information such as IP addresses and mail servers.

$$\text{app.example.com} \longrightarrow \text{DNS resolution} \longrightarrow \text{203.0.113.10} \longrightarrow \text{Application server}$$

### DNS Hierarchy
```
                 Root (.)
                    |
         ---------------------
         |                   |
        .com                .org
         |
    example.com
         |
    ----------------
    |              |
   www            api
```

### Important DNS Terminology
| Term | Meaning |
| :--- | :--- |
| **Domain name** | A human-readable name, such as `example.com` |
| **DNS resolver** | Finds DNS answers on behalf of clients |
| **Authoritative DNS server** | Provides authoritative records for a zone |
| **DNS query** | A request for DNS information |
| **DNS response** | The answer to a DNS query |
| **TTL** | How long an answer may be cached |
| **FQDN** | Fully Qualified Domain Name |
| **DNS delegation** | Assigning authority for a domain to nameservers |
| **NS record** | Identifies authoritative nameservers |
| **Recursive lookup** | Resolver obtains an answer through additional queries |
| **Authoritative lookup** | Query directed to the DNS authority for the zone |

### Common DNS Ports
| Protocol | Port | Purpose |
| :--- | :--- | :--- |
| **UDP** | 53 | Most ordinary DNS queries |
| **TCP** | 53 | Large responses and other DNS operations |
| **HTTPS** | 443 | DNS-over-HTTPS, when supported |
| **TLS** | 853 | DNS-over-TLS, when supported |

⚠️ **Important:** Route 53's public authoritative DNS service is not a general-purpose DNS-over-HTTPS recursive resolver.

---

## 3. How Route 53 Works

### DNS Resolution Flow
```
User's browser 
      ↓
Operating system DNS cache 
      ↓
Recursive DNS resolver 
      ↓
Root DNS server 
      ↓
.com TLD DNS server 
      ↓
Route 53 authoritative nameserver 
      ↓
A / AAAA / Alias record 
      ↓
IP address or service destination 
      ↓
Browser connects to destination
```

### Step-by-Step Explanation
1. The browser needs the IP address for the requested hostname.
2. The operating system or configured resolver checks its cache.
3. If no usable cached answer exists, a recursive resolver performs the necessary lookups.
4. DNS delegation directs the resolver to the authoritative nameservers.
5. Route 53 returns the applicable DNS record.
6. The client connects to the returned destination.
7. The application processes the request.

> 💡 **Remember:** Cached answers can eliminate some or all intermediate DNS lookups. Route 53 does not normally proxy application traffic after DNS resolution.

### Example Architecture
$$\text{Internet user} \longrightarrow \text{www.example.com} \longrightarrow \text{Amazon Route 53} \longrightarrow \text{CloudFront CDN} \longrightarrow \text{Application origin} \longrightarrow \text{Database}$$

---

## 4. Hosted Zones

A hosted zone is a container for DNS records associated with a domain or subdomain.

```
Hosted zone: example.com

Records:
├── example.com
├── www.example.com
├── api.example.com
└── mail.example.com
```

### 4.1 Public Hosted Zone
A public hosted zone contains records used to resolve domain names on the public internet.

**Typical use cases:**
* Public websites.
* Public APIs.
* Public load balancers.
* CloudFront distributions.
* Public email routing.

*Important:* A public hosted zone does not make a private application publicly reachable by itself.

### 4.2 Private Hosted Zone
A private hosted zone contains DNS records resolvable from associated VPCs, subject to the relevant DNS configuration and connectivity.

```
VPC
 |
 +-- app.internal.example.com
 |
 +-- db.internal.example.com
 |
 +-- cache.internal.example.com
```

**Typical use cases:**
* Internal microservices.
* Private application endpoints.
* Database hostname resolution.
* Multi-VPC internal naming.
* Hybrid environments connected through VPN or Direct Connect with suitable DNS forwarding.

### Public vs. Private Hosted Zones
| Feature | Public | Private |
| :--- | :--- | :--- |
| **Resolution scope** | Public DNS | Associated VPCs and configured connected networks |
| **Primary purpose** | Internet-facing services | Internal services |
| **VPC association required** | No | Yes |
| **Public delegation** | Relevant | Not used for public delegation |
| **Example** | `www.example.com` | `db.internal.example.com` |

### Important Hosted-Zone Records
* **SOA (Start of Authority):** Contains zone metadata and authoritative DNS information.
* **NS (Name Server):** Identifies the authoritative nameservers for a zone.
* **Hosted-zone ID:** Identifies the zone in AWS CLI and API operations.

Route 53 creates NS and SOA records when a hosted zone is created. Preserve these records unless you have a specific, correctly configured DNS requirement.

---

## 5. DNS Record Types

### 5.1 A Record — IPv4
Maps a hostname to an IPv4 address.
$$\text{example.com} \quad \text{A} \quad \text{203.0.113.10}$$

### 5.2 AAAA Record — IPv6
Maps a hostname to an IPv6 address.
$$\text{example.com} \quad \text{AAAA} \quad \text{2001:db8::10}$$

### 5.3 CNAME Record — Hostname Alias
Maps one hostname to another hostname.
$$\text{www.example.com} \quad \text{CNAME} \quad \text{example.com}$$

**Important restrictions:**
* A CNAME cannot coexist with other record types at the same owner name.
* An ordinary CNAME cannot be created at the zone apex, such as `example.com`.
* CNAME targets are hostnames, not IP addresses.

### 5.4 Alias Record
A Route 53 Alias record can point to supported AWS resources and certain supported DNS record targets.

**Common targets include:**
* CloudFront distributions.
* Elastic Load Balancing load balancers.
* Amazon S3 website endpoints.
* Other supported AWS resource targets.
* Certain records within Route 53.

$$\text{example.com} \quad \text{A (Alias)} \quad \text{CloudFront distribution}$$

### Alias vs. CNAME
| Feature | Alias | CNAME |
| :--- | :--- | :--- |
| **Route 53-specific** | Yes | No |
| **Zone apex support** | Yes, for eligible targets and configurations | No |
| **Target** | Supported AWS resources or supported records | Another hostname |
| **Coexists with other record types** | Depends on record type and configuration | No |
| **Arbitrary external hostname** | Not as a general-purpose replacement | Yes |

> 💡 **Best practice:** Consider an Alias record for supported AWS resources, especially when mapping a zone apex to an AWS endpoint. Verify the supported target and applicable pricing rules.

### 5.5 MX Record — Mail Servers
$$\text{example.com} \quad \text{MX} \quad 10 \quad \text{mail.example.com}$$
Lower MX priority values indicate a higher preference.

### 5.6 TXT Record — Text Data
Common uses include domain ownership verification, SPF email sender policy, DKIM public-key information, DMARC policy, and service verification.
$$\text{example.com} \quad \text{TXT} \quad \text{"v=spf1 include:example.net -all"}$$

### 5.7 NS Record — Nameservers
Identifies authoritative nameservers and supports domain or subdomain delegation.

### 5.8 PTR Record — Reverse DNS
Maps an IP address to a hostname through reverse DNS.

### 5.9 SRV Record — Service Discovery
Specifies service priority, weight, port, and target.

### 5.10 CAA Record — Certificate Authority Restrictions
Specifies which certificate authorities are authorized to issue TLS certificates for a domain.
$$\text{example.com} \quad \text{CAA} \quad 0 \quad \text{issue "letsencrypt.org"}$$

⚠️ **Note:** CAA records do not replace certificate validation or renewal management.

---

## 6. Routing Policies

Route 53 routing policies determine which DNS answer is returned for a query.

| Policy | Primary purpose |
| :--- | :--- |
| **Simple** | Straightforward DNS answer |
| **Weighted** | Distribution according to configured weights |
| **Latency-based** | Lower-latency AWS Region |
| **Failover** | Primary or secondary answer based on health |
| **Geolocation** | Route by geographic location |
| **Geoproximity** | Route by location and configured bias |
| **Multivalue answer** | Return multiple eligible record values |
| **IP-based** | Route using source IP ranges and CIDR mappings |

### 6.1 Simple Routing
Used for a straightforward DNS response.
$$\text{www.example.com} \longrightarrow 203.0.113.10$$
**Use cases:** Simple websites, single endpoints, and basic hostname-to-address mappings.

### 6.2 Weighted Routing
Distributes DNS responses according to configured relative weights.
* **Version A:** Weight 80
* **Version B:** Weight 20

**Use cases:** Blue-green deployments, canary releases, gradual migrations, testing new application versions.

⚠️ **Important:** DNS weights do not guarantee exact percentages of application requests. Caching and client behavior influence actual traffic.

### 6.3 Latency-Based Routing
Returns an answer associated with an AWS Region expected to provide lower network latency (`us-east-1` vs `ap-south-1`).
**Use cases:** Global applications, multi-Region architectures, and latency-sensitive services.

### 6.4 Failover Routing
Directs DNS responses toward a primary endpoint and a secondary endpoint based on health checks.
**Use cases:** Disaster recovery, multi-Region failover, and standby application environments.

### 6.5 Geolocation Routing
Routes queries based on the geographic origin of the DNS query as determined by Route 53 (India, Europe, North America). Configure a default record when appropriate.

### 6.6 Geoproximity Routing
Routes based on the geographic location of users and resources, with optional bias to expand or shrink the effective routing area.

### 6.7 Multivalue Answer Routing
Returns multiple eligible values in a DNS response. Health-check evaluation can be used for supported configurations.

### 6.8 IP-Based Routing
Routes queries according to source IP ranges and configured CIDR collection mappings.

> 💡 **Exam tip:** Understand the difference between weighted, latency-based, failover, geolocation, and geoproximity routing. Their purposes are distinct.

---

## 7. Health Checks and Failover

### What Are Route 53 Health Checks?
Health checks evaluate whether configured endpoints or resources are healthy according to supported checks and settings.

### Health-Check Types
* **Endpoint health checks:** Monitor an endpoint using supported protocols such as HTTP, HTTPS, or TCP.
* **Calculated health checks:** Combine the status of other health checks.
* **CloudWatch alarm health checks:** Use a supported CloudWatch alarm's state.

### Important Considerations
* Endpoint health checks do not automatically verify every dependency.
* Private endpoints may require a different health-monitoring design.
* Health-check success does not guarantee that the entire application works.
* DNS failover is affected by TTLs and resolver caching.

> 🔑 **Key takeaway:** DNS failover changes the answer available to resolvers. It does not replicate databases, create standby infrastructure, or terminate existing connections.

---

## 8. Route 53 with AWS Services

### 8.1 Route 53 + Application Load Balancer
$$\text{Users} \longrightarrow \text{Route 53} \longrightarrow \text{Application Load Balancer} \longrightarrow \text{Target group} \longrightarrow \text{EC2 / containers}$$
* **Recommended configuration:** Create an Alias record for the supported load balancer target, configure listeners, target groups, TLS certificates, security groups, and application health checks.

### 8.2 Route 53 + CloudFront
$$\text{User} \longrightarrow \text{Route 53} \longrightarrow \text{CloudFront} \longrightarrow \text{Origin: S3 / ALB / application}$$
* **Benefits:** Global content delivery, edge caching, HTTPS delivery, and application acceleration. Configure alternate domain names and TLS certificates.

### 8.3 Route 53 + S3 Static Website
$$\text{Browser} \longrightarrow \text{Route 53} \longrightarrow \text{S3 website endpoint}$$
⚠️ **Important:** An S3 website endpoint does not itself provide HTTPS. CloudFront in front of S3 is standard for production HTTPS websites. S3 website endpoints and S3 REST endpoints are distinct endpoint types.

### 8.4 Route 53 + EC2
A DNS record can point to a suitable EC2 public IP address or another supported destination. Prefer a stable endpoint like a load balancer when instance IPs change.

### 8.5 Route 53 + VPC
Important VPC DNS settings include `enableDnsSupport` and `enableDnsHostnames`.

### 8.6 Route 53 Resolver
* **Inbound endpoint:** Allows connected networks to send DNS queries into a VPC.
* **Outbound endpoint:** Forwards DNS queries from a VPC to configured external DNS servers through resolver rules.

---

## 9. Private Hosted Zones

Private hosted zones allow applications to use meaningful internal DNS names without publishing those records through public authoritative DNS.

### Multi-VPC Design
```
       Private hosted zone
       internal.example.com
                 |
     -------------------------
     |                       |
   VPC A                   VPC B
     |                       |
Application               Services
```

### Split-Horizon DNS
The same hostname can return different answers depending on the resolver context:
* **Public DNS:** `app.example.com` $\longrightarrow$ public endpoint
* **Private DNS:** `app.example.com` $\longrightarrow$ private endpoint

### Common Mistakes
* Forgetting to associate the hosted zone with the required VPC.
* Overlapping private hosted zones and resolver rules.
* Misconfiguring VPC DNS settings.
* Forgetting to configure hybrid DNS forwarding.
* Assuming DNS resolution automatically provides network connectivity.

---

## 10. Domain Registration and Delegation

### Domain vs. Hosted Zone
| Concept | Responsibility |
| :--- | :--- |
| **Domain registration** | Registers and manages a domain name |
| **Hosted zone** | Stores DNS records |
| **Registrar configuration** | Identifies the authoritative nameservers |
| **DNS record** | Defines the answer for a hostname or service |

### Nameserver Delegation
$$\text{Domain registrar} \longrightarrow \text{Nameserver delegation} \longrightarrow \text{Route 53 public hosted zone} \longrightarrow \text{DNS records}$$
If the registrar points to another DNS provider, the Route 53 hosted zone may exist without being authoritative for public queries.

### Subdomain Delegation
Create the child hosted zone for `dev.example.com`, then configure the corresponding NS records in the parent zone.

---

## 11. TTL, Caching, and DNS Propagation

### What Is TTL?
TTL (Time to Live) specifies how long a DNS response may be cached.
$$\text{api.example.com} \quad \text{A} \quad \text{203.0.113.10} \quad (\text{TTL} = 300\text{ seconds})$$

### How TTL Affects DNS Changes
* **Before:** `api.example.com` $\longrightarrow$ `203.0.113.10`
* **After:** `api.example.com` $\longrightarrow$ `203.0.113.20`

Resolvers that cached the old answer may continue using it until the cached entry expires. Lowering a TTL does not retroactively shorten an already cached answer.

### Recommended TTL Strategy
| Scenario | Approach |
| :--- | :--- |
| **Stable records** | Use a reasonable TTL |
| **Planned migration** | Lower TTL sufficiently in advance |
| **Frequently changing destinations** | Consider lower TTLs |
| **High-volume DNS workloads** | Balance freshness with caching efficiency |

---

## 12. Security and Best Practices

1. **Apply Least Privilege with IAM:** Limit who can create hosted zones, change DNS records, modify health checks, and configure resolver rules.
2. **Protect Domain Registration:** Enable MFA, use domain transfer locks, keep registration information current, and monitor renewals.
3. **Use DNSSEC Appropriately:** Adds cryptographic validation to DNS data and helps protect against tampering. (Does not encrypt queries or replace HTTPS).
4. **Use Private Hosted Zones for Internal Names:** Helps control resolution scope, but not a substitute for network access controls.
5. **Monitor DNS Changes:** Use AWS monitoring, CloudTrail logging, alerting, and operational reviews.
6. **Configure Meaningful Health Checks:** Test the application's actual ability to serve requests.
7. **Plan DNS Migrations:** Verify target hosted zones, NS delegation, record values, TTLs, certificates, and endpoint configurations.

---

## 13. AWS CLI Commands

Prerequisites: Install and configure AWS CLI, select the appropriate AWS account, and ensure required IAM permissions are available.

### List Hosted Zones
```bash
aws route53 list-hosted-zones
```

### Get Hosted-Zone Details
```bash
aws route53 get-hosted-zone --id Z123456789EXAMPLE
```

### List DNS Records
```bash
aws route53 list-resource-record-sets --hosted-zone-id Z123456789EXAMPLE
```

### Create a Hosted Zone
```bash
aws route53 create-hosted-zone \
  --name example.com \
  --caller-reference "example-com-20261026"
```

### Create or Update an A Record
Create `change-record.json`:
```json
{
  "Comment": "Create or update an A record",
  "Changes": [
    {
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "app.example.com",
        "Type": "A",
        "TTL": 300,
        "ResourceRecords": [
          {
            "Value": "203.0.113.10"
          }
        ]
      }
    }
  ]
}
```
Apply the change:
```bash
aws route53 change-resource-record-sets \
  --hosted-zone-id Z123456789EXAMPLE \
  --change-batch file://change-record.json
```

### Check Change Status
```bash
aws route53 get-change --id CHANGE_ID
```
*(Status `INSYNC` indicates the change has propagated across Route 53's authoritative infrastructure).*

### Delete a Record
```json
{
  "Changes": [
    {
      "Action": "DELETE",
      "ResourceRecordSet": {
        "Name": "app.example.com",
        "Type": "A",
        "TTL": 300,
        "ResourceRecords": [
          {
            "Value": "203.0.113.10"
          }
        ]
      }
    }
  ]
}
```

### Additional Useful Commands
```bash
aws route53 list-health-checks
aws route53 list-traffic-policies

dig example.com A
dig example.com +trace
dig @1.1.1.1 example.com
```

---

## 14. Troubleshooting Guide

* **Problem 1: Domain Is Not Resolving**
  * *Causes:* Incorrect nameserver delegation, missing DNS record, incorrect hosted-zone configuration, cached negative DNS response.
  * *Commands:* `dig example.com`, `dig example.com NS`, `dig +trace example.com`
* **Problem 2: DNS Returns the Old IP Address**
  * *Causes:* Cached response, unexpired TTL, OS cache, wrong hosted zone.
  * *Commands:* `dig @1.1.1.1 app.example.com`, `dig @8.8.8.8 app.example.com`
* **Problem 3: Alias Record Does Not Work**
  * *Check:* Target support, record configuration, resource reachability, and domain delegation.
* **Problem 4: Private Hosted Zone Does Not Resolve**
  * *Check:* VPC association, VPC DNS support, query origin, resolver rules, and hybrid forwarding.
* **Problem 5: Failover Is Not Occurring**
  * *Check:* Health-check status, endpoint accessibility, protocol/path, failure thresholds, and cached DNS answers.

---

## 15. Real-World Architecture Examples

### Architecture 1: Highly Available Web Application
$$\text{Users} \longrightarrow \text{Route 53} \longrightarrow \text{Application Load Balancer} \longrightarrow (\text{EC2 / ECS Clusters}) \longrightarrow \text{Data Layer}$$

### Architecture 2: Multi-Region Disaster Recovery
$$\text{Users} \longrightarrow \text{Route 53 (Failover)} \longrightarrow \begin{cases} \text{Region A (Primary App)} \longleftrightarrow \text{Data Replication} \\ \text{Region B (Standby App)} \end{cases}$$

### Architecture 3: Global Application with Latency Routing
$$\text{Global Users} \longrightarrow \text{Route 53 (Latency Routing)} \longrightarrow \begin{cases} \text{US Region} \\ \text{India Region} \end{cases}$$

### Architecture 4: Hybrid DNS
$$\text{On-Premises Network} \longrightarrow \text{Corporate DNS} \longrightarrow \text{VPN / Direct Connect} \longrightarrow \text{Route 53 Resolver} \longrightarrow \text{AWS VPC / Private Zones}$$

---

## 16. Interview Questions and Answers

### 🟢 Beginner Level
1. **What is Amazon Route 53?** An AWS DNS service used for domain registration, DNS record management, and traffic routing.
2. **Why is it called Route 53?** Refers to standard DNS port 53.
3. **What is a hosted zone?** A collection of DNS records associated with a domain or subdomain.
4. **What is the difference between public and private hosted zones?** Public serves public DNS resolution; private serves associated VPC environments.
5. **What is TTL?** Duration for which a DNS answer may be cached.
6. **Can you create a CNAME at the root domain?** No. Use an Alias record for eligible Route 53 targets at the zone apex.

### 🟡 Intermediate Level
7. **What is weighted routing?** Routing DNS responses according to configured relative weights.
8. **What is latency-based routing?** Routing DNS queries toward a regional resource based on latency-routing measurements.
9. **Geolocation vs. geoproximity routing?** Geolocation routes by geographic origin of query; geoproximity routes by user/resource locations with optional geographic bias.
10. **What is the difference between Route 53 and a load balancer?** Route 53 resolves names and steers answers; an ALB distributes application traffic among targets.

### 🔴 Advanced Level
11. **How would you design disaster recovery using Route 53?** Configure DNS failover, meaningful health evaluations, secondary environments, data replication, and account for RTO, RPO, and TTLs.
12. **What is split-horizon DNS?** The same hostname returns different DNS answers depending on the resolver context (public vs private).
13. **What is DNSSEC?** Uses cryptographic signatures to support validation of DNS data authenticity and integrity (does not encrypt queries).

---

## 17. Quick Revision Cheat Sheet

### Record Types
| Record | Purpose |
| :--- | :--- |
| **A** | IPv4 address |
| **AAAA** | IPv6 address |
| **CNAME** | Alias hostname to another hostname |
| **Alias** | Route 53 mapping to supported AWS resources/records |
| **MX** | Mail servers |
| **TXT** | Verification and policy data |
| **NS** | Authoritative nameservers |

### Routing Policies
| Requirement | Policy |
| :--- | :--- |
| **Simple DNS response** | Simple |
| **Gradual traffic shift** | Weighted |
| **Lower-latency regional destination** | Latency-based |
| **Primary and secondary endpoints** | Failover |
| **Geographic routing** | Geolocation |
| **Multiple eligible answers** | Multivalue answer |

### ⭐ Five Concepts to Remember
1. Route 53 is DNS, not a proxy load balancer.
2. Public and private hosted zones have different resolution scopes.
3. Alias records support zone-apex routing for eligible targets; ordinary CNAME records cannot.
4. Routing policies influence DNS answers, not guaranteed application-request percentages.
5. DNS failover, TTLs, application health, and data replication are separate concerns.

---
*End of AWS Route 53 Complete Notes.*
