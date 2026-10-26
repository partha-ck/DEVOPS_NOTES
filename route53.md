🌐 AWS Route 53 — Complete Notes
A comprehensive guide to Amazon Route 53, covering DNS fundamentals, hosted zones, records, routing policies, health checks, integrations, security, troubleshooting, and interview preparation.
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

For example:

text
www.example.com  →  203.0.113.10


Instead of remembering an IP address, users can access an application through a domain name.

Why is it called Route 53?
Route refers to directing requests to appropriate destinations.
53 refers to port 53, the standard port used by DNS.

DNS commonly uses UDP port 53 and TCP port 53.

Key features
Domain registration and DNS management.
Public and private hosted zones.
Multiple DNS routing policies.
DNS-based health checks and failover.
Geographic and latency-aware routing.
Integration with CloudFront, Elastic Load Balancing, S3, and other AWS services.
DNSSEC support for supported DNS configurations.
Private DNS resolution within Amazon VPC environments.
Route 53 is not
A web server.
A load balancer that directly proxies application traffic.
A replacement for an application firewall.
A service that automatically transfers application data.

Route 53 primarily answers DNS queries. The client then connects to the destination returned by DNS.

2. DNS Fundamentals
What is DNS?

DNS is a distributed, hierarchical naming system that maps domain names to information such as IP addresses and mail servers.

Example:

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

DNS hierarchy
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


The hierarchy consists of:

Root DNS servers — direct resolvers toward the appropriate top-level domain.
TLD DNS servers — manage delegation for domains such as .com, .org, and .in.
Authoritative DNS servers — provide DNS records for a domain.
Recursive resolvers — obtain answers on behalf of clients and cache them.
Important DNS terminology
Term	Meaning
Domain name	A human-readable name such as example.com
DNS resolver	A service that finds DNS answers for clients
Authoritative DNS server	A server that provides authoritative records
DNS query	A request for DNS information
DNS response	The answer to a DNS query
TTL	Time an answer may be cached
FQDN	Fully Qualified Domain Name
DNS delegation	Assigning authority for a domain to name servers
Nameserver (NS)	Identifies DNS servers responsible for a zone
Recursive lookup	Resolver obtains the answer through additional queries
Authoritative lookup	Query directed to the DNS authority for the zone
Common DNS ports
Protocol	Port	Purpose
UDP	53	Most ordinary DNS queries
TCP	53	Large responses, zone transfers, and other DNS operations
HTTPS	443	DNS-over-HTTPS, when supported by a resolver
TLS	853	DNS-over-TLS, when supported by a resolver

Route 53's public authoritative DNS service uses the standard DNS protocol; it is not itself a general-purpose DNS-over-HTTPS recursive resolver.

3. How Route 53 Works
DNS resolution flow

Suppose a user visits www.example.com.

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
Authoritative Route 53 nameserver
      |
      v
A / AAAA / Alias record
      |
      v
IP address or service destination
      |
      v
Browser connects to the destination


In practice, cached answers may eliminate some or all of the intermediate DNS lookups.

Step-by-step explanation
The browser needs the IP address for the requested hostname.
The operating system or configured resolver checks its cache.
If no usable cached answer exists, a recursive resolver performs the necessary lookups.
DNS delegation directs the resolver to the authoritative nameservers.
Route 53 returns the applicable DNS record.
The client connects to the returned destination.
The application processes the request.

Important: Route 53 does not normally sit between the browser and application after DNS resolution. The browser connects directly to the selected endpoint, unless another service such as CloudFront or a load balancer is the destination.

Example architecture
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


Here, Route 53 resolves the hostname to the appropriate CloudFront destination. CloudFront handles content delivery and forwards requests to its configured origin.

4. Hosted Zones
What is a hosted zone?

A hosted zone is a container for DNS records associated with a domain or subdomain.

Example:

text
Hosted zone: example.com

Records:
  example.com
  www.example.com
  api.example.com
  mail.example.com


Route 53 provides two primary hosted-zone types.

4.1 Public hosted zone

A public hosted zone contains records used to resolve domain names on the public internet.

Example:

text
www.example.com
        |
        v
