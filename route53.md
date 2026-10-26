🌐 AWS Route 53 — Complete Notes
A comprehensive guide to Amazon Route 53

DNS fundamentals · Hosted zones · Routing policies · Health checks · AWS integrations · Security · Troubleshooting · Interview preparation

!AWS !Level !Format

📚 Table of Contents
Introduction to Route 53
DNS Fundamentals
How Route 53 Works
Hosted Zones
DNS Record Types
Routing Policies
Health Checks and Failover
Route 53 with AWS Services
Private Hosted Zones
Domain Registration and Delegation
TTL, Caching, and DNS Propagation
Security and Best Practices
AWS CLI Commands
Troubleshooting Guide
Real-World Architecture Examples
Interview Questions and Answers
Quick Revision Cheat Sheet
1. Introduction to Route 53
What is Amazon Route 53?

Amazon Route 53 is a highly available and scalable Domain Name System (DNS) web service provided by AWS.

DNS translates human-readable domain names into IP addresses that computers use to communicate.

Example:

text
www.example.com  →  203.0.113.10


Instead of remembering an IP address, users can access an application through a domain name.

Why is it called Route 53?
Route: Directs DNS queries toward appropriate destinations.
53: Refers to the standard DNS port.

DNS commonly uses UDP port 53 and TCP port 53.

Key Features
🌐 Domain registration and DNS management.
🗂️ Public and private hosted zones.
⚖️ Multiple DNS routing policies.
❤️ DNS health checks and failover.
🌍 Geographic and latency-aware routing.
☁️ Integration with CloudFront, Elastic Load Balancing, S3, and other AWS services.
🔐 DNSSEC support for supported configurations.
🏢 Private DNS resolution within Amazon VPC environments.
What Route 53 Is Not
❌ A web server.
❌ A proxy load balancer.
❌ A replacement for an application firewall.
❌ A service that automatically replicates application data.
💡 Key takeaway: Route 53 primarily answers DNS queries. The client then connects to the destination returned by DNS.
2. DNS Fundamentals
What is DNS?

DNS (Domain Name System) is a distributed, hierarchical naming system that maps domain names to information such as IP addresses and mail servers.

text
app.example.com
       |
       v
  DNS resolution
       |
       v
 203.0.113.10
       |
       v
Application server

DNS Hierarchy
text
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
   www             api

Important DNS Terminology
Term	Meaning
Domain name	A human-readable name, such as example.com
DNS resolver	Finds DNS answers on behalf of clients
Authoritative DNS server	Provides authoritative records for a zone
DNS query	A request for DNS information
DNS response	The answer to a DNS query
TTL	How long an answer may be cached
FQDN	Fully Qualified Domain Name
DNS delegation	Assigning authority for a domain to nameservers
NS record	Identifies authoritative nameservers
Recursive lookup	Resolver obtains an answer through additional queries
Authoritative lookup	Query directed to the DNS authority for the zone
Common DNS Ports
Protocol	Port	Purpose
UDP	53	Most ordinary DNS queries
TCP	53	Large responses and other DNS operations
HTTPS	443	DNS-over-HTTPS, when supported
TLS	853	DNS-over-TLS, when supported
⚠️ Important: Route 53's public authoritative DNS service is not a general-purpose DNS-over-HTTPS recursive resolver.
3. How Route 53 Works
DNS Resolution Flow
text
User's browser
      |
      v
Operating system DNS cache
      |
      v
Recursive DNS resolver
      |
      v
Root DNS server
      |
      v
.com TLD DNS server
      |
      v
Route 53 authoritative nameserver
      |
      v
A / AAAA / Alias record
      |
      v
IP address or service destination
      |
      v
Browser connects to destination

