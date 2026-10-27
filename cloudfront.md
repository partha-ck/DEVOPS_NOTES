Amazon CloudFront — Senior DevOps Engineer Notes

Level: 5+ Years DevOps / SRE / Cloud Engineer
Topics: Architecture, caching, security, Terraform, troubleshooting, and interview questions

Table of Contents
What Is CloudFront?
Architecture and Request Flow
Cache Policies vs Origin Request Policies
TTL, Cache-Control, and Invalidation
Cache Behaviors and Origins
S3 Security: OAC vs OAI
DNS and TLS
CloudFront Functions vs Lambda@Edge
WAF and Security
API Caching and Data Leakage
Origin Failover and Error Handling
Monitoring and Troubleshooting
Production Architectures
Terraform Example
Advanced Caching Concepts
Easy Interview Questions
Medium Interview Questions
Hard Interview Questions
Expert Production Scenarios
Senior DevOps Checklist
1. What Is CloudFront?

Amazon CloudFront is AWS's Content Delivery Network (CDN). It delivers static and dynamic content through a globally distributed network of edge locations.

Main benefits
Lower latency for users.
Reduced origin load through caching.
Global content delivery.
Integration with AWS WAF and AWS Shield protections.
HTTPS and custom-domain support.
Private S3 origin access using Origin Access Control (OAC).
Support for static websites, APIs, downloads, and streaming content.
Request and response processing at the edge.
Common origins
Amazon S3
Application Load Balancer (ALB)
EC2
API Gateway
External HTTP/HTTPS servers

Senior-level insight: CloudFront is more than a cache. It is also a TLS endpoint, routing layer, security integration point, and request-processing platform.

2. Architecture and Request Flow
Main components
Component	Purpose
Distribution	Defines how CloudFront delivers content
Origin	Backend source of content
Cache behavior	Matches paths and configures caching, methods, and policies
Edge location	Serves viewer requests
Regional edge cache	Additional caching layer for eligible requests
Cache key	Determines which requests can share a cached response
Cache policy	Defines cache-key inputs and TTL settings
Origin request policy	Controls additional viewer request values forwarded to the origin
OAC	Secures CloudFront access to an S3 origin
CloudFront Functions	Lightweight edge request/response processing
Lambda@Edge	More complex edge processing
Request flow
A user requests https://www.example.com/assets/app.js.
DNS directs the hostname to CloudFront.
The browser establishes HTTPS with CloudFront.
Applicable viewer-side protections, including AWS WAF, are evaluated.
CloudFront selects the matching cache behavior.
CloudFront constructs the cache key.
If a usable cached response exists, CloudFront returns it.
Otherwise, CloudFront obtains or revalidates content through its caching hierarchy and origin.
CloudFront returns the response and caches it when permitted by the configuration and response rules.
Cache hit vs cache miss

Cache hit: A usable cached response is available.

Benefits:

Lower latency.
Less origin traffic.
Potentially lower origin infrastructure costs.

Cache miss: CloudFront needs to obtain or revalidate content.

Consequences:

Greater origin dependency.
Potentially higher latency.
More origin requests.

Useful response headers:

http
X-Cache: Hit from cloudfront
X-Cache: Miss from cloudfront
X-Cache: RefreshHit from cloudfront
Age: 120


A cache miss does not necessarily mean the origin itself was contacted; another cache layer or request path may satisfy the request.

3. Cache Policies vs Origin Request Policies

This is one of the most important CloudFront concepts.

Cache policy

Controls:

Cache-key query strings.
Cache-key headers.
Cache-key cookies.
Minimum TTL.
Default TTL.
Maximum TTL.
Origin request policy

Controls additional viewer request values forwarded to the origin without necessarily including those values in the cache key.

Values included in the cache key are also forwarded to the origin.

Example

Requests:

text
/products?id=101&campaign=google
/products?id=101&campaign=facebook


If the response depends only on id, a suitable design could be:

Include id in the cache key.
Forward campaign separately if the origin needs it for analytics.
Exclude campaign from the cache key.

Both requests may then share a cached product response.

Warning: This is safe only if campaign does not affect the response. If it changes the content, the cache key must account for it.

Interview question

Q: If a cookie is forwarded to the origin, is CloudFront caching separately for each cookie value?

Answer: No. Forwarding alone does not partition the cache. Include the relevant cookie in the cache key or disable shared caching if the response is user-specific.

4. TTL, Cache-Control, and Invalidation