Public Route 53 hosted zone
        |
        v
Publicly resolvable destination


Typical uses:

Public websites.
Public APIs.
Public load balancers.
CloudFront distributions.
Public email routing.

A public hosted zone does not make a private application publicly reachable by itself. Network configuration, endpoint exposure, and security controls still determine reachability.

4.2 Private hosted zone

A private hosted zone contains DNS records that are resolvable from associated VPCs, subject to the relevant DNS configuration and connectivity.

Example:

text
VPC
 |
 +-- app.internal.example.com
 |
 +-- db.internal.example.com
 |
 +-- cache.internal.example.com


Typical uses:

Internal microservices.
Private application endpoints.
Database hostname resolution.
Multi-VPC internal naming.
Hybrid environments connected through VPN or Direct Connect with suitable DNS forwarding.
Public vs. private hosted zones
Feature	Public hosted zone	Private hosted zone
Resolution scope	Public DNS	Associated VPCs and configured connected networks
Typical purpose	Internet-facing services	Internal services
VPC association required	No	Yes
Public DNS delegation	Relevant for domain authority	Not used for public delegation
Common use case	www.example.com	db.internal.example.com
Important hosted-zone concepts
SOA record

The Start of Authority record contains zone metadata and authoritative DNS information.

NS record

The Name Server record identifies the authoritative nameservers for a zone.

Route 53 creates NS and SOA records when a hosted zone is created. The automatically created records should generally be preserved unless you have a specific, correctly configured DNS requirement.

Hosted-zone ID

Every hosted zone has an identifier. AWS CLI operations often use this ID to select the zone.

5. DNS Record Types

DNS records determine what information Route 53 returns for a domain.

5.1 A record

Maps a hostname to an IPv4 address.

text
example.com  A  203.0.113.10


Use it when you need to return an IPv4 address.

5.2 AAAA record

Maps a hostname to an IPv6 address.

text
example.com  AAAA  2001:db8::10


Use it for IPv6 connectivity.

5.3 CNAME record

Maps one hostname to another hostname.

text
www.example.com  CNAME  example.com


The resolver follows the target hostname to obtain its address.

Important restrictions:

A CNAME cannot coexist with other record types at the same owner name.
A CNAME generally cannot be created at the zone apex, such as example.com, because the apex must also contain records such as SOA and NS.
CNAME targets are hostnames, not IP addresses.
5.4 Alias record

An Alias record is a Route 53-specific capability that allows supported DNS names to point to selected AWS resources and certain records.

Common targets include:

CloudFront distributions.
Elastic Load Balancing load balancers.
Amazon S3 website endpoints.
Other supported AWS resource targets.
Certain records within Route 53.

Example:

text
example.com  A (Alias)  CloudFront distribution


Alias records can be used at the zone apex, unlike ordinary CNAME records.

Alias vs. CNAME
Feature	Alias	CNAME
Route 53-specific feature	Yes	No
Zone apex supported	Yes, for supported targets and record configurations	No
Target	Supported AWS resources or supported DNS record targets	Another hostname
Can coexist with other record types at the same name	Depends on record type and configuration	No
Works with arbitrary external hostname	Not as a general arbitrary-hostname replacement	Yes
DNS query charge	Queries to Alias records targeting qualifying AWS resources can be free under Route 53 pricing rules	Standard DNS query pricing applies

An Alias record is not a generic CNAME for every internet hostname. Verify that the intended target is supported.

5.5 MX record

Specifies mail servers for a domain.

text
example.com  MX  10 mail.example.com


The priority value determines preference; lower values indicate higher preference.

5.6 TXT record

Stores text data associated with a hostname.

Common uses:

Domain ownership verification.
SPF email sender policy.
DKIM public-key information.
DMARC policy.
Service verification.

Example:

text
example.com TXT "v=spf1 include:example.net -all"


SPF, DKIM, and DMARC serve different purposes and should be configured according to the email provider's requirements.

5.7 NS record

Identifies authoritative nameservers.

Common uses:

Delegating a domain.
Delegating a subdomain to another hosted zone.
Identifying a zone's authoritative servers.
5.8 PTR record

Used for reverse DNS lookups, mapping an IP address to a hostname.

