# Amazon CloudFront — Senior DevOps Engineer Notes

**Level:** 5+ Years DevOps / SRE / Cloud Engineer  
**Topics:** Architecture, caching, security, Terraform, troubleshooting, and interview questions

---

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

### Main benefits
* Lower latency for users.
* Reduced origin load through caching.
* Global content delivery.
* Integration with AWS WAF and AWS Shield protections.
* HTTPS and custom-domain support.
* Private S3 origin access using Origin Access Control (OAC).
* Support for static websites, APIs, downloads, and streaming content.
* Request and response processing at the edge.

### Common origins
* Amazon S3
* Application Load Balancer (ALB)
* EC2
* API Gateway
* External HTTP/HTTPS servers

> **Senior-level insight:** CloudFront is more than a cache. It is also a TLS endpoint, routing layer, security integration point, and request-processing platform.

---

## 2. Architecture and Request Flow

### Main components

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

### Request flow
1. A user requests `https://www.example.com/assets/app.js`.
2. DNS directs the hostname to CloudFront.
3. The browser establishes HTTPS with CloudFront.
4. Applicable viewer-side protections, including AWS WAF, are evaluated.
5. CloudFront selects the matching cache behavior.
6. CloudFront constructs the cache key.
7. If a usable cached response exists, CloudFront returns it.
8. Otherwise, CloudFront obtains or revalidates content through its caching hierarchy and origin.
9. CloudFront returns the response and caches it when permitted by the configuration and response rules.

### Cache hit vs. cache miss
* **Cache hit:** A usable cached response is available.
  * *Benefits:* Lower latency, less origin traffic, potentially lower origin infrastructure costs.
* **Cache miss:** CloudFront needs to obtain or revalidate content.
  * *Consequences:* Greater origin dependency, potentially higher latency, more origin requests.

### Useful response headers
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

### Cache policy
Controls:
* Cache-key query strings.
* Cache-key headers.
* Cache-key cookies.
* Minimum TTL, Default TTL, Maximum TTL.

### Origin request policy
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

### Interview question
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
The effective freshness lifetime can be $120$ seconds, subject to the applicable policy and caching rules. If the origin sends a freshness lifetime longer than the maximum TTL, the configured maximum limits it.

### Important Cache-Control directives
| Directive | Meaning |
| :--- | :--- |
| `public` | Response may be stored by shared caches |
| `private` | Intended for private caches, not shared-cache storage under normal HTTP semantics |
| `no-cache` | Storage may be allowed, but revalidation is required before reuse |
| `no-store` | Instructs caches not to store the response |
| `max-age=300` | Freshness lifetime of $300$ seconds |

> **Critical trap:** A positive CloudFront minimum TTL can override the origin's intent not to cache. Use a minimum TTL of zero for sensitive content and when origin cache directives must be honored.

### Invalidation
Invalidate cached objects when content must be refreshed before normal expiration:
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
*(Always quote wildcard paths in the shell).*

### Recommended strategy
* Versioned JavaScript/CSS: long TTL.
* HTML entry point: shorter TTL or revalidation.
* Public APIs: short TTL if appropriate.
* User-specific APIs: disable shared caching unless the design explicitly supports safe reuse.

> **Senior-level insight:** Content-hashed filenames usually provide more predictable deployments than repeatedly invalidating the same asset URLs. Invalidation does not fix a broken origin, an incorrect cache key, or a stale browser cache.

---

## 5. Cache Behaviors and Origins

A distribution can contain multiple origins and cache behaviors.

### Example layout
| Path | Origin | Caching |
| :--- | :--- | :--- |
| `/assets/*` | Private S3 | Long TTL |
| `/images/*` | Private S3 | Medium/long TTL |
| `/api/*` | ALB/API Gateway | Disabled or selective |
| Default `*` | Frontend origin | Shorter HTML TTL |

### Important settings
* Path pattern.
* Target origin.
* Allowed HTTP methods.
* Viewer protocol policy.
* Cache policy, Origin request policy, Response headers policy.
* Trusted key groups.
* Edge-function associations.

> **Tricky question:** Can CloudFront route requests to multiple origins? Yes. Different cache behaviors can route different paths to different origins. Edge processing can support more advanced origin selection. Ensure the cache key matches the content identity.

---

## 6. S3 Security: OAC vs. OAI

### Recommended architecture
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
1. Private S3 bucket.
2. OAC attached to the CloudFront S3 origin.
3. Bucket policy allowing the CloudFront service principal to read the required objects.
4. Restriction to the intended distribution using `AWS:SourceArn`.
5. KMS permissions if `SSE-KMS` is used.
6. No unintended public access permissions.

### Example bucket-policy statement
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

### OAC vs. OAI
| Feature | OAC | OAI |
| :--- | :--- | :--- |
| New deployments | Preferred | Legacy approach |
| SigV4 signing | Supported | More limited |
| SSE-KMS | Supported with permissions | Not supported in the same way |
| S3 website endpoint | Not supported as a private S3 REST origin configuration | Different legacy limitations |

