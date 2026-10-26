# AWS WAF: Complete DevOps Engineer Guide — 5 Years of Experience

For a DevOps engineer with around five years of experience, knowing AWS WAF means more than creating a Web ACL and adding managed rules. You should be able to design the security architecture, write and tune rules, prevent origin bypass, troubleshoot false positives, automate deployments, monitor attacks, control costs, and explain production incidents in interviews.

We'll approach this as a combination of production engineering and interview preparation.

## Learning Roadmap

*   **Level 1 — Fundamentals**: Web ACLs, rules, rule groups, scopes, actions, priorities, and request evaluation.
*   **Level 2 — Security engineering**: SQL injection, XSS, bot mitigation, rate limiting, IP reputation, and managed rules.
*   **Level 3 — Production architecture**: CloudFront, ALB, API Gateway, origin protection, multi-account setups, and incident handling.
*   **Level 4 — DevOps implementation**: Terraform, CI/CD, logging, CloudWatch, metrics, cost optimization, and safe rule deployments.
*   **Level 5 — Interview preparation**: Easy, medium, hard, and tricky scenario-based questions with model answers.

---

## Part 1 — Understand AWS WAF Deeply

### 1. What exactly is AWS WAF?
AWS WAF is a managed web application firewall that inspects HTTP/HTTPS requests and evaluates them against configured rules. Based on the rules, it can allow, block, count, challenge, or CAPTCHA requests where supported.

It helps protect web applications and APIs against threats such as:
*   SQL injection and cross-site scripting (XSS).
*   Automated abusive traffic and certain bot activities.
*   Excessive requests to sensitive endpoints.
*   Requests from prohibited IP addresses or address ranges.
*   Suspicious request patterns, headers, URI paths, and query strings.

AWS WAF does not replace application authentication, authorization, secure coding, network security, or DDoS protection. It is one layer in a defense-in-depth design.

### 2. Where does AWS WAF sit in the architecture?
A five-year DevOps engineer should understand which resource WAF protects and how traffic reaches it.

```text
Client / Browser / API consumer
  │
  ▼
Route 53 (DNS resolution — not the WAF inspection layer)
  │
  ▼
CloudFront + AWS WAF (Edge delivery and request filtering)
  │
  ▼
Application Load Balancer (Can have its own associated Web ACL)
  │
  ▼
Application servers / Containers (EC2, ECS, or EKS workloads)
```

#### CloudFront WAF vs ALB WAF

| Area | CloudFront-associated WAF | ALB-associated WAF |
| :--- | :--- | :--- |
| **Inspection location** | At the CloudFront edge | At the regional ALB |
| **Scope** | CloudFront / global scope | Regional scope |
| **Common use** | Public websites and global APIs | Regional application entry points |
| **Can block before origin?** | Yes, at the edge | Yes, before reaching ALB targets |
| **Important concern** | Direct origin access must be controlled | ALB must be reachable by trusted upstreams |

**Production consideration:** If you put WAF on CloudFront but leave the ALB publicly accessible, an attacker might bypass CloudFront and send requests directly to the ALB. To prevent that, restrict origin access with appropriate network controls and an origin-authentication design (e.g., CloudFront VPC origins or a carefully validated custom header combined with restrictions). A custom header alone is not sufficient if attackers can discover or reproduce it.

---

## Part 2 — Core AWS WAF Concepts You Must Master

### 3. Web ACL, Rule, Rule Group, and Statement
These terms are frequently mixed up in interviews.

| Component | Meaning | Example |
| :--- | :--- | :--- |
| **Web ACL** | The top-level policy associated with a protected resource | `production-web-acl` |
| **Rule** | A condition and action | Block a particular IP range |
| **Rule statement** | The actual matching logic | URI path equals `/admin` |
| **Rule group** | A reusable collection of rules | AWS Common Rule Set |
| **Rule priority** | Determines evaluation order | Priority 10 runs before 20 |
| **Default action** | What happens when no terminating rule decides | Allow or Block |
| **WCU** | Web ACL capacity unit used to measure inspection capacity | A rule group's capacity contribution |

#### Example of a Web ACL Configuration
```text
Priority | Rule                                 | Action
---------+--------------------------------------+-------------------------------------
0        | Allow trusted health-monitoring      | Allow
10       | Block known malicious IPs            | Block
20       | Rate-limit login requests            | Block when rate limit exceeded
30       | AWS Common Rule Set                  | Managed rule-group action
Default  | Allow                                | Allow
```

### 4. How Rule Priority and Terminating Actions Work
AWS WAF evaluates rules in ascending numerical priority: priority 0 is evaluated before priority 10. A terminating action stops further evaluation.