Reverse DNS zones are commonly managed through the authority responsible for the relevant IP address range.

5.9 SRV record

Specifies the location of a service, including its priority, weight, port, and target.

5.10 CAA record

Specifies which certificate authorities are authorized to issue TLS certificates for a domain.

Example:

text
example.com CAA 0 issue "letsencrypt.org"


CAA records are a restriction on certificate issuance, not a substitute for certificate validation or renewal management.

6. Routing Policies

Route 53 routing policies determine which DNS answer is returned when a query is made.

They support traffic distribution, geographic routing, latency optimization, failover, and other application-routing designs.

Routing-policy overview
Policy	Primary purpose
Simple	Return a straightforward DNS answer
Weighted	Distribute traffic according to configured weights
Latency-based	Direct users toward a lower-latency AWS Region
Failover	Return primary or secondary answers based on health
Geolocation	Route based on the user's geographic location
Geoproximity	Route based on location and configured bias
Multivalue answer	Return multiple healthy record values
IP-based	Route according to the source IP address and configured CIDR mapping
6.1 Simple routing

Used for a straightforward DNS response.

Example:

text
www.example.com
        |
        v
203.0.113.10


Use cases:

A simple website.
A single endpoint.
A basic hostname-to-address mapping.

Simple routing does not provide sophisticated health-based traffic steering.

6.2 Weighted routing

Distributes DNS responses according to configured record weights.

Example:

text
                 Route 53
                    |
           -------------------
           |                 |
        Weight 80         Weight 20
           |                 |
       Version A         Version B


If two eligible records have weights of 80 and 20, their relative selection proportions are approximately 80% and 20%, subject to DNS caching and client behavior.

Use cases
Blue-green deployments.
Canary releases.
Gradual migrations.
Testing a new application version.
Traffic distribution across endpoints.

Important: Weighted routing does not guarantee an exact percentage of application requests. DNS caching, resolver behavior, and client reuse of addresses influence actual traffic.

6.3 Latency-based routing

Returns an answer associated with an AWS Region expected to provide lower network latency to the requester.

Example:

text
               Route 53
              /        \
             v          v
       us-east-1     ap-south-1
             \          /
              Application


Use cases:

Global applications.
Multi-Region architectures.
Latency-sensitive services.

Latency-based routing uses AWS's latency measurements and routing configuration; it does not continuously measure every user's actual end-to-end application response time.

6.4 Failover routing

Used to direct DNS responses toward a primary endpoint and a secondary endpoint.

text
              Route 53
                  |
           Primary healthy?
             /        \
           Yes         No
            |           |
            v           v
        Primary      Secondary


Typical configuration:

Primary record.
Secondary record.
Appropriate health check or supported health-evaluation configuration.

Use cases:

Disaster recovery.
Multi-Region failover.
Standby application environments.

Failover does not instantly move every existing connection. DNS caches and application retry behavior affect recovery time.

6.5 Geolocation routing

Routes queries based on the geographic origin of the DNS query as determined by Route 53.

Example:

text
India       → India-specific endpoint
Europe      → Europe-specific endpoint
North America → North America-specific endpoint


Use cases:

Localized websites.
Region-specific content.
Compliance-related traffic steering.
Country-specific user experiences.

Configure a default record for queries that do not match a more specific geographic rule when appropriate.

Geolocation is not a precise user-location guarantee. VPNs, recursive resolvers, and network topology can affect the apparent location.

6.6 Geoproximity routing

Routes traffic based on the geographic location of users and resources, with optional bias to expand or shrink the effective routing area.

Use cases:

Adjusting regional traffic distribution.
Shifting traffic toward a selected region.
Managing traffic across distributed infrastructure.

Geoproximity routing is more flexible than fixed geographic matching, but it requires careful planning and supported routing configuration.

6.7 Multivalue answer routing

Allows Route 53 to return multiple eligible values in a DNS response, with health-check evaluation available for supported configurations.

Use cases:

Returning multiple server addresses.
Providing simple DNS-level distribution across endpoints.
Excluding unhealthy endpoints when health evaluation is configured.

Multivalue answer routing is not a replacement for a fully featured load balancer.