Step-by-Step Explanation
The browser needs the IP address for the requested hostname.
The operating system or configured resolver checks its cache.
If no usable cached answer exists, a recursive resolver performs the necessary lookups.
DNS delegation directs the resolver to the authoritative nameservers.
Route 53 returns the applicable DNS record.
The client connects to the returned destination.
The application processes the request.
💡 Remember: Cached answers can eliminate some or all intermediate DNS lookups. Route 53 does not normally proxy application traffic after DNS resolution.
Example Architecture
text
Internet user
      |
      v
www.example.com
      |
      v
Amazon Route 53
      |
      v
CloudFront CDN
      |
      v
Application origin
      |
      v
Database

4. Hosted Zones

A hosted zone is a container for DNS records associated with a domain or subdomain.

text
Hosted zone: example.com

Records:
├── example.com
├── www.example.com
├── api.example.com
└── mail.example.com

4.1 Public Hosted Zone

A public hosted zone contains records used to resolve domain names on the public internet.

Typical use cases:

Public websites.
Public APIs.
Public load balancers.
CloudFront distributions.
Public email routing.
Important: A public hosted zone does not make a private application publicly reachable by itself.
4.2 Private Hosted Zone

A private hosted zone contains DNS records resolvable from associated VPCs, subject to the relevant DNS configuration and connectivity.

text
VPC
 |
 +-- app.internal.example.com
 |
 +-- db.internal.example.com
 |
 +-- cache.internal.example.com


Typical use cases:

Internal microservices.
Private application endpoints.
Database hostname resolution.
Multi-VPC internal naming.
Hybrid environments connected through VPN or Direct Connect with suitable DNS forwarding.
Public vs. Private Hosted Zones
Feature	Public	Private
Resolution scope	Public DNS	Associated VPCs and configured connected networks
Primary purpose	Internet-facing services	Internal services
VPC association required	No	Yes
Public delegation	Relevant	Not used for public delegation
Example	www.example.com	db.internal.example.com
Important Hosted-Zone Records
SOA (Start of Authority): Contains zone metadata and authoritative DNS information.
NS (Name Server): Identifies the authoritative nameservers for a zone.
Hosted-zone ID: Identifies the zone in AWS CLI and API operations.

Route 53 creates NS and SOA records when a hosted zone is created. Preserve these records unless you have a specific, correctly configured DNS requirement.

5. DNS Record Types
5.1 A Record — IPv4

Maps a hostname to an IPv4 address.

text
example.com  A  203.0.113.10

5.2 AAAA Record — IPv6

Maps a hostname to an IPv6 address.

text
example.com  AAAA  2001:db8::10

5.3 CNAME Record — Hostname Alias

Maps one hostname to another hostname.

text
www.example.com  CNAME  example.com


Important restrictions:

A CNAME cannot coexist with other record types at the same owner name.
An ordinary CNAME cannot be created at the zone apex, such as example.com.
CNAME targets are hostnames, not IP addresses.
5.4 Alias Record

A Route 53 Alias record can point to supported AWS resources and certain supported DNS record targets.

Common targets include:

CloudFront distributions.
Elastic Load Balancing load balancers.
Amazon S3 website endpoints.
Other supported AWS resource targets.
Certain records within Route 53.
text
example.com  A (Alias)  CloudFront distribution

Alias vs. CNAME
Feature	Alias	CNAME
Route 53-specific	Yes	No
Zone apex support	Yes, for eligible targets and configurations	No
Target	Supported AWS resources or supported records	Another hostname
Coexists with other record types	Depends on record type and configuration	No
Arbitrary external hostname	Not as a general-purpose replacement	Yes
💡 Best practice: Consider an Alias record for supported AWS resources, especially when mapping a zone apex to an AWS endpoint. Verify the supported target and applicable pricing rules.
5.5 MX Record — Mail Servers
text
example.com  MX  10 mail.example.com


Lower MX priority values indicate a higher preference.

5.6 TXT Record — Text Data

Common uses include:

Domain ownership verification.
SPF email sender policy.
DKIM public-key information.
DMARC policy.
Service verification.
text
example.com TXT "v=spf1 include:example.net -all"

5.7 NS Record — Nameservers

