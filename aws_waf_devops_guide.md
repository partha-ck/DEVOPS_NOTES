Here’s your `waf.md` file, formatted as a structured Markdown guide with tables, code examples, architecture diagrams, interview questions, and checklists.

Writing

# 🛡️ AWS WAF: Complete DevOps Engineer Guide

## Production Security, Architecture, Terraform, Troubleshooting & Interview Preparation

**Target audience:** DevOps Engineers with approximately 5 years of experience\
**Level:** Beginner to Advanced\
**Focus:** Production implementation, security engineering, automation, and interviews

---

## 📚 Table of Contents

1. Learning Roadmap
2. AWS WAF Fundamentals
3. Core WAF Concepts
4. Advanced Architecture and Operations
5. Production Scenario: CloudFront + WAF + ALB
6. Interview Questions and Answers
7. DevOps Skills Assessment
8. Hands-On Lab
9. Quick Revision Cheat Sheet
10. Official AWS References

---

# Learning Roadmap

| Level | Focus | What You Should Learn |
| --- | --- | --- |
| 1 | Fundamentals | Web ACLs, rules, rule groups, actions, priorities |
| 2 | Security engineering | SQL injection, XSS, bots, rate limiting, managed rules |
| 3 | Production architecture | CloudFront, ALB, API Gateway, origin protection |
| 4 | DevOps implementation | Terraform, CI/CD, logging, monitoring, cost optimization |
| 5 | Interview preparation | Scenario-based questions and production troubleshooting |

# Part 1 — AWS WAF Fundamentals

## 1\. What Exactly Is AWS WAF?

**AWS WAF (Web Application Firewall)** is a managed security service that inspects HTTP and HTTPS web requests and evaluates them against configured rules.

Depending on the rule and supported action, AWS WAF can:

- Allow legitimate requests.
- Block unwanted requests.
- Count matching requests without blocking them.
- Apply CAPTCHA or Challenge actions.
- Rate-limit excessive traffic.
- Detect common web exploits.
- Filter requests based on IP addresses, paths, headers, query strings, and request bodies.
- Integrate with AWS Managed Rules and supported bot protection capabilities.

### Common threats addressed by WAF

| Threat | Example | Possible WAF Protection |
| --- | --- | --- |
| SQL injection | Malicious SQL fragments in request parameters | SQL injection inspection |
| Cross-site scripting (XSS) | Script payloads in user-controlled input | XSS inspection |
| Brute-force attempts | Excessive requests to `/login` | Rate-based rules |
| Malicious IP traffic | Requests from known hostile addresses | IP reputation rules |
| Automated bots | Repeated scraping or abusive automation | Bot Control and other controls |
| Application abuse | Excessive requests to sensitive APIs | Custom rate-based rules |
| Suspicious HTTP traffic | Malformed or unexpected request patterns | Managed and custom rules |

> **Important:** AWS WAF is one layer of defense in depth. It does not replace application authentication, authorization, secure coding, network security, or dedicated DDoS protections.