6.8 IP-based routing

Routes DNS queries according to the source IP address range and configured CIDR collection mappings.

Use cases:

Network-specific routing.
Routing based on known client network ranges.
Specialized enterprise traffic-steering designs.

It requires accurate CIDR definitions and careful handling of recursive resolver source addresses.

7. Health Checks and Failover
What are Route 53 health checks?

Health checks evaluate whether configured endpoints or resources are healthy according to supported checks and settings.

They can be used with DNS routing configurations to influence which records are returned.

Health-check types
1. Endpoint health checks

Monitor a specified endpoint using supported protocols and settings.

Examples:

HTTP.
HTTPS.
TCP.
2. Calculated health checks

Combine the health status of other health checks using configured conditions.

Useful for monitoring several components or defining aggregate health.

3. CloudWatch alarm health checks

Use the state of a supported CloudWatch alarm to determine health.

Useful when application health is represented by metrics or alarms rather than a directly reachable endpoint.

Failover example
text
                 Route 53
                     |
             Primary health check
                /          \
             Healthy       Unhealthy
                |              |
                v              v
           Primary site    Secondary site

Important health-check considerations
An endpoint health check does not automatically verify every dependency behind the endpoint.
Private endpoints may not be directly reachable by public Route 53 endpoint health checkers. Use an appropriate monitoring design, such as CloudWatch alarms, calculated health checks, or a publicly reachable health endpoint where suitable.
Health-check success is not proof that the entire application works correctly.
Configure health checks with an application-specific health endpoint and meaningful success criteria.
DNS failover is affected by TTLs and resolver caching.
Active-passive disaster recovery
text
                 Users
                   |
                Route 53
                   |
           -----------------
           |               |
        Primary         Secondary
        Region A        Region B
        Active          Standby


When the primary is considered unhealthy, Route 53 can return the secondary destination for new DNS resolutions.

The secondary region must already have the required infrastructure, data availability, and operational readiness.

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


Recommended approach:

Create an Alias record for the supported load balancer target.
Configure the load balancer listener and target groups.
Use the appropriate TLS certificate.
Configure security groups and application health checks.

An ALB is responsible for application-layer request distribution. Route 53 resolves the hostname to the load balancer destination.

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


Typical uses:

Global content delivery.
Website acceleration.
HTTPS delivery.
Edge caching.
Web application architectures.

For a custom domain, configure the required CloudFront alternate domain name and a suitable TLS certificate, in addition to the Route 53 record.

8.3 Route 53 + S3 static website
text
Browser
   |
   v
Route 53
   |
   v
S3 website endpoint


An Alias record can target a supported S3 website endpoint.

Important distinctions:

An S3 website endpoint does not itself provide HTTPS.
For a production HTTPS website, CloudFront in front of S3 is a common design.
S3 website hosting and S3 REST endpoints are different endpoint types.
8.4 Route 53 + EC2

A DNS record can point to a suitable EC2 public IP address or other supported destination.

However, directly mapping a production hostname to an ephemeral public IP can cause problems when an instance is replaced.

Prefer a stable endpoint, such as a load balancer, where appropriate.

8.5 Route 53 + VPC

Route 53 supports DNS designs for private AWS environments.

Important VPC DNS settings include:

enableDnsSupport
enableDnsHostnames

The exact setting requirements depend on the VPC design and the services being used.

8.6 Route 53 Resolver

Route 53 Resolver provides DNS resolution capabilities for VPCs and hybrid DNS architectures.

Resolver endpoints:

Inbound endpoint: allows connected networks to send DNS queries into a VPC for resolution.
Outbound endpoint: allows DNS queries from a VPC to be forwarded to specified external DNS servers through configured forwarding rules.

Example hybrid design:

text
On-premises DNS
       |
       v
Resolver inbound endpoint
       |
       v
AWS VPC DNS resolution


And the reverse direction:

text
AWS VPC
   |
   v
Resolver outbound endpoint
   |
   v
On-premises DNS server


Hybrid DNS requires appropriate network connectivity, security-group rules, routing, and resolver rules.

9. Private Hosted Zones
What are they used for?