Identifies authoritative nameservers and supports domain or subdomain delegation.

5.8 PTR Record — Reverse DNS

Maps an IP address to a hostname through reverse DNS.

5.9 SRV Record — Service Discovery

Specifies service priority, weight, port, and target.

5.10 CAA Record — Certificate Authority Restrictions

Specifies which certificate authorities are authorized to issue TLS certificates for a domain.

text
example.com CAA 0 issue "letsencrypt.org"

⚠️ Note: CAA records do not replace certificate validation or renewal management.
6. Routing Policies

Route 53 routing policies determine which DNS answer is returned for a query.

Policy	Primary purpose
Simple	Straightforward DNS answer
Weighted	Distribution according to configured weights
Latency-based	Lower-latency AWS Region
Failover	Primary or secondary answer based on health
Geolocation	Route by geographic location
Geoproximity	Route by location and configured bias
Multivalue answer	Return multiple eligible record values
IP-based	Route using source IP ranges and CIDR mappings
6.1 Simple Routing

Used for a straightforward DNS response.

text
www.example.com
       |
       v
203.0.113.10


Use cases: Simple websites, single endpoints, and basic hostname-to-address mappings.

6.2 Weighted Routing

Distributes DNS responses according to configured relative weights.

text
                 Route 53
                    |
           -------------------
           |                 |
        Weight 80         Weight 20
           |                 |
       Version A         Version B


Use cases:

Blue-green deployments.
Canary releases.
Gradual migrations.
Testing new application versions.
⚠️ Important: DNS weights do not guarantee exact percentages of application requests. Caching and client behavior influence actual traffic.
6.3 Latency-Based Routing

Returns an answer associated with an AWS Region expected to provide lower network latency.

text
               Route 53
              /        \
             v          v
       us-east-1     ap-south-1


Use cases: Global applications, multi-Region architectures, and latency-sensitive services.

6.4 Failover Routing

Directs DNS responses toward a primary endpoint and a secondary endpoint.

text
              Route 53
                  |
           Primary healthy?
             /        \
           Yes         No
            |           |
            v           v
        Primary      Secondary


Use cases: Disaster recovery, multi-Region failover, and standby application environments.

6.5 Geolocation Routing

Routes queries based on the geographic origin of the DNS query as determined by Route 53.

text
India           → India-specific endpoint
Europe          → Europe-specific endpoint
North America   → North America-specific endpoint


Configure a default record when appropriate for queries that do not match a more specific geographic rule.

6.6 Geoproximity Routing

Routes based on the geographic location of users and resources, with optional bias to expand or shrink the effective routing area.

6.7 Multivalue Answer Routing

Returns multiple eligible values in a DNS response. Health-check evaluation can be used for supported configurations.

6.8 IP-Based Routing

Routes queries according to source IP ranges and configured CIDR collection mappings.

💡 Exam tip: Understand the difference between weighted, latency-based, failover, geolocation, and geoproximity routing. Their purposes are distinct.
7. Health Checks and Failover
What Are Route 53 Health Checks?

Health checks evaluate whether configured endpoints or resources are healthy according to supported checks and settings.

Health-Check Types
Endpoint health checks: Monitor an endpoint using supported protocols such as HTTP, HTTPS, or TCP.
Calculated health checks: Combine the status of other health checks.
CloudWatch alarm health checks: Use a supported CloudWatch alarm's state.
Failover Example
text
                 Route 53
                     |
             Primary health check
                /          \
             Healthy       Unhealthy
                |              |
                v              v
           Primary site    Secondary site

Important Considerations
Endpoint health checks do not automatically verify every dependency.
Private endpoints may require a different health-monitoring design.
Health-check success does not guarantee that the entire application works.
DNS failover is affected by TTLs and resolver caching.
Key takeaway: DNS failover changes the answer available to resolvers. It does not replicate databases, create standby infrastructure, or terminate existing connections.
8. Route 53 with AWS Services
8.1 Route 53 + Application Load Balancer
text
Users
  |
  v
