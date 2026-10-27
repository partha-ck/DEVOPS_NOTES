# Amazon CloudFront — Senior DevOps Engineer Notes

**Level:** 5+ Years DevOps / SRE / Cloud Engineer

**Topics:** Architecture, caching, security, Terraform, troubleshooting, and interview questions

## Table of Contents

1. [What Is CloudFront?](#1-what-is-cloudfront)
2. [Architecture and Request Flow](#2-architecture-and-request-flow)
3. [Cache Policies vs. Origin Request Policies](#3-cache-policies-vs-origin-request-policies)
4. [TTL, Cache-Control, and Invalidation](#4-ttl-cache-control-and-invalidation)
5. [Cache Behaviors and Origins](#5-cache-behaviors-and-origins)
6. [S3 Security: OAC vs. OAI](#6-s3-security-oac-vs-oai)
7. [DNS and TLS](#7-dns-and-tls)
8. [CloudFront Functions vs. Lambda@Edge](#8-cloudfront-functions-vs-lambdaedge)
9. [WAF and Security](#9-waf-and-security)
10. [API Caching and Data Leakage](#10-api-caching-and-data-leakage)
11. [Origin Failover and Error Handling](#11-origin-failover-and-error-handling)
12. [Monitoring and Troubleshooting](#12-monitoring-and-troubleshooting)
13. [Production Architectures](#13-production-architectures)
14. [Terraform Example](#14-terraform-example)
15. [Advanced Caching Concepts](#15-advanced-caching-concepts)
16. [Easy Interview Questions](#16-easy-interview-questions)
17. [Medium Interview Questions](#17-medium-interview-questions)
18. [Hard Interview Questions](#18-hard-interview-questions)
19. [Expert Production Scenarios](#19-expert-production-scenarios)
20. [Senior DevOps Checklist](#20-senior-devops-checklist)

---

## 1. What Is CloudFront?

Amazon CloudFront is AWS's Content Delivery Network (CDN). It delivers static and dynamic content through a globally distributed network of edge locations.

### Main Benefits
* Lower latency for users.
* Reduced origin load through caching.
* Global content delivery.
* Integration with AWS WAF and AWS Shield protections.
* HTTPS and custom-domain support.
* Private S3 origin access using Origin Access Control (OAC).
* Support for static websites, APIs, downloads, and streaming content.
* Request and response processing at the edge.

### Common Origins
* Amazon S3
* Application Load Balancer (ALB)
* EC2
* API Gateway
* External HTTP/HTTPS servers

> **Senior-level insight:** CloudFront is more than a cache. It is also a TLS endpoint, routing layer, security integration point, and request-processing platform.

---

## 2. Architecture and Request Flow

### Main Components

| Component | Purpose |
| :--- | :--- |
| **Distribution** | Defines how CloudFront delivers content |
| **Origin** | Backend source of content |
| **Cache behavior** | Matches paths and configures caching, methods, and policies |
| **Edge location** | Serves viewer requests |
| **Regional edge cache** | Additional caching layer for eligible requests |
| **Cache key** | Determines which requests can share a cached response |
| **Cache policy** | Defines cache-key inputs and TTL settings |
| **Origin request policy** | Controls additional viewer request values forwarded to the origin |
| **OAC** | Secures CloudFront access to an S3 origin |
| **CloudFront Functions** | Lightweight edge request/response processing |
| **Lambda@Edge** | More complex edge processing |

### Request Flow
1. A user requests `https://www.example.com/assets/app.js`.
2. DNS directs the hostname to CloudFront.
3. The browser establishes HTTPS with CloudFront.
4. Applicable viewer-side protections, including AWS WAF, are evaluated.
5. CloudFront selects the matching cache behavior.
6. CloudFront constructs the cache key.
7. If a usable cached response exists, CloudFront returns it.
8. Otherwise, CloudFront obtains or revalidates content through its caching hierarchy and origin.
9. CloudFront returns the response and caches it when permitted by the configuration and response rules.

### Cache Hit vs. Cache Miss
* **Cache hit:** A usable cached response is available.
  * *Benefits:* Lower latency, less origin traffic, potentially lower origin infrastructure costs.
* **Cache miss:** CloudFront needs to obtain or revalidate content.
  * *Consequences:* Greater origin dependency, potentially higher latency, more origin requests.

### Useful Response Headers
```http
X-Cache: Hit from cloudfront
X-Cache: Miss from cloudfront
X-Cache: RefreshHit from cloudfront
Age: 120
```

> **Note:** A cache miss does not necessarily mean the origin itself was contacted; another cache layer or request path may satisfy the request.

---

## 3. Cache Policies vs. Origin Request Policies

This is one of the most important CloudFront concepts.

### Cache Policy
Controls:
* Cache-key query strings.
* Cache-key headers.
* Cache-key cookies.
* Minimum TTL, Default TTL, Maximum TTL.

### Origin Request Policy
Controls additional viewer request values forwarded to the origin without necessarily including those values in the cache key. Values included in the cache key are also automatically forwarded to the origin.

### Example
Requests:
```text
/products?id=101&campaign=google
/products?id=101&campaign=facebook
```

If the response depends only on `id`, a suitable design could be:
* Include `id` in the cache key.
* Forward `campaign` separately if the origin needs it for analytics.
* Exclude `campaign` from the cache key.

Both requests may then share a cached product response.

> **Warning:** This is safe only if `campaign` does not affect the response. If it changes the content, the cache key must account for it.

### Interview Question
* **Q:** If a cookie is forwarded to the origin, is CloudFront caching separately for each cookie value?
* **Answer:** No. Forwarding alone does not partition the cache. Include the relevant cookie in the cache key or disable shared caching if the response is user-specific.

---

## 4. TTL, Cache-Control, and Invalidation

TTL means time to live. CloudFront cache policies define:
* Minimum TTL
* Default TTL
* Maximum TTL

### Example
```text
Minimum TTL = 0 seconds
Default TTL = 300 seconds
Maximum TTL = 3600 seconds
```

If the origin sends:
```http
Cache-Control: public, max-age=120
```
the effective freshness lifetime can be 120 seconds, subject to the applicable policy and caching rules. If the origin sends a freshness lifetime longer than the maximum TTL, the configured maximum limits it.

### Important Cache-Control Directives
| Directive | Meaning |
| :--- | :--- |
| `public` | Response may be stored by shared caches |
| `private` | Intended for private caches, not shared-cache storage under normal HTTP semantics |
| `no-cache` | Storage may be allowed, but revalidation is required before reuse |
| `no-store` | Instructs caches not to store the response |
| `max-age=300` | Freshness lifetime of 300 seconds |

> **Critical Trap:** A positive CloudFront minimum TTL can override the origin's intent not to cache. Use a minimum TTL of zero for sensitive content and when origin cache directives must be honored.

### Invalidation
Invalidate cached objects when content must be refreshed before normal expiration.
```bash
aws cloudfront create-invalidation \
  --distribution-id E1234567890ABC \
  --paths "/index.html" "/config.json"
```

Invalidate all paths:
```bash
aws cloudfront create-invalidation \
  --distribution-id E1234567890ABC \
  --paths "/*"
```
*(Quote wildcard paths in the shell.)*

### Recommended Strategy
* Versioned JavaScript/CSS: long TTL.
* HTML entry point: shorter TTL or revalidation.
* Public APIs: short TTL if appropriate.
* User-specific APIs: disable shared caching unless the design explicitly supports safe reuse.

> **Senior-level insight:** Content-hashed filenames usually provide more predictable deployments than repeatedly invalidating the same asset URLs. Invalidation does not fix a broken origin, an incorrect cache key, or a stale browser cache.

---

## 5. Cache Behaviors and Origins

A distribution can contain multiple origins and cache behaviors.

### Example Mapping
| Path Pattern | Target Origin | Caching Strategy |
| :--- | :--- | :--- |
| `/assets/*` | Private S3 | Long TTL |
| `/images/*` | Private S3 | Medium/long TTL |
| `/api/*` | ALB/API Gateway | Disabled or selective |
| Default `*` | Frontend origin | Shorter HTML TTL |

### Important Settings
* Path pattern
* Target origin
* Allowed HTTP methods
* Viewer protocol policy
* Cache policy
* Origin request policy
* Response headers policy
* Trusted key groups
* Edge-function associations

> **Tricky Question:** Can CloudFront route requests to multiple origins? 
> **Answer:** Yes. Different cache behaviors can route different paths to different origins. Edge processing can support more advanced origin selection. Ensure the cache key matches the content identity. Requests routed to different content sources must not accidentally share an inappropriate cache entry.

---

## 6. S3 Security: OAC vs. OAI

### Recommended Architecture
```text
Internet users
      |
      v
CloudFront
      |
      | Signed origin requests
      v
Private S3 bucket
```

### Origin Access Control (OAC)
OAC lets CloudFront securely access a private S3 origin. A typical setup requires:
* Private S3 bucket.
* OAC attached to the CloudFront S3 origin.
* Bucket policy allowing the CloudFront service principal to read required objects.
* Restriction to the intended distribution using `AWS:SourceArn`.
* KMS permissions if SSE-KMS is used.
* No unintended public access permissions.

### Example Bucket Policy Statement
```json
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
```
*(This is a policy statement fragment, not a complete bucket policy.)*

### OAC vs. OAI
| Feature | OAC | OAI |
| :--- | :--- | :--- |
| **New deployments** | Preferred | Legacy approach |
| **SigV4 signing** | Supported | More limited |
| **SSE-KMS** | Supported with permissions | Not supported in the same way |
| **S3 website endpoint** | Not supported as a private S3 REST origin configuration | Different legacy limitations |

> **Interview Question:** What happens if someone accesses the S3 URL directly?
> **Answer:** If the bucket is private and the request has no independent authorization, S3 denies direct access. CloudFront's read permission does not grant anonymous users direct S3 access. *(Note: S3 website endpoints are custom HTTP origins. For OAC, use an appropriate S3 REST origin configuration.)*

---

## 7. DNS and TLS

To serve `www.example.com` through CloudFront:
1. Provision an ACM certificate for the hostname.
2. Configure the alternate domain name on the distribution.
3. Attach the certificate.
4. Configure DNS to point to the distribution.
5. Enforce the required HTTPS policy.

> **Classic Interview Question:** In which region must the ACM viewer certificate be created for CloudFront?
> **Answer:** `us-east-1` (US East, N. Virginia), regardless of the origin's AWS Region.

### DNS Options
* Route 53 alias record.
* CNAME through an external DNS provider where supported.
* Alias or DNS-flattening functionality for apex domains.

### Viewer TLS vs. Origin TLS
These are separate connections:
1. Viewer to CloudFront.
2. CloudFront to the origin.

A valid viewer certificate does not guarantee that CloudFront can establish HTTPS to the origin. Always check origin protocol policy, certificate expiration, hostname matching, certificate chain, and origin TLS compatibility.

---

## 8. CloudFront Functions vs. Lambda@Edge

| Feature | CloudFront Functions | Lambda@Edge |
| :--- | :--- | :--- |
| **Main use** | Lightweight transformations | More complex edge processing |
| **Common events** | Viewer request/response | Viewer request/response, origin request/response |
| **URL rewrites** | Yes | Yes |
| **Header manipulation** | Yes | Yes |
| **Complex processing** | More limited | Broader capabilities |
| **Origin selection** | Not the usual choice for advanced origin routing | Suitable for supported origin-request use cases |

### CloudFront Functions Use Cases
* URL redirects.
* URI normalization.
* Header manipulation.
* Simple request validation.
* Lightweight viewer-side processing.

### Lambda@Edge Use Cases
* More complex authorization logic.
* Dynamic origin selection.
* Supported processing requiring network access or external libraries.
* Supported request-body processing use cases.

### Four Lambda@Edge Event Types
| Event | Timing |
| :--- | :--- |
| **Viewer request** | Before cache lookup |
| **Origin request** | When the request proceeds toward the origin |
| **Origin response** | After origin response, before caching |
| **Viewer response** | Before response reaches the viewer, subject to invocation restrictions |

> **Tricky Question:** Why might an origin-request function not run for a cache hit?
> **Answer:** Because the cached response can be served without proceeding to the origin-request stage. 
> *Best Practice:* Use built-in CloudFront settings for standard HTTP-to-HTTPS redirects rather than writing custom code.

---

## 9. WAF and Security

CloudFront integrates with AWS WAF for:
* Managed attack rules.
* IP allow/deny lists.
* Rate-based rules.
* Geographic restrictions.
* Custom request inspection.
* Security logging and alerting.

### Production Design Pattern
```text
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
```

> **Critical Distinction:** WAF is not a replacement for authentication or authorization. A request can pass WAF and still be rejected by the application.

### For Sensitive APIs
* Disable shared caching when appropriate.
* Forward only required authentication data.
* Ensure response-varying request attributes are represented in the cache key if caching is enabled.
* Test requests from different user sessions.
* Keep origin access appropriately restricted.

---

## 10. API Caching and Data Leakage

Consider:
```http
GET /api/profile
Authorization: Bearer USER_A_TOKEN
```
The origin returns User A's private profile. If CloudFront reuses the response for another user, private data may leak.

### Prevention
* Disable shared caching for sensitive responses when appropriate.
* Authenticate and authorize at the appropriate application layer.
* Do not assume forwarding an authorization header makes caching safe.
* Include every relevant response-varying attribute in the cache key if shared caching is intentional.
* Use appropriate origin `Cache-Control` directives.
* Test cross-user and cross-tenant requests.

### Example Behavior Layout
| Path | Caching Strategy |
| :--- | :--- |
| `/assets/*` | Long-lived |
| `/api/catalog` | Short TTL if public and safe |
| `/api/profile` | Disabled |
| `/api/orders/*` | Usually disabled |

> **Senior-level rule:** Cache-key design is a security decision, not just a performance decision.

---

## 11. Origin Failover and Error Handling

CloudFront supports origin groups with primary and secondary origins.
```text
CloudFront
    |
    v
Primary origin
    |
    | Configured failover condition
    v
Secondary origin
```

Evaluate:
* Configured failover status codes.
* Whether the secondary origin has equivalent content.
* Independent failure domains.
* Data consistency.
* Monitoring and recovery procedures.

*(Note: Origin failover is not a replacement for application redundancy, database resilience, or multi-region data replication.)*

### Custom Error Responses
CloudFront can map selected origin errors to custom pages (e.g., mapping a `403` to an SPA entry point). 
* **Risk:** A missing JavaScript file might return HTML with status `200`, masking the real error. Keep genuine missing assets distinguishable from application routes, and inspect both origin and viewer response statuses.

---

## 12. Monitoring and Troubleshooting

### Important Metrics and Signals
| Signal | Purpose |
| :--- | :--- |
| **Requests** | Traffic volume |
| **Cache hit rate** | Cache efficiency |
| **Bytes downloaded** | Viewer delivery volume |
| **Bytes uploaded** | Traffic sent toward origins |
| **4xx errors** | Request, access, or client problems |
| **5xx errors** | Origin or infrastructure problems |
| **Origin latency** | Origin response performance |
| **WAF blocked requests** | Security rule activity |
| **Age and X-Cache** | Individual-response caching clues |

### Troubleshooting Checklist
* **DNS:** Does the hostname resolve to the intended distribution?
* **TLS:** Is the certificate valid for the hostname?
* **Distribution:** Is it deployed and configured correctly?
* **Behavior:** Does the URL match the intended behavior?
* **WAF:** Is a rule blocking the request?
* **Cache:** Is the response stale or unexpected?
* **Origin:** Is it healthy and reachable?
* **Permissions:** Can CloudFront access the object?
* **Application:** Does the origin return the correct response?
* **Logs:** Can the viewer request be correlated with origin logs?

### Common Errors
| Error | Possible Causes |
| :--- | :--- |
| **403** | WAF block, origin permissions, OAC, KMS, or access denial |
| **502** | Origin TLS, connection, or response problems |
| **503** | Origin overload, unavailable origin, or other service-side conditions |
| **504** | Origin timeout or connectivity problems |
| **Stale content** | TTL, cache key, browser cache, or deployment strategy |
| **Redirect loop** | Mismatched HTTPS and forwarded-protocol handling |

### Debugging Command
```bash
curl -sS -D - -o /dev/null \
  "https://www.example.com/assets/app.js"
```
Inspect HTTP/2 status, cache control, age, and `X-Cache` headers, comparing them against origin content and deployment versions.

---

## 13. Production Architectures

### A. Static Frontend
```text
Route 53
   |
   v
CloudFront + ACM + WAF
   |
   v
Private S3 bucket via OAC
```
*Best practices:* Keep S3 private, use OAC, use long TTLs for versioned assets, keep HTML freshness shorter, automate deployments, and use least-privilege deployment IAM roles.

### B. Frontend Plus API
```text
               CloudFront
              /          \
      /assets/*          /api/*
          |              |
          v              v
      Private S3     ALB/API Gateway
                         |
                         v
                    Application
```
*Best practices:* Separate static and API behaviors, restrict allowed methods, avoid shared caching for sensitive responses, forward required headers and query strings deliberately, and monitor cache performance and origin latency separately.

### C. Multi-Region or Disaster Recovery
CloudFront can front multiple origins, but origin selection, health handling, failover, and data consistency must be designed explicitly. A primary/secondary origin group does not automatically create a multi-region active-active application.

---

## 14. Terraform Example

```hcl
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
```

### CI/CD Deployment Sequence
1. Build the application.
2. Upload versioned assets.
3. Upload the updated HTML entry point.
4. Run smoke tests.
5. Invalidate selected paths if required.
6. Verify the application version through CloudFront.
7. Monitor errors and roll back when necessary.

> **Important:** Upload new assets before publishing HTML that references them.

---

## 15. Advanced Caching Concepts

* **Cache key explosion:** Every additional cache-key dimension (query params, cookies, headers, device variants) can increase variants. Include only attributes that materially change the response.
* **Query parameter ordering:** `/products?a=1&b=2` and `/products?b=2&a=1` do not guarantee identical CloudFront cache keys. Normalize when appropriate.
* **Versioned filenames:** Prefer `app.8a31f2.js` over repeatedly replacing `app.js`. Content-hashed assets support long TTLs and safer deployments.
* **Minimum TTL overrides:** A positive minimum TTL can cause CloudFront to retain responses even when the origin requests no caching.
* **Origin changes:** Changing the origin doesn't necessarily eliminate cached responses associated with the existing cache key.
* **CDN consistency:** CloudFront is not a transactional database consistency mechanism. TTLs and invalidations do not guarantee instant global propagation.

---

## 16. Easy Interview Questions

* **Q1. What is CloudFront?** A CDN that delivers content globally through edge locations, reducing latency and origin load.
* **Q2. Is CloudFront only for static content?** No. It supports dynamic content, APIs, and streaming use cases as well.
* **Q3. What is an origin?** The backend source of content, such as S3, ALB, EC2, or API Gateway.
* **Q4. What is a cache hit?** CloudFront serves a usable cached response without needing a new origin response.
* **Q5. What is TTL?** The freshness lifetime configured through caching rules.
* **Q6. What is invalidation?** A request to remove specified objects from CloudFront caches.
* **Q7. CloudFront vs Route 53?** Route 53 provides DNS services; CloudFront delivers content through a CDN.
* **Q8. OAC vs OAI?** OAC is the preferred modern method for securing S3 origins.
* **Q9. Where must an ACM certificate for CloudFront be provisioned?** `us-east-1`.
* **Q10. CloudFront Functions vs Lambda@Edge?** Functions suit lightweight viewer processing; Lambda@Edge supports more complex processing and additional event types.

---

## 17. Medium Interview Questions

* **Q11. Cache policy vs origin request policy?** Cache policy controls cache identity and TTL; origin request policy controls additional values forwarded to the origin.
* **Q12. CloudFront serves an old JavaScript file. What do you do?** Inspect Age, X-Cache, and Cache-Control; verify origin; check cache key/TTL; invalidate if needed; use versioned filenames.
* **Q13. Private S3 returns 403. Why?** Check OAC, bucket policy, distribution ARN restrictions, KMS permissions, and object existence.
* **Q14. no-cache vs no-store?** `no-cache` permits storage but requires revalidation before reuse; `no-store` instructs caches not to store the response at all.
* **Q15. Why not forward all headers?** Increases complexity, reduces caching efficiency, and causes origin compatibility issues.
* **Q16. How do you configure an SPA?** Private S3 plus OAC, appropriate HTML/asset TTLs, and scoped SPA routing without returning HTML for missing static assets.
* **Q17. How do you improve cache hit ratio?** Reduce unnecessary cache-key dimensions, normalize URLs, use versioned assets, and set suitable TTLs.
* **Q18. Redirect HTTP to HTTPS vs HTTPS Only?** Redirect mode redirects HTTP requests; HTTPS-only rejects them.
* **Q19. How does WAF help?** Evaluates security rules; does not replace application authorization.
* **Q20. Origin works directly, but CloudFront returns 502. Why?** Investigate TLS, hostname matching, protocol configuration, connection problems, and differences between direct/CloudFront requests.

---

## 18. Hard Interview Questions

* **Q21. Two users receive the same API response despite different cookies. Why?** Cache key likely doesn't distinguish cookie values. Disable shared caching or include required attributes.
* **Q22. X-Tenant-ID is forwarded but not in the cache key. Is that safe?** Only if the response is identical across tenants. Tenant-specific responses require appropriate cache separation.
* **Q23. An edge function changes the URI, but cached content is unexpected. Why?** Viewer-request processing occurs before cache lookup; origin-request processing occurs after cache lookup. Event timing must match.
* **Q24. A new origin is configured, but old content is still served. Why?** Existing cached objects satisfy requests; check deployment status, cache key, and invalidation rules.
* **Q25. Can CloudFront cache POST responses?** Do not assume so. Shared caching is primarily designed around GET/HEAD responses.
* **Q26. An API varies by header, but the same response is returned. Why?** The header may be forwarded but excluded from the cache key. Include it in the cache key.
* **Q27. Origin failover vs active-active?** Origin failover uses configured failure conditions for a secondary origin; active-active requires separate traffic distribution and data consistency strategies.
* **Q28. Why might invalidation not fix stale content?** Origin may be stale, wrong path invalidated, another cache involved, or origin returns the same stale response.
* **Q29. How can a cache policy create a security vulnerability?** Different users sharing the same cache key despite needing different responses, resulting in cross-user data leakage.
* **Q30. Minimum TTL is 60 seconds and origin sends no-store. What happens?** Positive minimum TTL can override the origin's intended `no-store` behavior.
* **Q31. Origin returns 404, but viewer sees 200. Why?** CloudFront custom error mapping may return a different status code or custom error page.
* **Q32. Why can a low hit ratio increase costs?** More traffic reaches origins, increasing request charges and compute utilization.
* **Q33. Regional edge cache vs edge location?** Edge locations serve viewer requests; regional edge caches provide another caching layer for eligible requests.
* **Q34. How do you debug intermittent 403s?** Capture request details, inspect logs, identify responding layer, and check permissions and signed content.
* **Q35. How do you implement rollback?** Preserve known-good artifacts, use versioned assets, deploy in a controlled order, revert changes, and verify.

---

## 19. Expert Production Scenarios

* **Q36. Hit ratio is 95%, but the ALB is overloaded. Why?** Remaining 5% might contain expensive, non-cacheable requests. Analyze origin traffic by path, method, and workload cost.
* **Q37. Serve different content to mobile and desktop users?** Prefer responsive design. Otherwise use separate URLs or normalized device classification with cache-key separation.
* **Q38. How do you stop users bypassing CloudFront and accessing the ALB directly?** Restrict origin ingress, validate origin requests, and maintain application-level authentication.
* **Q39. Can authenticated pages be cached?** Only when content and authorization design make shared reuse safe; otherwise disable shared caching.
* **Q40. Cache hit ratio drops after a release. How do you debug?** Compare metrics before/after release, inspect cache policies, check query strings/cookies/headers, review URL changes, and check TTL headers.
* **Q41. CloudFront is healthy, but the application is down. Will CloudFront solve it?** No. Cached content may remain available, but dynamic requests can fail. Origin resilience is required.
* **Q42. Millions of files need invalidation on every release. What would you change?** Use content-hashed asset filenames, long TTLs for immutable assets, shorter TTLs for HTML, and publish assets before updating the entry point.

---

## 20. Senior DevOps Checklist

### Architecture
* Explain request flow and cache layers.
* Design multiple origins and path-based behaviors.
* Explain origin failover limitations.

### Caching
* Explain cache keys, TTLs, and invalidation.
* Troubleshoot stale content and cache misses.
* Prevent cross-user and cross-tenant data leakage.

### Security
* Configure private S3 origins with OAC.
* Explain viewer and origin TLS.
* Design WAF and least-privilege origin access.
* Understand signed URLs and cookies.

### Operations
* Diagnose 403, 502, 503, and 504 responses.
* Correlate CloudFront and origin logs.
* Optimize cache hit ratio without compromising correctness.
* Understand cost and availability tradeoffs.

### Automation
* Deploy CloudFront with Terraform or CloudFormation.
* Design safe CI/CD releases and rollbacks.
* Test cache, behavior, and security changes.

---

## Official AWS Resources
* [CloudFront Developer Guide](https://docs.aws.amazon.com/cloudfront/)
* [Cache and origin request policies](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/controlling-the-cache-key.html)
* [TTL and expiration](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Expiration.html)
* [Private S3 origin access](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-restricting-access-to-s3.html)
* [CloudFront Functions and Lambda@Edge](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/edge-workloads.html)

---

## Final Interview Advice
For a five-year DevOps interview, don't just memorize definitions. Explain the tradeoffs, identify the failure modes, and describe how you would validate a fix using logs, metrics, and reproducible tests. The most important topics to master are cache-key design, TTL behavior, OAC, TLS, API cache isolation, origin failover, and production troubleshooting.