Private hosted zones let applications use meaningful DNS names for internal resources without exposing those records through public DNS.

Example:

text
internal.example.com

api.internal.example.com
db.internal.example.com
cache.internal.example.com

Multi-VPC design
text
                 Private hosted zone
                 internal.example.com
                         |
              -----------------------
              |                     |
             VPC A                 VPC B
              |                     |
         Application             Services


The hosted zone can be associated with multiple VPCs, subject to AWS's supported association and authorization requirements.

Split-horizon DNS

Split-horizon DNS means the same domain name returns different DNS answers depending on the resolver context.

Example:

text
example.com

Public DNS:
  app.example.com → public endpoint

Private DNS:
  app.example.com → private endpoint


This can support internal access to services that are also publicly available.

Common mistakes
Forgetting to associate the private hosted zone with the required VPC.
Overlapping private hosted zones and resolver rules.
Misconfiguring VPC DNS settings.
Assuming private DNS records resolve automatically from on-premises networks without appropriate forwarding.
Assuming a private hosted zone makes a service reachable; DNS resolution and network connectivity are separate.
10. Domain Registration and Delegation
Registering a domain

A domain name must be registered through an appropriate registrar.

Route 53 Domains supports domain registration for supported top-level domains.

A domain can also be registered with another provider while its DNS is hosted in Route 53.

Domain vs. hosted zone
Concept	Responsibility
Domain registration	Registers the domain name and manages its registration
Hosted zone	Stores DNS records for a domain or subdomain
Registrar configuration	Identifies the authoritative nameservers for the domain
DNS record	Defines the answer for a hostname or service

Registering a domain does not automatically mean that its DNS is configured correctly.

Nameserver delegation

Suppose you create a public hosted zone for example.com.

Route 53 provides a set of authoritative nameservers.

The registrar's nameserver configuration must point to the correct nameservers for that hosted zone.

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


If the registrar points to another DNS provider, the Route 53 hosted zone may exist but will not be authoritative for public queries to the domain.

Subdomain delegation

You can delegate a subdomain to another hosted zone.

Example:

text
example.com
     |
     v
dev.example.com


Create the child hosted zone for dev.example.com, then create the appropriate NS records for dev.example.com in the parent zone.

This enables separate DNS administration for the subdomain.

11. TTL, Caching, and DNS Propagation
What is TTL?

TTL stands for Time to Live. It specifies how long a DNS response may be cached.

Example:

text
api.example.com
A
203.0.113.10
TTL = 300 seconds


The TTL is 300 seconds, or five minutes.

How TTL affects DNS changes

Suppose the destination changes:

text
Before:
api.example.com → 203.0.113.10

After:
api.example.com → 203.0.113.20


Resolvers that have cached the old answer may continue using it until their cached entry expires.

Reducing a TTL does not retroactively shorten a cached answer that was already obtained with a longer TTL.

Recommended TTL strategy
Scenario	General approach
Stable records	Use a reasonable TTL based on operational needs
Planned migration	Lower the TTL sufficiently in advance
Frequently changing destinations	Consider a lower TTL where appropriate
High-volume DNS workloads	Balance freshness against query volume and caching efficiency
DNS propagation vs. caching

DNS changes are often described as “propagation,” but several different processes may be involved:

Updating the authoritative record.
Updating domain delegation at the registrar.
Expiration of recursive resolver caches.
Operating-system and browser DNS caching.
Application-level connection reuse.

Changes to Route 53 record data generally become available through the authoritative service after the change is applied. Cached answers and delegation changes may take additional time to be reflected in clients.

TTL does not control everything

TTL does not necessarily govern:

Every application cache.
Existing TCP connections.
Every intermediate caching layer.
Client behavior after an IP address has been resolved.

For critical migrations, plan for clients to continue using old destinations temporarily.

12. Security and Best Practices
Security best practices
1. Apply least privilege with IAM

Limit who can:

Create or delete hosted zones.
Change DNS records.
Modify health checks.
Change domain registration settings.
Configure Route 53 Resolver rules and endpoints.

Use role-based access and scoped IAM policies where practical.