Route 53
  |
  v
Application Load Balancer
  |
  v
Target group
  |
  v
EC2 / containers


Recommended configuration:

Create an Alias record for the supported load balancer target.
Configure listeners and target groups.
Use an appropriate TLS certificate.
Configure security groups and application health checks.
8.2 Route 53 + CloudFront
text
User
  |
  v
Route 53
  |
  v
CloudFront
  |
  v
Origin: S3 / ALB / application


Benefits: Global content delivery, edge caching, HTTPS delivery, and application acceleration.

For a custom domain, configure the required CloudFront alternate domain name and TLS certificate in addition to the Route 53 record.

8.3 Route 53 + S3 Static Website
text
Browser
   |
   v
Route 53
   |
   v
S3 website endpoint

⚠️ Important: An S3 website endpoint does not itself provide HTTPS. CloudFront in front of S3 is a common approach for production HTTPS websites.

S3 website endpoints and S3 REST endpoints are different endpoint types.

8.4 Route 53 + EC2

A DNS record can point to a suitable EC2 public IP address or another supported destination.

Best practice: Prefer a stable endpoint, such as a load balancer, when direct instance IPs could change during replacement.

8.5 Route 53 + VPC

Important VPC DNS settings include:

text
enableDnsSupport
enableDnsHostnames


The required configuration depends on the VPC architecture and services in use.

8.6 Route 53 Resolver

Route 53 Resolver supports DNS resolution in VPC and hybrid environments.

Inbound endpoint: Allows connected networks to send DNS queries into a VPC.
Outbound endpoint: Forwards DNS queries from a VPC to configured external DNS servers through resolver rules.
text
On-premises DNS
       |
       v
Resolver inbound endpoint
       |
       v
AWS VPC DNS resolution

text
AWS VPC
   |
   v
Resolver outbound endpoint
   |
   v
On-premises DNS server

9. Private Hosted Zones

Private hosted zones allow applications to use meaningful internal DNS names without publishing those records through public authoritative DNS.

text
internal.example.com

api.internal.example.com
db.internal.example.com
cache.internal.example.com

Multi-VPC Design
text
             Private hosted zone
             internal.example.com
                      |
             -------------------
             |                 |
            VPC A             VPC B
             |                 |
        Application          Services

Split-Horizon DNS

The same hostname can return different answers depending on the resolver context.

text
example.com

Public DNS:
  app.example.com → public endpoint

Private DNS:
  app.example.com → private endpoint

Common Mistakes
Forgetting to associate the hosted zone with the required VPC.
Overlapping private hosted zones and resolver rules.
Misconfiguring VPC DNS settings.
Forgetting to configure hybrid DNS forwarding.
Assuming DNS resolution automatically provides network connectivity.
10. Domain Registration and Delegation
Domain vs. Hosted Zone
Concept	Responsibility
Domain registration	Registers and manages a domain name
Hosted zone	Stores DNS records
Registrar configuration	Identifies the authoritative nameservers
DNS record	Defines the answer for a hostname or service
Nameserver Delegation
text
Domain registrar
       |
       v
Nameserver delegation
       |
       v
Route 53 public hosted zone
       |
       v
DNS records


If the registrar points to another DNS provider, the Route 53 hosted zone may exist without being authoritative for public queries.

Subdomain Delegation
text
example.com
     |
     v
dev.example.com


Create the child hosted zone for dev.example.com, then configure the corresponding NS records in the parent zone.

11. TTL, Caching, and DNS Propagation
What Is TTL?

TTL (Time to Live) specifies how long a DNS response may be cached.

text
api.example.com
A
203.0.113.10
TTL = 300 seconds


A TTL of 300 seconds equals five minutes.

How TTL Affects DNS Changes
text
Before:
api.example.com → 203.0.113.10

After:
api.example.com → 203.0.113.20


Resolvers that cached the old answer may continue using it until the cached entry expires.