> **Interview question:** What happens if someone accesses the S3 URL directly? If the bucket is private and the request has no independent authorization, S3 denies direct access. CloudFront's read permission does not grant anonymous users direct S3 access.

---

## 7. DNS and TLS

To serve `www.example.com` through CloudFront:
1. Provision an ACM certificate for the hostname.
2. Configure the alternate domain name on the distribution.
3. Attach the certificate.
4. Configure DNS to point to the distribution.
5. Enforce the required HTTPS policy.

> **Classic interview question:** In which region must the ACM viewer certificate be created for CloudFront?  
> **Answer:** `us-east-1` (US East, N. Virginia), regardless of the origin's AWS Region.

### DNS options
* Route 53 alias record.
* CNAME through an external DNS provider where supported.
* Alias or DNS-flattening functionality for apex domains.

### Viewer TLS vs. origin TLS
These are separate connections:
1. Viewer to CloudFront.
2. CloudFront to the origin.

> A valid viewer certificate does not guarantee that CloudFront can establish HTTPS to the origin. Check origin protocol policy, certificate expiration, hostname matching, certificate chain, and origin TLS compatibility.

---

## 8. CloudFront Functions vs. Lambda@Edge

| Feature | CloudFront Functions | Lambda@Edge |
| :--- | :--- | :--- |
| **Main use** | Lightweight transformations | More complex edge processing |
| **Common events** | Viewer request/response | Viewer request/response, origin request/response |
| **URL rewrites** | Yes | Yes |
| **Header manipulation** | Yes | Yes |
| **Complex processing** | More limited | Broader capabilities |
| **Origin selection** | Not typical for advanced routing | Suitable for supported origin-request use cases |

### Four Lambda@Edge event types
* **Viewer request:** Before cache lookup.
* **Origin request:** When the request proceeds toward the origin.
* **Origin response:** After origin response, before caching.
* **Viewer response:** Before response reaches the viewer, subject to invocation restrictions.

> **Tricky question:** Why might an origin-request function not run for a cache hit? Because the cached response can be served without proceeding to the origin-request stage.

---

## 9. WAF and Security

CloudFront integrates with AWS WAF for:
* Managed attack rules.
* IP allow/deny lists.
* Rate-based rules.
* Geographic restrictions.
* Custom request inspection.
* Security logging and alerting.

> **Critical distinction:** WAF is not a replacement for authentication or authorization. A request can pass WAF and still be rejected by the application.

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

> **Senior-level rule:** Cache-key design is a security decision, not just a performance decision.

---

## 11. Origin Failover and Error Handling

CloudFront supports origin groups with primary and secondary origins:
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

### Custom error responses
CloudFront can map selected origin errors to custom pages (e.g., mapping a `403` to an SPA entry point). 

> **Risk:** A missing JavaScript file might return HTML with status `200`, masking the real error. Keep genuine missing assets distinguishable from application routes.

---

## 12. Monitoring and Troubleshooting

### Important metrics and signals
* **Requests & Cache hit rate:** Traffic volume & cache efficiency.
* **Bytes downloaded / uploaded:** Viewer delivery volume & origin traffic.
* **4xx / 5xx errors:** Client problems vs. origin/infrastructure problems.
* **Origin latency & WAF blocked requests:** Performance & security rule activity.

### Common error codes
* `403`: WAF block, origin permissions, OAC, KMS, or access denial.
* `502`: Origin TLS, connection, or response problems.
* `503`: Origin overload, unavailable origin, or service-side conditions.
* `504`: Origin timeout or connectivity problems.

### Debugging command
```bash
curl -sS -D - -o /dev/null \
  "https://www.example.com/assets/app.js"
```

---

## 13. Production Architectures

### A. Static frontend
* **Flow:** Route 53 $\rightarrow$ CloudFront + ACM + WAF $\rightarrow$ Private S3 bucket via OAC.
* **Best practices:** Keep S3 private, use OAC, use long TTLs for versioned assets, automate deployments.

### B. Frontend plus API
* **Flow:** CloudFront splits traffic: `/assets/*` $\rightarrow$ Private S3; `/api/*` $\rightarrow$ ALB/API Gateway.
* **Best practices:** Separate behaviors, restrict allowed methods, avoid shared caching for sensitive responses.

---

## 14. Terraform Example

```hcl
resource "aws_cloudfront_distribution" "site" {
  enabled             = true
  is_ipv6_enabled     = true
  comment             = "Production frontend"

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

---

## 15. Advanced Caching Concepts
* **Cache key explosion:** Every extra cache-key dimension increases variants. Include only attributes that materially change the response.
* **Query parameter ordering:** URL order does not guarantee identical cache keys. Normalize when appropriate.
* **Versioned filenames:** Prefer `app.8a31f2.js` over repeatedly replacing `app.js`.

---

## 16-20. Interview Questions & Checklist

Mastering these concepts prepares engineers for high-level SRE/DevOps architectural rounds focusing on cache key isolation, OAC security implementation, DNS/TLS topology, and robust production debugging.