TTL means time to live.

CloudFront cache policies define:

Minimum TTL
Default TTL
Maximum TTL

Example:

text
Minimum TTL = 0 seconds
Default TTL = 300 seconds
Maximum TTL = 3600 seconds


If the origin sends:

http
Cache-Control: public, max-age=120


the effective freshness lifetime can be 120 seconds, subject to the applicable policy and caching rules.

If the origin sends a freshness lifetime longer than the maximum TTL, the configured maximum limits it.

Important Cache-Control directives
Directive	Meaning
public	Response may be stored by shared caches
private	Intended for private caches, not shared-cache storage under normal HTTP semantics
no-cache	Storage may be allowed, but revalidation is required before reuse
no-store	Instructs caches not to store the response
max-age=300	Freshness lifetime of 300 seconds

Critical trap: A positive CloudFront minimum TTL can override the origin's intent not to cache. Use a minimum TTL of zero for sensitive content and when origin cache directives must be honored.

Invalidation

Invalidate cached objects when content must be refreshed before normal expiration.

bash
aws cloudfront create-invalidation \
  --distribution-id E1234567890ABC \
  --paths "/index.html" "/config.json"


Invalidate all paths:

bash
aws cloudfront create-invalidation \
  --distribution-id E1234567890ABC \
  --paths "/*"


Quote wildcard paths in the shell.

Recommended strategy
Versioned JavaScript/CSS: long TTL.
HTML entry point: shorter TTL or revalidation.
Public APIs: short TTL if appropriate.
User-specific APIs: disable shared caching unless the design explicitly supports safe reuse.

Senior-level insight: Content-hashed filenames usually provide more predictable deployments than repeatedly invalidating the same asset URLs.

Invalidation does not fix a broken origin, an incorrect cache key, or a stale browser cache.

5. Cache Behaviors and Origins

A distribution can contain multiple origins and cache behaviors.

Example:

Path	Origin	Caching
/assets/*	Private S3	Long TTL
/images/*	Private S3	Medium/long TTL
/api/*	ALB/API Gateway	Disabled or selective
Default *	Frontend origin	Shorter HTML TTL
Important settings
Path pattern.
Target origin.
Allowed HTTP methods.
Viewer protocol policy.
Cache policy.
Origin request policy.
Response headers policy.
Trusted key groups.
Edge-function associations.

Tricky question: Can CloudFront route requests to multiple origins?

Yes. Different cache behaviors can route different paths to different origins. Edge processing can support more advanced origin selection.

Important: Ensure the cache key matches the content identity. Requests routed to different content sources must not accidentally share an inappropriate cache entry.

6. S3 Security: OAC vs OAI

Recommended architecture:

text
Internet users
      |
      v
CloudFront
      |
      | Signed origin requests
      v
Private S3 bucket

Origin Access Control (OAC)

OAC lets CloudFront securely access a private S3 origin.

A typical setup requires:

Private S3 bucket.
OAC attached to the CloudFront S3 origin.
Bucket policy allowing the CloudFront service principal to read the required objects.
Restriction to the intended distribution using AWS:SourceArn.
KMS permissions if SSE-KMS is used.
No unintended public access permissions.

Example bucket-policy statement:

json
{
  "Effect": "Allow",
  "Principal": {
    "Service": "cloudfront.amazonaws.com"
  },
  "Action": "s3:GetObject",
  "Resource": "arn:aws:s3:::my-private-site/*",
  "Condition": {
    "StringEquals": {
      "AWS:SourceArn": "arn:aws:cloudfront::123456789012:distribution/E1234567890ABC"
    }
  }
}


This is a policy statement fragment, not a complete bucket policy.

OAC vs OAI
Feature	OAC	OAI
New deployments	Preferred	Legacy approach
SigV4 signing	Supported	More limited
SSE-KMS	Supported with permissions	Not supported in the same way
S3 website endpoint	Not supported as a private S3 REST origin configuration	Different legacy limitations

Interview question: What happens if someone accesses the S3 URL directly?

If the bucket is private and the request has no independent authorization, S3 denies direct access. CloudFront's read permission does not grant anonymous users direct S3 access.

Important distinction: S3 website endpoints are custom HTTP origins. For OAC, use an appropriate S3 REST origin configuration.

7. DNS and TLS

To serve www.example.com through CloudFront:

Provision an ACM certificate for the hostname.
Configure the alternate domain name on the distribution.
Attach the certificate.
Configure DNS to point to the distribution.
Enforce the required HTTPS policy.

Classic interview question: In which region must the ACM viewer certificate be created for CloudFront?

Answer: us-east-1 (US East, N. Virginia), regardless of the origin's AWS Region.

DNS options
Route 53 alias record.
CNAME through an external DNS provider where supported.
Alias or DNS-flattening functionality for apex domains.
Viewer TLS vs origin TLS

These are separate connections:

Viewer to CloudFront.
CloudFront to the origin.

A valid viewer certificate does not guarantee that CloudFront can establish HTTPS to the origin.

Check:

Origin protocol policy.
Certificate expiration.
Hostname matching.
Certificate chain.
Origin TLS compatibility.
8. CloudFront Functions vs Lambda@Edge
Feature	CloudFront Functions	Lambda@Edge
Main use	Lightweight transformations	More complex edge processing
Common events	Viewer request/response	Viewer request/response, origin request/response
URL rewrites	Yes	Yes
Header manipulation	Yes	Yes
Complex processing	More limited	Broader capabilities
Origin selection	Not the usual choice for advanced origin routing	Suitable for supported origin-request use cases
CloudFront Functions use cases
URL redirects.
URI normalization.
Header manipulation.
Simple request validation.
Lightweight viewer-side processing.
Lambda@Edge use cases
More complex authorization logic.
Dynamic origin selection.
Supported processing requiring network access or external libraries.
Supported request-body processing use cases.
Four Lambda@Edge event types
Event	Timing
Viewer request	Before cache lookup
Origin request	When the request proceeds toward the origin
Origin response	After origin response, before caching
Viewer response	Before response reaches the viewer, subject to invocation restrictions

Tricky question: Why might an origin-request function not run for a cache hit?

Because the cached response can be served without proceeding to the origin-request stage.

Best practice: Use built-in CloudFront settings for standard HTTP-to-HTTPS redirects rather than writing custom code.

9. WAF and Security

CloudFront integrates with AWS WAF for:

Managed attack rules.
IP allow/deny lists.
Rate-based rules.
Geographic restrictions.
Custom request inspection.
Security logging and alerting.

A production design might use:

text
Users
  |
  v
CloudFront + WAF
  |
  v
ALB / API Gateway
  |
  v
Application


Critical distinction: WAF is not a replacement for authentication or authorization. A request can pass WAF and still be rejected by the application.

For sensitive APIs:

Disable shared caching when appropriate.
Forward only required authentication data.
Ensure response-varying request attributes are represented in the cache key if caching is enabled.
Test requests from different user sessions.
Keep origin access appropriately restricted.
10. API Caching and Data Leakage

Consider:

http
GET /api/profile
Authorization: Bearer USER_A_TOKEN


The origin returns User A's private profile.

If CloudFront reuses the response for another user, private data may leak.

Prevention
Disable shared caching for sensitive responses when appropriate.
Authenticate and authorize at the appropriate application layer.
Do not assume forwarding an authorization header makes caching safe.
Include every relevant response-varying attribute in the cache key if shared caching is intentional.
Use appropriate origin Cache-Control directives.
Test cross-user and cross-tenant requests.
Example behavior layout
Path	Caching
/assets/*	Long-lived
/api/catalog	Short TTL if public and safe
/api/profile	Disabled
/api/orders/*	Usually disabled

Senior-level rule: Cache-key design is a security decision, not just a performance decision.

11. Origin Failover and Error Handling

CloudFront supports origin groups with primary and secondary origins.

text
CloudFront
    |
    v
Primary origin
    |
    | Configured failover condition
    v
Secondary origin


Evaluate:

Configured failover status codes.
Whether the secondary origin has equivalent content.
Independent failure domains.
Data consistency.
Monitoring and recovery procedures.

Origin failover is not a replacement for application redundancy, database resilience, or multi-region data replication.

Custom error responses

CloudFront can map selected origin errors to custom pages.

Example: mapping a 403 to an SPA entry point.

Risk: A missing JavaScript file might return HTML with status 200, masking the real error.

Keep genuine missing assets distinguishable from application routes, and inspect both origin and viewer response statuses.

12. Monitoring and Troubleshooting
Important metrics and signals
Signal	Purpose
Requests	Traffic volume
Cache hit rate	Cache efficiency
Bytes downloaded	Viewer delivery volume
Bytes uploaded	Traffic sent toward origins
4xx errors	Request, access, or client problems
5xx errors	Origin or infrastructure problems
Origin latency	Origin response performance
WAF blocked requests	Security rule activity
Age and X-Cache	Individual-response caching clues
Troubleshooting checklist
DNS: Does the hostname resolve to the intended distribution?
TLS: Is the certificate valid for the hostname?
Distribution: Is it deployed and configured correctly?
Behavior: Does the URL match the intended behavior?
WAF: Is a rule blocking the request?
Cache: Is the response stale or unexpected?
Origin: Is it healthy and reachable?
Permissions: Can CloudFront access the object?
Application: Does the origin return the correct response?
Logs: Can the viewer request be correlated with origin logs?
Common errors
Error	Possible causes
403	WAF block, origin permissions, OAC, KMS, or access denial
502	Origin TLS, connection, or response problems
503	Origin overload, unavailable origin, or other service-side conditions
504	Origin timeout or connectivity problems
Stale content	TTL, cache key, browser cache, or deployment strategy
Redirect loop	Mismatched HTTPS and forwarded-protocol handling

These are clues, not definitive diagnoses.

Debugging command
bash
curl -sS -D - -o /dev/null \
  "https://www.example.com/assets/app.js"


Inspect:

http
HTTP/2 200
cache-control: public, max-age=31536000
age: 120
x-cache: Hit from cloudfront


Compare the result with the current origin content and deployment version.

13. Production Architectures
A. Static frontend
text
Route 53
   |
   v
CloudFront + ACM + WAF
   |
   v
Private S3 bucket via OAC


Best practices:

Keep S3 private.
Use OAC.
Use long TTLs for versioned assets.
Keep HTML freshness shorter.
Automate deployments.
Use least-privilege deployment IAM roles.
B. Frontend plus API
text
                CloudFront
                /        \
        /assets/*        /api/*
            |                |
            v                v
        Private S3         ALB/API Gateway
                               |
                               v
                          Application


Best practices:

Separate static and API behaviors.
Restrict allowed methods.
Avoid shared caching for sensitive responses.
Forward required headers and query strings deliberately.
Monitor cache performance and origin latency separately.
C. Multi-region or disaster recovery

CloudFront can front multiple origins, but origin selection, health handling, failover, and data consistency must be designed explicitly.

A primary/secondary origin group does not automatically create a multi-region active-active application.

14. Terraform Example

The following is a simplified CloudFront distribution example:

hcl
resource "aws_cloudfront_distribution" "site" {
  enabled         = true
  is_ipv6_enabled = true
  comment         = "Production frontend"

  aliases = ["www.example.com"]

  origin {
    domain_name              = aws_s3_bucket.site.bucket_regional_domain_name
    origin_id                = "private-s3-site"
    origin_access_control_id = aws_cloudfront_origin_access_control.site.id
  }

  default_cache_behavior {
    target_origin_id       = "private-s3-site"
    viewer_protocol_policy = "redirect-to-https"

    allowed_methods = ["GET", "HEAD"]
    cached_methods  = ["GET", "HEAD"]

    cache_policy_id = aws_cloudfront_cache_policy.site.id
  }

  viewer_certificate {
    acm_certificate_arn      = aws_acm_certificate.site.arn
    ssl_support_method       = "sni-only"
    minimum_protocol_version = "TLSv1.2_2021"
  }

  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }
}


This is not a complete deployable configuration. Additional resources include:

S3 bucket policy.
OAC definition.
Cache policy.
ACM certificate validation.
DNS configuration.
Optional WAF and logging resources.
CI/CD deployment sequence
Build the application.
Upload versioned assets.
Upload the updated HTML entry point.
Run smoke tests.
Invalidate selected paths if required.
Verify the application version through CloudFront.
Monitor errors and roll back when necessary.

Important: Upload new assets before publishing HTML that references them.

15. Advanced Caching Concepts
Cache key explosion

Every additional cache-key dimension can increase the number of variants.

For example:

Query parameters.
Cookies.
Headers.
Device variants.

Include only attributes that materially change the response.

Query parameter ordering

These URLs might represent equivalent requests:

text
/products?a=1&b=2
/products?b=2&a=1


Do not assume application-level equivalence guarantees identical CloudFront cache keys. Normalize when appropriate and test the configured behavior.

Versioned filenames

Prefer:

text
app.8a31f2.js


over repeatedly replacing:

text
app.js


Content-hashed assets support long TTLs and safer deployments.

Minimum TTL overrides

A positive minimum TTL can cause CloudFront to retain responses even when the origin requests no caching. This is especially important for sensitive content.

Origin changes

Changing the origin doesn't necessarily eliminate cached responses associated with the existing cache key. Plan invalidation or versioned URL changes when immediate replacement is required.

CDN consistency

CloudFront is not a transactional database consistency mechanism. TTLs and invalidations do not guarantee that every cache layer and browser instantly reflects a database update.

16. Easy Interview Questions
Q1. What is CloudFront?

Answer: A CDN that delivers content globally through edge locations, reducing latency and potentially reducing origin load.

Q2. Is CloudFront only for static content?

Answer: No. It supports dynamic content, APIs, and streaming use cases as well.

Q3. What is an origin?

Answer: The backend source of content, such as S3, ALB, EC2, or API Gateway.

Q4. What is a cache hit?

Answer: CloudFront serves a usable cached response without needing a new origin response.

Q5. What is TTL?

Answer: The freshness lifetime configured through caching rules.

Q6. What is invalidation?

Answer: A request to remove specified objects from CloudFront caches so subsequent requests can obtain fresh content.

Q7. CloudFront vs Route 53?

Answer: Route 53 provides DNS services; CloudFront delivers content through a CDN.

Q8. OAC vs OAI?

Answer: OAC is the preferred modern method for securing S3 origins.

Q9. Where must an ACM certificate for CloudFront be provisioned?

Answer: us-east-1.

Q10. CloudFront Functions vs Lambda@Edge?

Answer: Functions suit lightweight viewer processing; Lambda@Edge supports more complex processing and additional event types.

17. Medium Interview Questions
Q11. Cache policy vs origin request policy?

Answer: The cache policy controls cache identity and TTL. The origin request policy controls additional request values forwarded to the origin.

Q12. CloudFront serves an old JavaScript file. What do you do?

Answer:

Inspect Age, X-Cache, and Cache-Control.
Verify the origin object.
Check the cache key and TTL.
Invalidate the affected path if needed.
Use versioned filenames in future deployments.
Q13. Private S3 returns 403. Why?

Answer: Check OAC, bucket policy, distribution ARN restrictions, KMS permissions, object existence, and origin configuration.

Q14. no-cache vs no-store?

Answer: no-cache permits storage but requires revalidation before reuse. no-store instructs caches not to store the response.

Q15. Why not forward all headers?

Answer: Unnecessary forwarding increases complexity, can reduce caching efficiency, and may cause origin compatibility issues.

Q16. How do you configure an SPA?

Answer: Use private S3 plus OAC, appropriate HTML and asset TTLs, and carefully scoped SPA routing. Avoid returning HTML for missing static assets.

Q17. How do you improve cache hit ratio?

Answer: Reduce unnecessary cache-key dimensions, normalize URLs, use versioned assets, set suitable TTLs, and cache public API responses only when safe.

Q18. Redirect HTTP to HTTPS vs HTTPS Only?

Answer: Redirect mode redirects HTTP requests; HTTPS-only rejects them.

Q19. How does WAF help?

Answer: It evaluates configured security rules, such as managed rules and rate-based rules. It does not replace application authorization.

Q20. Origin works directly, but CloudFront returns 502. Why?

Answer: Investigate TLS, hostname matching, protocol configuration, connection problems, and differences between direct and CloudFront requests.

18. Hard Interview Questions
Q21. Two users receive the same API response despite different cookies. Why?

Answer: The cache key probably doesn't distinguish the relevant cookie values. Disable shared caching or include the required response-varying attributes.

Q22. X-Tenant-ID is forwarded but not in the cache key. Is that safe?

Answer: Only if the response is identical across tenants. Tenant-specific responses require appropriate cache separation or disabled shared caching.

Q23. An edge function changes the URI, but cached content is unexpected. Why?

Answer: Viewer-request processing occurs before cache lookup; origin-request processing occurs after cache lookup. The event timing and cache-key design must match the intended content identity.

Q24. A new origin is configured, but old content is still served. Why?

Answer: Existing cached objects may still satisfy requests. Check deployment status, cache key, and invalidation requirements.

Q25. Can CloudFront cache POST responses?

Answer: Do not assume so. CloudFront can forward configured methods, but normal shared caching is primarily designed around GET/HEAD responses.

Q26. An API varies by header, but the same response is returned for all values. Why?

Answer: The header may be forwarded but excluded from the cache key. Include the relevant header in the cache key or normalize the representation into the URL.

Q27. Origin failover vs active-active?

Answer: Origin failover uses configured failure conditions to select a secondary origin. Active-active requires separate traffic distribution, health management, and data consistency strategies.

Q28. Why might invalidation not fix stale content?

Answer: The origin may be stale, the wrong path may have been invalidated, another cache may be involved, or the origin may return the same stale response again.

Q29. How can a cache policy create a security vulnerability?

Answer: Different users may share the same cache key despite requiring different responses, resulting in cross-user data leakage.

Q30. Minimum TTL is 60 seconds and origin sends no-store. What happens?

Answer: A positive minimum TTL can override the origin's intended no-store behavior. Use minimum TTL zero for sensitive content and verify the rest of the configuration.

Q31. Origin returns 404, but viewer sees 200. Why?

Answer: CloudFront custom error mapping may return a different status code or custom error page.

Q32. Why can a low hit ratio increase costs?

Answer: More traffic reaches origins, potentially increasing request charges, compute utilization, and origin data transfer. CloudFront delivery charges also remain relevant.

Q33. Regional edge cache vs edge location?

Answer: Edge locations serve viewer requests; regional edge caches provide another caching layer for eligible requests.

Q34. How do you debug intermittent 403s?

Answer: Capture request details, inspect CloudFront and WAF logs, identify the responding layer, and check permissions, signed content, behavior matching, and request differences.

Q35. How do you implement rollback?

Answer: Preserve known-good artifacts and configuration, use versioned assets, deploy in a controlled order, revert the appropriate changes, invalidate only when necessary, and verify the restored version.

19. Expert Production Scenarios
Q36. Hit ratio is 95%, but the ALB is overloaded. Why?

Answer: The remaining 5% might contain expensive, non-cacheable requests. Analyze origin traffic by path, method, response status, and workload cost.

Q37. Serve different content to mobile and desktop users?

Answer: Prefer responsive design where possible. Otherwise use separate URLs or normalized device classification and ensure the cache key distinguishes the required variants.

Q38. How do you stop users bypassing CloudFront and accessing the ALB directly?

Answer: Restrict origin ingress appropriately, validate origin requests, and maintain application-level authentication. A secret custom header can add protection but should not be the only security boundary.

Q39. Can authenticated pages be cached?

Answer: Only when the content and authorization design make shared reuse safe. Otherwise disable shared caching and honor appropriate origin cache directives.

Q40. Cache hit ratio drops after a release. How do you debug?

Answer:

Compare metrics before and after release.
Inspect cache policies and behaviors.
Check query strings, cookies, and headers.
Review URL changes and edge-function modifications.
Check origin TTL headers.
Roll back the change if evidence supports it.
Q41. CloudFront is healthy, but the application is down. Will CloudFront solve it?

Answer: No. Cached content may remain available, but dynamic and uncached requests can fail. Origin resilience is still required.

Q42. Millions of files need invalidation on every release. What would you change?

Answer: Use content-hashed asset filenames, long TTLs for immutable assets, shorter TTLs for HTML, and publish assets before updating the entry point.

20. Senior DevOps Checklist
Architecture
Explain request flow and cache layers.
Design multiple origins and path-based behaviors.
Explain origin failover limitations.
Caching
Explain cache keys, TTLs, and invalidation.
Troubleshoot stale content and cache misses.
Prevent cross-user and cross-tenant data leakage.
Security
Configure private S3 origins with OAC.
Explain viewer and origin TLS.
Design WAF and least-privilege origin access.
Understand signed URLs and cookies.
Operations
Diagnose 403, 502, 503, and 504 responses.
Correlate CloudFront and origin logs.
Optimize cache hit ratio without compromising correctness.
Understand cost and availability tradeoffs.
Automation
Deploy CloudFront with Terraform or CloudFormation.
Design safe CI/CD releases and rollbacks.
Test cache, behavior, and security changes.
Official AWS Resources
CloudFront Developer Guide
Cache and origin request policies
TTL and expiration
Private S3 origin access
CloudFront Functions and Lambda@Edge
Final Interview Advice

For a five-year DevOps interview, don't just memorize definitions. Explain the tradeoffs, identify the failure modes, and describe how you would validate a fix using logs, metrics, and reproducible tests.

The most important topics to master are cache-key design, TTL behavior, OAC, TLS, API cache isolation, origin failover, and production troubleshooting.