Important: Lowering a TTL does not retroactively shorten an already cached answer.
Recommended TTL Strategy
Scenario	Approach
Stable records	Use a reasonable TTL
Planned migration	Lower TTL sufficiently in advance
Frequently changing destinations	Consider lower TTLs
High-volume DNS workloads	Balance freshness with caching efficiency
DNS Propagation vs. Caching

Potential sources of delay include:

Authoritative record updates.
Registrar delegation changes.
Recursive resolver caches.
Operating-system and browser caches.
Application-level connection reuse.
12. Security and Best Practices
1. Apply Least Privilege with IAM

Limit who can create hosted zones, change DNS records, modify health checks, and configure resolver rules.

2. Protect Domain Registration
Enable multifactor authentication.
Use domain transfer locks where supported.
Keep registration information current.
Monitor expiration and renewal failures.
3. Use DNSSEC Appropriately

DNSSEC adds cryptographic validation to DNS data and helps protect against certain forms of DNS response tampering.

Note: DNSSEC does not encrypt DNS queries or replace HTTPS.
4. Use Private Hosted Zones for Internal Names

Private DNS helps control resolution scope, but it is not a substitute for network access controls.

5. Monitor DNS Changes

Use appropriate AWS monitoring, CloudTrail logging, alerting, and operational reviews.

6. Configure Meaningful Health Checks

Test the application's ability to serve requests rather than merely checking whether a process is running.

7. Plan DNS Migrations

Before changing production records:

Verify the target hosted zone.
Verify NS delegation.
Validate record values.
Review TTLs.
Confirm certificates and endpoint configuration.
Test the destination independently.
13. AWS CLI Commands
Prerequisites: Install and configure AWS CLI, select the appropriate AWS account, and ensure the required IAM permissions are available.
13.1 List Hosted Zones
bash
aws route53 list-hosted-zones

13.2 Get Hosted-Zone Details
bash
aws route53 get-hosted-zone \
  --id Z123456789EXAMPLE

13.3 List DNS Records
bash
aws route53 list-resource-record-sets \
  --hosted-zone-id Z123456789EXAMPLE

13.4 Create a Hosted Zone
bash
aws route53 create-hosted-zone \
  --name example.com \
  --caller-reference "example-com-20261026"


Review the returned nameservers and configure delegation when necessary.

13.5 Create or Update an A Record

Create change-record.json:

json
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


Apply the change:

bash
aws route53 change-resource-record-sets \
  --hosted-zone-id Z123456789EXAMPLE \
  --change-batch file://change-record.json


UPSERT creates a record if it does not exist or updates it if it does.

13.6 Check Change Status
bash
aws route53 get-change \
  --id CHANGE_ID


A status of INSYNC indicates that the change has propagated across Route 53's authoritative infrastructure. It does not mean every recursive resolver has discarded its cache.

13.7 Delete a Record

The submitted record set must match the existing record set.

json
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

⚠️ Warning: Validate the record before deleting production DNS entries.
13.8 List Health Checks
bash
aws route53 list-health-checks

13.9 List Traffic Policies
bash
aws route53 list-traffic-policies

13.10 Validate DNS
bash
dig example.com
dig example.com A
dig example.com AAAA
dig example.com NS
dig example.com MX
dig +trace example.com


Query a specific resolver:

bash
dig @1.1.1.1 example.com

14. Troubleshooting Guide
Problem 1: Domain Is Not Resolving

Possible causes:

Incorrect nameserver delegation.
Missing DNS record.
Incorrect hosted-zone configuration.
Cached negative DNS response.
Domain registration or delegation problems.

Commands:

bash
dig example.com
dig example.com NS
dig +trace example.com

Problem 2: DNS Returns the Old IP Address

Possible causes: Cached response, unexpired TTL, operating-system cache, wrong hosted zone, or an existing connection.

bash
dig app.example.com
dig @1.1.1.1 app.example.com
dig @8.8.8.8 app.example.com