Consider these rules:
```text
Priority 0:  Allow requests from trusted IPs
Priority 10: Block requests containing malicious patterns
Priority 20: Count requests to /login
Default:     Block
```

**Security Issue:** If a trusted IP sends a request containing a SQL injection pattern, it matches priority 0, the `Allow` action terminates evaluation, and priority 10 is never checked. 

**Production Lesson:** Avoid broad early Allow rules unless bypassing all subsequent rules is intentional. Prefer narrowly scoped exceptions and rule-action overrides.

#### Understand the Actions
*   **Allow:** Permit the request to continue.
*   **Block:** Reject the request (default response: HTTP 403).
*   **Count:** Record a match and continue evaluation.
*   **CAPTCHA / Challenge:** Validate client sessions through tokens and interactive/silent challenges.

### 5. What are AWS Managed Rules?
AWS Managed Rules are predefined rule groups maintained by AWS (e.g., Common Rule Set, SQL Database Rule Set, Known Bad Inputs, Amazon IP Reputation List, Bot Control).

**The Biggest Misconception:** Enabling a managed rule group does not mean every matching request is automatically blocked. You can configure rule actions and overrides, such as running in `Count` mode initially to spot false positives.

### 6. Custom Rules and Scope-Down Statements
AWS WAF can inspect URI paths, query strings, methods, headers, source IPs, and request bodies. You can combine conditions using logical `AND`, `OR`, and `NOT`.

A **scope-down statement** narrows the requests evaluated by a managed rule group or rate-based rule. For example, rate-limiting only requests where the URI path equals `/login` avoids throttling static assets or product pages.

### 7. Rate-Based Rules in Detail
Rate-based rules track requests over a rolling 5-minute window and trigger actions when a threshold is exceeded.
*   They use an approximate tracking mechanism, not an exact transactional counter.
*   NAT gateways and corporate proxies can place many legitimate users behind a single IP address; account for this when setting thresholds.

### 8. Labels: An Advanced WAF Feature
Labels let one rule or managed rule group classify a request (e.g., adding a label for suspicious login patterns) so later rules can evaluate that label to enforce targeted rate-limits or blocks.

---

## Part 3 — Advanced Architecture and Operational Engineering

### 9. Web ACL Capacity Units (WCUs)
*   WCUs measure rule-processing capacity, **not** requests-per-second throughput.
*   The standard service model supports a maximum Web ACL capacity of 5,000 WCUs.

### 10. Logging, Metrics, and Troubleshooting
To troubleshoot an unexpected HTTP 403 error:
1. Confirm the 403 originates from WAF and not the application.
2. Search WAF logs using timestamps and request identifiers.
3. Identify the terminating rule ID, action, and matched labels.
4. Reproduce in a safe test environment and create a narrow rule adjustment or exclusion.

### 11. AWS WAF in Terraform and CI/CD
Here is a simplified Terraform snippet for a regional Web ACL with a managed rule group:

```hcl
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

### 12. Production Scenario: CloudFront + WAF + ALB
Protecting an e-commerce architecture requires edge protection via CloudFront WAF combined with strict origin validation to ensure attackers cannot bypass CloudFront and send payloads directly to a public ALB endpoint.

---

## Part 4 — Tricky Interview Scenarios

*   **Q1: A request matches both a Count rule and a Block rule. What happens?**  
    *Answer:* It depends on priority. If the Count rule runs first, evaluation continues and the subsequent Block rule can still block it. If the Block rule runs first, evaluation terminates immediately.
*   **Q2: Your rate-based rule limit is 1,000 requests, but you observe higher counts. Is WAF broken?**  
    *Answer:* No. Rate-based tracking is approximate over a sliding window and can fluctuate. It is not an exact transactional quota.
*   **Q3: Can an early Allow rule bypass an AWS Managed Rule group?**  
    *Answer:* Yes, because `Allow` is a terminating action for the Web ACL evaluation.

---

## Part 5 — DevOps Skills Self-Assessment

*   [ ] Explain Web ACL evaluation order and terminating actions
*   [ ] Build custom IP, URI, header, and rate-based rules
*   [ ] Deploy AWS Managed Rules safely and handle false positives
*   [ ] Explain CloudFront versus ALB WAF scope and origin bypass
*   [ ] Configure logging, metrics, dashboards, and alerts
*   [ ] Deploy and update WAF using Terraform and CI/CD
*   [ ] Troubleshoot production 403 responses using WAF logs
*   [ ] Explain WCU capacity, rate-based tracking, and cost trade-offs
*   [ ] Design a controlled rollout and rollback procedure
*   [ ] Explain how WAF complements application security and DDoS protection