Official documentation: [https://docs.aws.amazon.com/waf/latest/developerguide/](<https://docs.aws.amazon.com/waf/latest/developerguide/>)

## 2\. Where Does AWS WAF Sit in the Architecture?

A common architecture for a public web application is:

text

```
                   Internet Users
                         |
                         v
                     Route 53
                    DNS Resolution
                         |
                         v
                      CloudFront
                         |
                         v
                    AWS WAF Web ACL
                         |
                         v
                Application Load Balancer
                         |
                         v
                 EC2 / ECS / EKS
                         |
                         v
                      Database
```

This diagram represents a possible architecture, not a mandatory chain. A Web ACL can protect supported resources such as CloudFront distributions, Application Load Balancers, and API Gateway APIs.

Route 53 resolves DNS names. CloudFront delivers content and forwards requests. AWS WAF inspects and filters requests. The ALB distributes application traffic.

### CloudFront WAF vs. ALB WAF

| Area | CloudFront-associated WAF | ALB-associated WAF |
| --- | --- | --- |
| Inspection location | CloudFront edge | Regional ALB |
| Scope | `CLOUDFRONT` | `REGIONAL` |
| Typical use | Public websites and global APIs | Regional application entry points |
| Blocks before application servers | Yes | Yes |
| Protects direct origin access automatically | No | Protects traffic reaching the ALB |
| Typical deployment | Global Web ACL | Regional Web ACL |

### Production concern: Origin bypass

If WAF protects CloudFront but the origin ALB remains publicly accessible, attackers may bypass CloudFront and send requests directly to the ALB.

Mitigation strategies include:

- Restricting origin network access where practical.
- Using CloudFront VPC origins where supported by the architecture.
- Using a carefully designed origin-authentication mechanism.
- Validating origin requests and trusted upstream identity.
- Testing whether the origin can be reached directly.

A custom origin header alone is insufficient if attackers can reach the origin and reproduce the header.

Reference: [https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/](<https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/>)

# Part 2 — Core AWS WAF Concepts

## 3\. Web ACL, Rules, Rule Groups, and Statements

| Component | Meaning | Example |
| --- | --- | --- |
| Web ACL | Top-level collection of rules associated with a protected resource | `production-web-acl` |
| Rule | Defines matching conditions and an action | Block traffic from a prohibited IP |
| Rule statement | Defines the matching logic | URI path equals `/admin` |
| Rule group | Reusable collection of rules | AWS Common Rule Set |
| Rule priority | Determines evaluation order | Priority `10` before `20` |
| Default action | Applies when no terminating rule determines the result | Allow or Block |
| WCU | Web ACL capacity unit | Measures rule-processing capacity |

A rule is defined within a Web ACL or rule group; it is not a standalone AWS resource.

### Example Web ACL

| Priority | Rule | Action |
| --- | --- | --- |
| 0 | Narrowly scoped trusted-monitoring exception | Allow, if intentional |
| 10 | Known malicious IP protection | Block |
| 20 | Login rate limiting | Block when threshold is exceeded |
| 30 | AWS Common Rule Set | Managed rule actions |
| Default | Default Web ACL action | Allow or Block |

**Security lesson:** Broad early Allow rules can bypass later protections because Allow is a terminating action. Use exceptions only when their scope and effect are understood.

## 4\. Rule Priority and Terminating Actions

AWS WAF evaluates rules in ascending numerical priority.

- Priority `0` is evaluated before priority `10`.
- A terminating action stops further evaluation.
- A non-terminating action allows evaluation to continue.

Example:

text

```
Priority 0:  Allow trusted IP addresses
Priority 10: Block malicious request patterns
Priority 20: Count requests to /login
Default:     Allow
```

If a request matches the first Allow rule, later SQL injection, XSS, and rate-based rules in that Web ACL may not evaluate it.

### WAF actions

| Action | Behavior |
| --- | --- |
| Allow | Allows the request and terminates Web ACL evaluation |
| Block | Blocks the request; the default response is HTTP 403 |
| Count | Records a match and continues evaluation |
| CAPTCHA | Requests CAPTCHA validation when required |
| Challenge | Performs a browser-session challenge when required |

CAPTCHA and Challenge behavior depends on the client's token status and the configured action. They are not interchangeable with ordinary blocking.

## 5\. AWS Managed Rules

AWS Managed Rules are predefined rule groups maintained by AWS or supported providers.

| Rule Group | Purpose |
| --- | --- |
| `AWSManagedRulesCommonRuleSet` | General web attack patterns |
| `AWSManagedRulesSQLiRuleSet` | SQL injection-related protection |
| `AWSManagedRulesKnownBadInputsRuleSet` | Known malicious or invalid input patterns |
| `AWSManagedRulesAmazonIpReputationList` | IP addresses associated with malicious activity |
| `AWSManagedRulesAnonymousIpList` | Certain anonymizing services and sources |
| AWS Bot Control rule group | Bot detection and mitigation capabilities |

Some managed rule groups and features incur additional charges. Confirm the current pricing and availability before deployment.

### Safe managed-rule rollout

1. Deploy to a test environment.
2. Configure suitable rule-action overrides for testing.
3. Use Count mode where appropriate.
4. Exercise legitimate application workflows.
5. Review logs and identify false positives.
6. Apply narrowly scoped exclusions or overrides.
7. Enable blocking gradually.
8. Monitor production after deployment.

**Important:** A managed rule group does not necessarily block every match. Its behavior depends on the configured rule actions and overrides.

## 6\. Custom Rules and Request Inspection

AWS WAF supports inspecting request components such as:

- URI paths.
- Query strings.
- HTTP methods.
- Headers.
- Source IP addresses.
- Request bodies, subject to applicable inspection limits.
- Geographic attributes.
- Labels generated by other rules.

Rules can combine logical `AND`, `OR`, and `NOT` statements.

### Example: Protect `/admin`

Requirement: Block requests to `/admin` unless the source belongs to an approved IP range.

text

```
IF URI path matches /admin
AND source IP is NOT in the trusted IP set
THEN BLOCK
```

Use precise path matching to avoid unintentionally matching paths such as `/administrator`.

Remember that a path rule does not replace authentication or authorization.

### Scope-down statements

A scope-down statement restricts which requests a rule or managed rule group evaluates.

Example:

text

```
Rate-based rule
    |
    +-- Scope down:
    |      URI path = /login
    |
    +-- Aggregate requests by source IP
    |
    +-- Block when the rate threshold is exceeded
```

This avoids applying a login-specific rate limit to every application endpoint.

## 7\. Rate-Based Rules

Rate-based rules help mitigate:

- Login brute-force attempts.
- Abusive API traffic.
- Scraping.
- Excessive requests.
- Traffic spikes from a source or defined request category.

A rate-based rule tracks requests according to configured aggregation criteria and applies its configured action when the rate exceeds the threshold.

### Example policy

text

```
Endpoint:           /login
Evaluation window:  5 minutes
Threshold:          500 requests
Aggregation:        Source IP
Action:             Block
```

This is an illustrative configuration, not a universal recommendation.

### Important rate-limiting considerations

- Thresholds must reflect normal application traffic.
- Rate-based enforcement is approximate, not an exact transactional quota.
- NAT gateways, corporate proxies, and mobile carrier networks may place legitimate users behind one public IP.
- Distributed attackers can spread requests across many IP addresses.
- Each rate-based rule instance maintains its own tracking.
- Identical rules in separate Web ACLs do not automatically share one global counter.
- Rate limiting should be combined with application-side account protection.

### Recommended login protection layers

text

```
                   Login Requests
                         |
                         v
                     AWS WAF
                         |
              +----------+----------+
              |                     |
              v                     v
        Rate Limiting          Managed Rules
              |                     |
              +----------+----------+
                         |
                         v
                    Application
                         |
              +----------+----------+
              |                     |
              v                     v
        Account Controls          MFA
              |
              v
      Authentication Service
```

Use account-level throttling, MFA, risk-based authentication, and suitable bot protection alongside WAF.

For exact business quotas, implement application-side enforcement or use an appropriate API management layer.

## 8\. Labels: Advanced WAF Feature

Labels allow a rule to classify a request so that later rules can make more specific decisions.

text

```
Rule 1: Detect suspicious login patterns
                |
                v
       Add a request label
                |
                v
Rule 2: Match the label
                |
                v
 Apply a targeted security action
```

Possible uses:

- Apply stricter rate limits to suspicious traffic.
- Use managed-rule detections as signals for later policies.
- Classify requests before blocking.
- Avoid duplicating complex inspection logic.

Labels must be evaluated in the correct rule order and within their applicable scope.

# Part 3 — Advanced Architecture and Operations

## 9\. Web ACL Capacity Units (WCUs)

WCUs measure the processing capacity required by WAF rules.

Capacity depends on factors such as:

- Statement types.
- Nested logical statements.
- Managed rule groups.
- Text transformations.
- Request inspection configuration.

Check the current AWS service quotas and capacity requirements for your configuration.

**Interview trap:** WCUs are not a direct measurement of requests per second. They represent rule-processing capacity.

## 10\. Logging, Metrics, and Troubleshooting

Production engineers need to understand which rule blocked a request and why.

### Important telemetry

| Signal | Purpose |
| --- | --- |
| Allowed requests | Tracks permitted traffic |
| Blocked requests | Tracks rejected traffic |
| Counted requests | Tracks matches in Count mode |
| Sampled requests | Provides selected request examples |
| WAF logs | Records detailed request evaluation information |
| Rule labels | Helps identify request classifications |

CloudWatch metrics and sampled requests are useful, but they do not replace a properly configured logging strategy.

### Troubleshooting HTTP 403 responses

1. Determine whether WAF or the application generated the response.
2. Find the corresponding WAF log event.
3. Inspect the terminating rule ID and action.
4. Review labels and relevant request metadata.
5. Reproduce the request in a safe test environment.
6. Identify the specific false-positive rule, if any.
7. Make the narrowest appropriate adjustment.
8. Test and deploy the change.
9. Verify that legitimate traffic succeeds and malicious traffic remains blocked.

Avoid disabling an entire managed rule group simply because one rule produces a false positive.

## 11\. AWS WAF with Terraform and CI/CD

Infrastructure as code enables repeatable, reviewable WAF deployments.

A typical implementation includes:

- `aws_wafv2_web_acl`
- Managed rule groups and custom rules
- `aws_wafv2_web_acl_association` where applicable
- Logging configuration
- CloudWatch metrics and alarms
- CI/CD validation and deployment
- Rollback procedures

### Example: Regional Web ACL with AWS Managed Rules

hcl

```
resource "aws_wafv2_web_acl" "app" {
  name        = "production-app-waf"
  description = "Baseline web application protection"
  scope       = "REGIONAL"

  default_action {
    allow {}
  }

  rule {
    name     = "AWSCommonRules"
    priority = 10

    override_action {
      none {}
    }

    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesCommonRuleSet"
        vendor_name = "AWS"
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "AWSCommonRules"
      sampled_requests_enabled   = true
    }
  }

  visibility_config {
    cloudwatch_metrics_enabled = true
    metric_name                = "ProductionAppWAF"
    sampled_requests_enabled   = true
  }
}
```

**Note:** This configuration uses the managed group's configured rule actions. It is not automatically a Count-mode rollout. Configure rule-action overrides when you want to test detections without enforcing the normal actions.

For an ALB, use the appropriate regional Web ACL association. For CloudFront, configure a Web ACL with `CLOUDFRONT` scope using the required CloudFront-related deployment configuration.

Validate Terraform syntax, provider versions, resource associations, and AWS quotas before deploying.

### Recommended CI/CD workflow

text

```
Developer changes Terraform
           |
           v
     Pull request
           |
           v
  Formatting and validation
           |
           v
    Security review
           |
           v
      Terraform plan
           |
           v
   Staging deployment
           |
           v
    Traffic testing
           |
           v
   Production approval
           |
           v
  Production deployment
           |
           v
   Metrics and log review
           |
           v
   Rollback if required
```

# Part 4 — Production Scenario: CloudFront + WAF + ALB

## 12\. Architecture and Security Design

### Requirements

- Global users access the application through CloudFront.
- The application runs behind an ALB.
- Login requests require rate limiting.
- SQL injection and XSS patterns must be detected.
- Attackers must not bypass CloudFront to reach the origin directly.
- Security events must be logged and monitored.

### Proposed architecture

text

```
                    Internet Users
                          |
                          v
                      CloudFront
                          |
                          v
                  CloudFront Web ACL
                          |
              +-----------+-----------+
              |           |           |
              v           v           v
          Rate Limit   Managed Rules  IP Reputation
              |           |           |
              +-----------+-----------+
                          |
                          v
                  Protected ALB Origin
                          |
                          v
                   Application Servers
                          |
                          v
                       Database

       Operations and Observability
       ----------------------------
       WAF Logs
       CloudWatch Metrics
       Alarms and Dashboards
       Incident Response
```

### Recommended controls

1. Attach a CloudFront-scoped Web ACL to the distribution.
2. Configure AWS Managed Rules for common web attacks.
3. Enable SQL injection and XSS-related inspections through suitable managed or custom rules.
4. Configure a rate-based rule scoped to `/login`.
5. Tune thresholds based on observed traffic.
6. Enable logging and monitoring.
7. Restrict direct access to the origin.
8. Add application-side login throttling and account security controls.
9. Test legitimate and malicious traffic before enforcement.

### Origin bypass prevention

CloudFront WAF does not inspect requests that bypass CloudFront and directly reach a publicly accessible ALB.

Use appropriate network restrictions and origin validation. Test direct-origin access explicitly as part of deployment validation.

### False-positive handling

If legitimate checkout or login requests are blocked:

1. Find the request in WAF logs.
2. Identify the terminating rule.
3. Inspect the matched component.
4. Determine whether the request is genuinely malicious.
5. Apply a narrowly scoped exclusion or rule-action override.
6. Test the application workflow.
7. Deploy the change with monitoring.

# Part 5 — Interview Questions and Answers

## Level 1: Beginner

### 1\. What is AWS WAF?

AWS WAF is a managed web application firewall that inspects HTTP/HTTPS requests and applies configured security rules.

### 2\. What is a Web ACL?

A Web ACL is the top-level collection of rules and a default action associated with a supported AWS resource.

### 3\. What is the difference between a managed rule and a custom rule?

Managed rules are maintained by AWS or another provider. Custom rules are designed and maintained by your team for application-specific requirements.

### 4\. What does the WAF Block action return?

By default, a blocked request receives HTTP 403. The response can be customized.

### 5\. Does AWS WAF require a `/health` endpoint?

No. WAF inspects requests; a health endpoint is not required for WAF to evaluate traffic.

### 6\. What is the difference between Route 53 health checks and WAF?

Route 53 health checks assess configured endpoint health and can influence DNS routing. WAF inspects incoming web requests and enforces security policies.

## Level 2: Intermediate

### 7\. What is the difference between Count and Allow?

Count records a match and continues evaluation. Allow permits the request and terminates Web ACL evaluation.

### 8\. How would you protect `/login` from brute-force traffic?

Create a rate-based rule scoped to `/login`, choose suitable aggregation criteria, test with Count mode, and enforce a suitable threshold. Combine it with application-side account protections.

### 9\. Why use a scope-down statement?

It restricts the requests evaluated by a rule, such as applying a rate limit only to `/login`.

### 10\. What are WCUs?

WCUs measure the processing capacity required by WAF rules and rule groups.

### 11\. How do you investigate an unexpected 403?

Inspect the WAF logs, identify the terminating rule, reproduce the request, determine whether it is a false positive, and apply a narrowly scoped fix.

### 12\. Can two Web ACLs share one rate-based counter?

No. Separate rate-based rule instances maintain separate tracking.

## Level 3: Advanced

### 13\. CloudFront has WAF, but an attacker is bypassing it. Why?

The origin may be directly accessible. Restrict origin access and implement suitable origin validation.

### 14\. A managed rule blocks legitimate JSON requests. How do you fix it?

Inspect the WAF logs, identify the exact rule and matched component, reproduce the issue, and use a narrow exclusion or action override. Test before enforcing the change.

### 15\. How do you rate-limit one endpoint without throttling all traffic?

Use a rate-based rule with a scope-down statement matching the required URI path and method where appropriate.

### 16\. How do you deploy a new WAF policy safely?

Validate in staging, use Count mode where appropriate, inspect logs, approve the change through CI/CD, deploy gradually, monitor, and retain a rollback procedure.

### 17\. What is the difference between WAF and Shield?

WAF focuses on web request inspection and filtering. AWS Shield provides managed DDoS protection. The services complement one another.

### 18\. What happens if an early Allow rule matches a malicious request?

The Allow action terminates evaluation, so later rules in that Web ACL do not inspect the request.

## Level 4: Tricky Scenario-Based Questions

### 19\. A rate-based rule has a threshold of 1,000 requests, but you observe more than 1,000. Is WAF broken?

Not necessarily. Rate-based enforcement is approximate and may lag or fluctuate around the configured threshold.

Check:

- Aggregation criteria.
- Scope-down statements.
- Distributed source IP addresses.
- The evaluation window.
- Whether traffic is being counted across separate rule instances.

Do not treat WAF rate limiting as an exact business quota.

### 20\. Can WAF blindly trust `X-Forwarded-For`?

No. Forwarded headers can be manipulated unless the trusted proxy chain is correctly designed and validated.

Understand how the source IP is determined, how trusted proxies populate the header, and how direct-origin access is restricted.

### 21\. Should you remove an entire managed rule group because of one false positive?

Usually not. Identify the exact internal rule, inspect the match, and apply a narrow exclusion or action override.

### 22\. Can WAF block a request based on a database error returned later by the application?

Not directly simply because the database returned an error. WAF primarily evaluates incoming requests and configured inspection criteria.

Use application monitoring, application logic, or other suitable controls when decisions depend on backend errors.

### 23\. Will WAF always detect a malicious string beyond the request-body inspection limit?

No. Request-body inspection has applicable limits. Review the relevant body-inspection limit and oversize handling configuration, and enforce validation in the application as well.

### 24\. Does attaching a Web ACL automatically configure security logging?

No. Configure an appropriate logging destination and retention strategy. Consider access controls, privacy, sensitive request data, and logging costs.

### 25\. If CloudFront and ALB each have WAF, do they share rules and rate counters?

No. Each Web ACL evaluates traffic at its own layer. Rules, logging, and rate-based tracking must be managed accordingly.

# Part 6 — DevOps Skills Assessment

## Production Readiness Checklist

- [ ]Explain Web ACL evaluation order and terminating actions.
- [ ]Build custom IP, URI, header, and rate-based rules.
- [ ]Deploy AWS Managed Rules safely.
- [ ]Diagnose and fix false positives.
- [ ]Explain CloudFront versus ALB WAF scope.
- [ ]Prevent direct-origin bypass.
- [ ]Configure WAF logging and CloudWatch monitoring.
- [ ]Deploy WAF using Terraform and CI/CD.
- [ ]Troubleshoot production 403 responses using logs.
- [ ]Explain WCUs, rate-based tracking, and cost trade-offs.
- [ ]Design controlled rollouts and rollback procedures.
- [ ]Explain how WAF complements application security and DDoS protection.

Check each item only when you can explain the concept and demonstrate it in a practical example.

# Part 7 — Hands-On Lab

## Recommended Lab Architecture

text

```
Client
  |
  v
CloudFront Distribution
  |
  v
AWS WAF Web ACL
  |
  +-- AWS Managed Rules
  |
  +-- Login Rate-Based Rule
  |
  +-- IP Reputation Protection
  |
  +-- Logging and Metrics
  |
  v
Application Origin
```

## Lab Tasks

1. Create a test application behind CloudFront.
2. Create and associate a CloudFront-scoped Web ACL.
3. Enable an AWS Managed Rule group.
4. Configure a rate-based rule scoped to `/login`.
5. Start with Count mode where appropriate.
6. Generate legitimate test requests.
7. Send controlled requests containing test attack patterns.
8. Review WAF logs and sampled requests.
9. Identify and correct any false positives.
10. Enable blocking after validation.
11. Verify direct-origin access is appropriately restricted.
12. Automate deployment using Terraform.
13. Configure monitoring and alarms.
14. Document rollback and incident-response procedures.

**Safety note:** Perform testing only against infrastructure you own or are explicitly authorized to test. Avoid high-volume load tests against production systems without an approved test plan.

# Part 8 — Quick Revision Cheat Sheet

## WAF Actions

| Action | Key Behavior |
| --- | --- |
| Allow | Terminates evaluation and permits the request |
| Block | Terminates evaluation and rejects the request |
| Count | Records a match and continues |
| CAPTCHA | Requests CAPTCHA validation when needed |
| Challenge | Challenges the client when required |

## Architecture Responsibilities

| Service | Responsibility |
| --- | --- |
| Route 53 | DNS resolution and traffic steering |
| CloudFront | Edge delivery and content caching |
| AWS WAF | Web request inspection and filtering |
| AWS Shield | DDoS protection |
| ALB | Application-layer traffic distribution |
| Application | Authentication, authorization, business logic |
| CloudWatch | Metrics, alarms, and operational monitoring |
| CloudTrail | AWS API activity auditing |

## Key Production Principles

1. WAF is not a replacement for secure application code.
2. CloudFront WAF does not automatically protect direct-origin requests.
3. Rate-based rules are approximate, not exact transactional quotas.
4. Broad Allow rules can bypass later protections.
5. Managed rules must be tested for false positives.
6. Request-body inspection has applicable limits.
7. Logging must be explicitly configured.
8. Application-side authentication and rate controls remain essential.
9. WAF at CloudFront and ALB operates at separate enforcement layers.
10. Every production rule change should have a validation and rollback plan.

---

# Official AWS References

- [AWS WAF Developer Guide](<https://docs.aws.amazon.com/waf/latest/developerguide/>)
- [AWS WAF Pricing](<https://aws.amazon.com/waf/pricing/>)
- [CloudFront Developer Guide](<https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/>)
- [AWS WAF API Reference](<https://docs.aws.amazon.com/waf/latest/APIReference/>)
- [Terraform AWS Provider — WAFv2 Web ACL](<https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/wafv2_web_acl>)
- [Terraform AWS Provider — WAFv2 Web ACL Association](<https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/wafv2_web_acl_association>)

---

## Final Summary

AWS WAF is an important component of production application security. A strong DevOps engineer should understand request evaluation, managed and custom rules, rate limiting, false-positive handling, origin protection, logging, Terraform automation, and incident response.

The most effective approach is layered security:

**CloudFront + AWS WAF + origin protection + application security + monitoring + tested recovery procedures.**

Learn the concepts, implement them in a non-production lab, test real request behavior, and practise explaining your design decisions in scenario-based interviews.

Save the document as `waf.md` in your notes or project repository.