Problem 3: Alias Record Does Not Work

Check target support, record configuration, resource configuration, resource reachability, and domain delegation.

Problem 4: Private Hosted Zone Does Not Resolve

Check VPC association, VPC DNS support, query origin, resolver rules, and hybrid forwarding.

Problem 5: Failover Is Not Occurring

Check health-check status, endpoint accessibility, protocol and path, failure thresholds, failover configuration, and cached DNS answers.

Problem 6: Website Is Reachable by IP but Not by Domain

Check DNS values, registrar nameservers, listener configuration, TLS certificate hostname coverage, CloudFront configuration, and caching.

15. Real-World Architecture Examples
Architecture 1: Highly Available Web Application
text
                    Users
                      |
                      v
                  Route 53
                      |
                      v
              Application Load Balancer
                      |
              --------------------
              |                  |
              v                  v
           EC2 / ECS          EC2 / ECS
              |                  |
              ---------+---------
                       |
                       v
                  Data layer


Key concepts:

Route 53 resolves the application hostname.
The load balancer distributes requests.
Application instances run in suitable subnets.
Security groups restrict traffic.
Data services meet availability and recovery requirements.
Architecture 2: Multi-Region Disaster Recovery
text
                     Users
                       |
                       v
                   Route 53
                       |
             ---------------------
             |                   |
             v                   v
         Region A             Region B
          Primary             Standby
             |                   |
             v                   v
        Application         Application
             |                   |
             v                   v
           Data  <--- replication ---> Data


Key concepts:

Configure an appropriate failover policy.
Prepare the secondary environment.
Replicate or otherwise make data available in the recovery region.
Test failover and failback.
Define RTO and RPO.
Account for DNS caching and client retries.
Architecture 3: Global Application with Latency Routing
text
                    Global users
                         |
                         v
                     Route 53
                         |
               ---------------------
               |                   |
               v                   v
          US Region             India Region
               |                   |
               v                   v
          Application           Application

Architecture 4: Hybrid DNS
text
              On-premises network
                       |
                Corporate DNS
                       |
                       v
                 VPN / Direct
                  Connect
                       |
                       v
               Route 53 Resolver
                       |
                       v
                    AWS VPC
                       |
                       v
             Private hosted zones

16. Interview Questions and Answers
🟢 Beginner Level

1. What is Amazon Route 53?

Amazon Route 53 is an AWS DNS service used for domain registration, DNS record management, and DNS-based traffic routing.

2. Why is Route 53 called Route 53?

The number 53 refers to the standard DNS port.

3. What is a hosted zone?

A collection of DNS records associated with a domain or subdomain.

4. What is the difference between public and private hosted zones?

Public hosted zones serve public DNS resolution. Private hosted zones serve associated VPC environments and appropriately configured connected networks.

5. What is TTL?

The duration for which a DNS answer may be cached.

6. What is a CNAME record?

A record that maps one hostname to another hostname.

7. What is an Alias record?

A Route 53 feature that maps a DNS name to supported AWS resources or supported DNS record targets.

8. Can you create a CNAME at the root domain?

No. Use an Alias record for eligible Route 53 targets at the zone apex.

🟡 Intermediate Level

9. What is weighted routing?

Routing DNS responses according to configured relative weights.

10. What is latency-based routing?

Routing DNS queries toward an eligible regional resource based on AWS latency-routing measurements.

11. How does failover routing work?

It returns primary or secondary DNS answers based on configured routing and health evaluation.

12. Geolocation vs. geoproximity routing?

Geolocation: Routes according to the geographic origin associated with a DNS query.
Geoproximity: Routes according to user and resource locations, with optional geographic bias.

13. What is the difference between Route 53 and a load balancer?

Route 53 resolves DNS names and steers DNS answers. A load balancer distributes application traffic among targets.

14. Does Route 53 immediately clear cached records?

No. Cached answers may remain until their TTL expires or the relevant cache refreshes.