2. Protect domain registration
Enable registrar account security features.
Use multifactor authentication.
Enable domain transfer locks where supported.
Keep registration contact and renewal information current.
Monitor domain expiration and renewal failures.
3. Use DNSSEC appropriately

DNSSEC adds cryptographic validation to DNS data and helps protect against certain forms of DNS response tampering.

For Route 53 public DNS, supported DNSSEC signing configurations can be used with the appropriate parent-zone DS record and key-management setup.

DNSSEC does not encrypt DNS queries or replace HTTPS.

4. Use private hosted zones for internal names

Avoid publishing internal naming details unnecessarily.

Remember that private DNS is not a substitute for network-level access control.

5. Monitor important records and configuration changes

Use appropriate AWS monitoring, CloudTrail logging, alerting, and operational review processes.

6. Use health checks thoughtfully

A health endpoint should test the application's meaningful ability to serve requests rather than merely confirming that a process is running.

7. Plan DNS migrations

Before switching nameservers or production records:

Verify the target hosted zone.
Verify NS delegation.
Validate record values.
Review TTLs.
Confirm HTTPS certificates and endpoint configuration.
Test the destination independently.
Common mistakes to avoid
Creating a hosted zone but not updating the registrar's nameservers.
Using a CNAME at the zone apex.
Confusing Alias records with ordinary CNAME records.
Assuming DNS changes immediately clear all client caches.
Using Route 53 health checks as a substitute for application monitoring.
Assuming a private hosted zone provides network access.
Using public endpoint health checks for private-only endpoints without a suitable design.
Treating DNS routing as a replacement for application load balancing.
Deleting an existing hosted zone without checking dependencies.
Forgetting to configure a default record for geographic routing when needed.
13. AWS CLI Commands
Prerequisites: Install and configure the AWS CLI, select the appropriate AWS account and Region for the relevant API, and ensure your IAM permissions allow the requested operations.
13.1 List hosted zones
bash
aws route53 list-hosted-zones

13.2 Get details of a hosted zone
bash
aws route53 get-hosted-zone \
  --id Z123456789EXAMPLE


Replace the sample hosted-zone ID with your actual ID.

13.3 List DNS records
bash
aws route53 list-resource-record-sets \
  --hosted-zone-id Z123456789EXAMPLE

13.4 Create a hosted zone
bash
aws route53 create-hosted-zone \
  --name example.com \
  --caller-reference "example-com-20261026"


The caller reference should be unique for each intended creation request.

Review the returned nameservers and configure delegation when necessary.

13.5 Create or update an A record

Create a file named change-record.json:

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


UPSERT creates the record if it does not exist or updates it if it does.

Use a valid destination IP address for your environment. The documentation address above is an example, not a production endpoint.

13.6 Check change status
bash
aws route53 get-change \
  --id CHANGE_ID


Replace CHANGE_ID with the ID returned by the record-change operation.

A status of INSYNC indicates that the change has propagated across Route 53's authoritative infrastructure. It does not mean every recursive resolver has discarded its cache.

13.7 Delete a record

For deletion, the submitted record set must match the existing record set.

Example change batch:

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


Apply it using the same change-resource-record-sets command.

Be careful when deleting production records.

13.8 List health checks
bash
aws route53 list-health-checks

13.9 List reusable traffic policies
bash
aws route53 list-traffic-policies


Traffic policies are a separate Route 53 feature for defining and managing certain routing configurations.

13.10 Validate DNS using system tools
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


dig +trace can help inspect delegation and authoritative resolution. Its results depend on network access and the resolver environment.

14. Troubleshooting Guide
Problem 1: Domain is not resolving

Possible causes:

Incorrect nameserver delegation.
Missing DNS record.
Wrong hosted-zone ID or domain name.
Cached negative DNS response.
Domain registration or delegation problem.

Useful commands:

bash
dig example.com
dig example.com NS
dig +trace example.com


Check the authoritative nameservers and compare them with the intended public hosted zone.

Problem 2: DNS returns the old IP address

Possible causes:

Cached DNS response.
TTL has not expired.
Application or operating-system cache.
Record updated in the wrong hosted zone.
Existing connection continues using the old destination.

Useful commands:

bash
dig app.example.com
dig @1.1.1.1 app.example.com
dig @8.8.8.8 app.example.com


Compare answers from multiple resolvers and query the authoritative nameserver directly when necessary.

Problem 3: Alias record does not work

Check:

Whether the target resource is supported.
Whether the target resource is configured correctly.
Whether the record type is appropriate.
Whether the resource is reachable independently.
Whether the DNS name is delegated to the intended hosted zone.

An Alias record does not automatically configure a load balancer, CloudFront distribution, or TLS certificate.

Problem 4: Private hosted zone does not resolve

Check:

The hosted zone is associated with the correct VPC.
VPC DNS support is enabled.
The query is made from an appropriate network.
Resolver rules are not overriding the expected resolution.
Hybrid DNS forwarding is configured if queries originate on-premises.
Problem 5: Failover is not occurring

Check:

Health-check status.
Health-check endpoint accessibility.
Health-check protocol and path.
Failure thresholds and timing.
Failover record configuration.
Whether the routing policy evaluates health as intended.
DNS TTL and cached responses.

Failover changes the DNS answer available to resolvers; it does not terminate existing connections to the failed endpoint.

Problem 6: Website is reachable by IP but not by domain

Check:

DNS record values.
Registrar nameservers.
Virtual host or load balancer listener configuration.
TLS certificate hostname coverage.
CloudFront alternate domain configuration, if applicable.
DNS caching.

A successful IP connection does not prove that HTTPS, host-header routing, or DNS configuration is correct.

15. Real-World Architecture Examples
Architecture 1: Highly available web application
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
Application instances run in appropriate subnets.
Security groups restrict permitted traffic.
Data services are configured for the required availability and recovery objectives.
Architecture 2: Multi-Region disaster recovery
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

Use a suitable failover routing policy.
Configure meaningful health evaluation.
Replicate or otherwise make data available in the recovery region.
Test failover and failback procedures.
Define recovery time objective (RTO) and recovery point objective (RPO).
Account for DNS caching and application retry behavior.

Route 53 provides DNS steering; it does not replicate databases or automatically build the standby environment.

Architecture 3: Global application with latency routing
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


Key concepts:

Configure eligible regional records with latency routing.
Deploy the application in each intended region.
Ensure data consistency and replication meet application requirements.
Test application behavior from different locations.
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


Key concepts:

Inbound endpoints accept queries from connected networks.
Outbound endpoints forward queries to specified DNS servers.
Resolver rules determine which queries are forwarded.
Network routes and security groups must permit DNS traffic.
Private hosted-zone associations and forwarding rules must be designed together.
16. Interview Questions and Answers
Beginner level
1. What is Amazon Route 53?

Amazon Route 53 is an AWS DNS service used for domain registration, DNS record management, and DNS-based traffic routing.

2. Why is Route 53 called Route 53?

The number 53 refers to the standard DNS port.

3. What is a hosted zone?

A hosted zone is a collection of DNS records associated with a domain or subdomain.

4. What is the difference between a public and private hosted zone?

A public hosted zone provides DNS records for public DNS resolution. A private hosted zone provides DNS records for associated VPC environments and appropriately configured connected networks.

5. What is TTL?

TTL specifies how long a DNS answer may be cached.

6. What is a CNAME record?

A CNAME record maps one hostname to another hostname.

7. What is an Alias record?

An Alias record is a Route 53 feature that maps a DNS name to supported AWS resources or supported Route 53 record targets.

8. Can you create a CNAME at the root domain?

Not as an ordinary CNAME record. Route 53 Alias records can support the zone apex for eligible targets.

Intermediate level
9. What is weighted routing?

Weighted routing returns DNS answers according to configured relative weights, making it useful for canary releases and traffic migration.

10. What is latency-based routing?

It selects a DNS answer based on AWS's latency-routing measurements for the requester and the configured regional resources.

11. How does failover routing work?

It uses primary and secondary records with appropriate health evaluation to return a secondary destination when the primary is considered unhealthy.

12. What is the difference between geolocation and geoproximity routing?
Geolocation routes based on the geographic location associated with the DNS query.
Geoproximity routes based on the locations of users and resources, with optional geographic bias.
13. What is the difference between a load balancer and Route 53?