15. What is a private hosted zone?

A DNS zone that provides records for associated VPC environments without publishing those records as public authoritative DNS records.

🔴 Advanced Level

16. How would you design disaster recovery using Route 53?

Configure DNS failover and meaningful health evaluation, prepare a secondary environment, replicate data, test recovery procedures, and account for RTO, RPO, TTLs, and retries.

17. How would you gradually migrate traffic to a new version?

Use weighted routing or another suitable traffic-management approach, monitor the application, and increase the new version's weight as confidence grows.

18. Why might failover not happen immediately?

Health evaluation takes time, DNS answers may be cached, and existing connections can continue using the old destination.

19. How would you troubleshoot nameserver delegation?

Inspect parent-zone delegation, query NS records, use dig +trace, and verify the registrar's nameserver configuration.

20. How do you resolve private DNS names from on-premises?

Configure connectivity, Route 53 Resolver inbound endpoints or another suitable hybrid design, and appropriate DNS forwarding.

21. Does a successful health check guarantee application availability?

No. It evaluates configured signals and may not cover every dependency or client path.

22. Can Route 53 replace CloudFront or an ALB?

No. Route 53 provides DNS, CloudFront delivers content, and an ALB distributes application-layer traffic.

23. How can you route traffic by client IP range?

Use IP-based routing with suitable CIDR collections and mappings.

24. What is split-horizon DNS?

The same hostname returns different DNS answers depending on the resolver context.

25. What is DNSSEC?

DNSSEC uses cryptographic signatures to support validation of DNS data authenticity and integrity. It does not encrypt DNS queries.

17. Quick Revision Cheat Sheet
Record Types
Record	Purpose
A	IPv4 address
AAAA	IPv6 address
CNAME	Alias from one hostname to another
Alias	Route 53-specific mapping to supported resources or records
MX	Mail servers
TXT	Verification and policy data
NS	Authoritative nameservers
SOA	Zone authority metadata
PTR	Reverse DNS
SRV	Service location and port
CAA	Certificate authority restrictions
Routing Policies
Requirement	Policy
Simple DNS response	Simple
Gradual traffic shift	Weighted
Lower-latency regional destination	Latency-based
Primary and secondary endpoints	Failover
Geographic routing	Geolocation
Geographic routing with bias	Geoproximity
Multiple eligible answers	Multivalue answer
Route by client IP ranges	IP-based
Essential Commands
bash
# List hosted zones
aws route53 list-hosted-zones

# Get hosted-zone information
aws route53 get-hosted-zone --id ZONE_ID

# List DNS records
aws route53 list-resource-record-sets \
  --hosted-zone-id ZONE_ID

# Submit a DNS record change
aws route53 change-resource-record-sets \
  --hosted-zone-id ZONE_ID \
  --change-batch file://change-record.json

# Inspect a DNS record
dig example.com

# Inspect nameserver delegation
dig example.com NS

# Trace DNS resolution
dig +trace example.com

# List health checks
aws route53 list-health-checks

⭐ Five Concepts to Remember
Route 53 is DNS, not a proxy load balancer.
Public and private hosted zones have different resolution scopes.
Alias records can support zone-apex routing for eligible targets; ordinary CNAME records cannot.
Routing policies influence DNS answers, not guaranteed application-request percentages.
DNS failover, TTLs, application health, and data replication are separate concerns.
🎯 Final Summary

Amazon Route 53 is a foundational AWS networking service for translating domain names into destinations and controlling DNS-based traffic steering.

To use it effectively, understand DNS resolution and delegation, record types, routing policies, private DNS, health checks, and TTLs.

For production systems, combine Route 53 with appropriate application load balancing, monitoring, IAM security, data replication, and tested recovery procedures.

Recommended learning order:

DNS fundamentals → Hosted zones → Records and Alias → Routing policies → Health checks → Resolver and private DNS → Disaster recovery → CLI and troubleshooting.

End of AWS Route 53 Complete Notes.