Route 53 provides DNS resolution and traffic steering. A load balancer distributes application traffic among configured targets.

14. Does Route 53 immediately remove cached DNS records after a change?

No. Authoritative changes can become available quickly, but cached answers may remain until their TTL expires or the relevant cache is otherwise refreshed.

15. What is a private hosted zone?

A private hosted zone provides DNS records for associated VPC environments without publishing those records as public authoritative DNS records.

Advanced level
16. How would you design Route 53 disaster recovery?

Use a suitable DNS failover configuration, health evaluation, a prepared secondary environment, appropriate data replication, and tested recovery procedures. Define RTO and RPO, and account for TTLs and client retries.

17. How would you gradually migrate traffic to a new application version?

Use weighted routing or another suitable traffic-management approach, monitor application behavior, and increase the new version's weight as confidence grows.

18. Why might failover not happen immediately?

Health evaluation takes time, DNS answers may be cached, and existing connections may continue using the old destination.

19. How would you troubleshoot incorrect nameserver delegation?

Inspect the parent-zone delegation, query NS records, use dig +trace, and verify that the registrar points to the intended hosted-zone nameservers.

20. How do you resolve private DNS names from an on-premises network?

Configure network connectivity and Route 53 Resolver inbound endpoints or another appropriate hybrid DNS design, then configure the on-premises DNS servers to forward the relevant queries.

21. Does Route 53 health checking guarantee application availability?

No. Health checks evaluate configured signals and endpoints. They may not cover every application dependency, client path, or failure mode.

22. Can Route 53 replace CloudFront or an ALB?

No. These services have different roles. Route 53 handles DNS, CloudFront provides content delivery, and an ALB distributes application-layer traffic.

23. How can you route traffic based on client IP ranges?

Use IP-based routing with appropriate CIDR collections and mappings.

24. What is split-horizon DNS?

It is a DNS configuration in which the same hostname can return different answers depending on the resolver context, such as public versus private DNS.

25. What is DNSSEC?

DNSSEC uses cryptographic signatures to support validation of DNS data authenticity and integrity. It does not encrypt DNS queries.

17. Quick Revision Cheat Sheet
Record types
Record	Purpose
A	IPv4 address
AAAA	IPv6 address
CNAME	Alias from one hostname to another
Alias	Route 53-specific mapping to supported AWS resources or records
MX	Mail servers
TXT	Text-based verification and policy data
NS	Authoritative nameservers
SOA	Zone authority metadata
PTR	Reverse DNS
SRV	Service location and port
CAA	Certificate authority restrictions
Routing policies
Requirement	Appropriate policy
Simple DNS response	Simple
Gradual traffic shift	Weighted
Lower-latency regional destination	Latency-based
Primary and secondary endpoints	Failover
Route by geographic location	Geolocation
Geographic routing with bias	Geoproximity
Multiple eligible answers	Multivalue answer
Route based on client IP ranges	IP-based
Essential commands
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

# Inspect authoritative delegation
dig example.com NS

# Trace DNS resolution
dig +trace example.com

# List health checks
aws route53 list-health-checks

Five concepts to remember
Route 53 is DNS, not a proxy load balancer.
Public and private hosted zones have different resolution scopes.
Alias records can support zone-apex routing for eligible targets; ordinary CNAME records cannot be used at the zone apex.
Routing policies influence DNS answers, not guaranteed application-request percentages.
DNS failover, TTLs, application health, and data replication are separate concerns that must work together.
Final Summary

Amazon Route 53 is a foundational AWS networking service for translating domain names into destinations and controlling DNS-based traffic steering.

To use it effectively, understand how DNS resolution and delegation work, when to choose each record type, how routing policies differ, how private DNS integrates with VPCs, and how health checks and TTLs influence failover.

For production systems, combine Route 53 with appropriate application load balancing, monitoring, IAM security, data replication, and tested recovery procedures.

Suggested learning order: DNS fundamentals → hosted zones → records and Alias → routing policies → health checks → Resolver and private DNS → disaster recovery → CLI and troubleshooting.

End of notes.