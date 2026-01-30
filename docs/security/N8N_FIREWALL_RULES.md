# n8n Firewall Rules for Production

This document provides comprehensive firewall rules and security recommendations for protecting n8n deployments in production environments.

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Endpoint Categories](#endpoint-categories)
3. [Critical Security Rules](#critical-security-rules)
4. [Recommended Firewall Configuration](#recommended-firewall-configuration)
5. [WAF Rules](#waf-rules)
6. [Rate Limiting Recommendations](#rate-limiting-recommendations)
7. [IP Allowlisting Recommendations](#ip-allowlisting-recommendations)

---

## Executive Summary

Based on comprehensive scanning of the n8n codebase, the following endpoint statistics were identified:

| Category | Count | Risk Level |
|----------|-------|------------|
| REST API Endpoints | 282 | Mixed |
| Webhook Endpoints | 8 | High |
| System Endpoints | 3 | Low-Medium |
| Public API Endpoints | 27 | Medium |
| **Unauthenticated Endpoints** | **38** | **High** |
| Rate-Limited Endpoints | 25 | Protected |

### Risk Distribution

- **Critical Risk**: 5 endpoints
- **High Risk**: 36 endpoints
- **Medium Risk**: 241 endpoints
- **Low Risk**: 0 endpoints (all require some protection)

---

## Endpoint Categories

### 1. Critical Risk Endpoints (MUST PROTECT)

These endpoints are sensitive and must have strict firewall rules:

| Endpoint | Method | Auth | Description | Recommended Action |
|----------|--------|------|-------------|-------------------|
| `/rest/owner/setup` | POST | No | Initial owner setup | Block after first use or IP-restrict |
| `/rest/me/password` | PATCH | Yes | Password change | Rate limit, MFA required |
| `/rest/users/:id/password-reset-link` | GET | Yes | Password reset link generation | Rate limit, audit log |
| `/rest/sso/oidc/login` | GET | No | OIDC login initiation | Rate limit, monitor for abuse |
| `/rest/debug/multi-main-setup` | GET | No | Debug endpoint | **BLOCK IN PRODUCTION** |

### 2. High Risk Unauthenticated Endpoints

| Endpoint Pattern | Method | Description | Recommended Action |
|-----------------|--------|-------------|-------------------|
| `/rest/e2e/*` | ALL | End-to-end testing endpoints | **BLOCK IN PRODUCTION** |
| `/rest/login` | POST | User authentication | Rate limit: 5/min per email, 1000/min per IP |
| `/rest/invitations/accept` | POST | Invitation acceptance | Rate limit: 100/5min |
| `/rest/invitations/:id/accept` | POST | Invitation acceptance | Rate limit + keyed limit |
| `/rest/password-reset/*` | ALL | Password reset flow | Rate limit: 20/5min |
| `/rest/binary-data/signed` | GET | Signed binary data access | Rate limit, validate signatures |
| `/rest/resolve-signup-token` | GET | Signup token validation | Rate limit |

### 3. Webhook Endpoints (External Access Required)

| Endpoint Pattern | Method | Description | Recommended Action |
|-----------------|--------|-------------|-------------------|
| `/webhook/*` | ALL | Live production webhooks | Allow external, rate limit per IP |
| `/webhook-test/*` | ALL | Test webhooks | **BLOCK or IP-restrict in production** |
| `/webhook-waiting/*` | ALL | Waiting webhooks | Rate limit per IP |
| `/form/*` | ALL | Live forms | Allow external, CSRF protection |
| `/form-test/*` | ALL | Test forms | **BLOCK or IP-restrict in production** |
| `/form-waiting/*` | ALL | Waiting forms | Rate limit per IP |
| `/mcp/*` | ALL | MCP AI integration | Rate limit strictly |
| `/mcp-test/*` | ALL | Test MCP endpoints | **BLOCK in production** |

### 4. Telemetry & Analytics Endpoints

| Endpoint Pattern | Method | Auth | Description | Recommended Action |
|-----------------|--------|------|-------------|-------------------|
| `/rest/posthog/*` | POST | No | PostHog analytics proxy | Rate limit: 50-200/min |
| `/rest/telemetry/proxy/*` | POST | No | Telemetry proxy | Rate limit: 50-100/min |

### 5. SSO/Authentication Endpoints

| Endpoint Pattern | Method | Auth | Description | Recommended Action |
|-----------------|--------|------|-------------|-------------------|
| `/rest/sso/saml/metadata` | GET | No | SAML metadata | Allow public access |
| `/rest/sso/saml/acs` | GET/POST | No | SAML assertion consumer | Rate limit, validate signatures |
| `/rest/sso/saml/initsso` | GET | No | SAML SSO initiation | Rate limit |
| `/rest/sso/oidc/login` | GET | No | OIDC login | Rate limit |
| `/rest/sso/oidc/callback` | GET | No | OIDC callback | Rate limit |
| `/rest/ldap/*` | ALL | Yes | LDAP configuration | Admin-only access |

### 6. System/Health Endpoints

| Endpoint | Method | Auth | Description | Recommended Action |
|----------|--------|------|-------------|-------------------|
| `/healthz` | GET | No | Health check | Allow (required for LB) |
| `/healthz/readiness` | GET | No | Readiness check | Allow (required for K8s) |
| `/metrics` | GET | No | Prometheus metrics | **IP-restrict** to monitoring systems |

---

## Critical Security Rules

### MUST BLOCK in Production

```nginx
# Block E2E testing endpoints
location ~ ^/rest/e2e {
    deny all;
    return 403;
}

# Block debug endpoints
location ~ ^/rest/debug {
    deny all;
    return 403;
}

# Block test webhook endpoints (unless specifically needed)
location ~ ^/webhook-test {
    deny all;
    return 403;
}

location ~ ^/form-test {
    deny all;
    return 403;
}

location ~ ^/mcp-test {
    deny all;
    return 403;
}
```

### MUST IP-Restrict

```nginx
# Restrict metrics endpoint to monitoring IPs
location /metrics {
    allow 10.0.0.0/8;      # Internal network
    allow 172.16.0.0/12;   # Internal network
    allow 192.168.0.0/16;  # Internal network
    deny all;
}

# Restrict owner setup after initial configuration
location ~ ^/rest/owner/setup {
    # Only allow from admin IPs or block entirely after setup
    allow 192.168.1.100;  # Admin IP
    deny all;
}
```

---

## Recommended Firewall Configuration

### AWS WAF / CloudFront Rules

```yaml
# AWS WAF Rule Set for n8n
Rules:
  - Name: BlockE2EEndpoints
    Priority: 1
    Action: Block
    Statement:
      RegexPatternSetReferenceStatement:
        ARN: !Ref E2EBlockPattern
        FieldToMatch:
          UriPath: {}
    # Pattern: ^/rest/e2e.*

  - Name: BlockDebugEndpoints
    Priority: 2
    Action: Block
    Statement:
      ByteMatchStatement:
        FieldToMatch:
          UriPath: {}
        PositionalConstraint: STARTS_WITH
        SearchString: "/rest/debug"

  - Name: RateLimitLogin
    Priority: 10
    Action: Block
    Statement:
      RateBasedStatement:
        AggregateKeyType: IP
        Limit: 1000  # per 5 minutes
        ScopeDownStatement:
          ByteMatchStatement:
            FieldToMatch:
              UriPath: {}
            PositionalConstraint: EXACTLY
            SearchString: "/rest/login"

  - Name: RateLimitWebhooks
    Priority: 20
    Action: Block
    Statement:
      RateBasedStatement:
        AggregateKeyType: IP
        Limit: 10000  # per 5 minutes
        ScopeDownStatement:
          ByteMatchStatement:
            FieldToMatch:
              UriPath: {}
            PositionalConstraint: STARTS_WITH
            SearchString: "/webhook/"
```

### Nginx Configuration

```nginx
# n8n Security Configuration

# Rate limiting zones
limit_req_zone $binary_remote_addr zone=login:10m rate=10r/m;
limit_req_zone $binary_remote_addr zone=api:10m rate=100r/s;
limit_req_zone $binary_remote_addr zone=webhook:10m rate=50r/s;
limit_req_zone $binary_remote_addr zone=password:10m rate=5r/m;

server {
    listen 443 ssl http2;
    server_name n8n.example.com;

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    # NOTE: n8n requires 'unsafe-inline' and 'unsafe-eval' for its dynamic
    # workflow editor functionality. This is a known trade-off for the UI.
    add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline' 'unsafe-eval'; style-src 'self' 'unsafe-inline';" always;

    # Block dangerous endpoints
    location ~ ^/rest/e2e {
        return 403;
    }

    location ~ ^/rest/debug {
        return 403;
    }

    location ~ ^/(webhook-test|form-test|mcp-test) {
        return 403;
    }

    # Rate limit authentication endpoints
    location = /rest/login {
        limit_req zone=login burst=5 nodelay;
        proxy_pass http://n8n_backend;
    }

    # Rate limit password endpoints
    location ~ ^/rest/(password-reset|me/password) {
        limit_req zone=password burst=3 nodelay;
        proxy_pass http://n8n_backend;
    }

    # Rate limit webhooks
    location ~ ^/webhook {
        limit_req zone=webhook burst=100 nodelay;
        proxy_pass http://n8n_backend;
    }

    # Restrict metrics
    location /metrics {
        allow 10.0.0.0/8;
        deny all;
        proxy_pass http://n8n_backend;
    }

    # Default API rate limiting
    location /rest {
        limit_req zone=api burst=50 nodelay;
        proxy_pass http://n8n_backend;
    }

    # Public API with rate limiting
    location /api {
        limit_req zone=api burst=50 nodelay;
        proxy_pass http://n8n_backend;
    }

    location / {
        proxy_pass http://n8n_backend;
    }
}
```

### Cloudflare Rules

```yaml
# Cloudflare Firewall Rules for n8n

# Rule 1: Block E2E and Debug endpoints
- expression: |
    (http.request.uri.path matches "^/rest/e2e.*") or 
    (http.request.uri.path matches "^/rest/debug.*")
  action: block
  description: "Block testing and debug endpoints"

# Rule 2: Block test webhook endpoints
- expression: |
    (http.request.uri.path matches "^/(webhook-test|form-test|mcp-test)/.*")
  action: block
  description: "Block test webhook endpoints"

# Rule 3: Rate limit login
- expression: |
    (http.request.uri.path eq "/rest/login") and 
    (http.request.method eq "POST")
  action: challenge
  rateLimit:
    characteristics:
      - ip.src
    period: 60
    requestsPerPeriod: 10
  description: "Rate limit login attempts"

# Rule 4: Protect password reset
- expression: |
    (http.request.uri.path contains "/password")
  action: managed_challenge
  rateLimit:
    characteristics:
      - ip.src
    period: 300
    requestsPerPeriod: 20
  description: "Protect password endpoints"

# Rule 5: Geographic blocking (optional)
- expression: |
    (not ip.geoip.country in {"US" "GB" "DE" "FR" "CA"})
  action: managed_challenge
  description: "Challenge requests from unexpected countries"
```

---

## WAF Rules

### OWASP Core Rule Set Recommendations

Enable these OWASP ModSecurity rules for n8n:

| Rule ID | Description | Status |
|---------|-------------|--------|
| 911100 | Method is not allowed by policy | Enable |
| 913100 | Scanner detection | Enable |
| 920170 | GET/HEAD request with body content | Enable |
| 920350 | Host header is a numeric IP address | Enable (if using domain) |
| 930100-930130 | Local File Inclusion | Enable |
| 931100-931130 | Remote File Inclusion | Enable |
| 932100-932150 | Remote Command Execution | Enable |
| 933100-933200 | PHP Injection | Enable |
| 941100-941350 | XSS | Enable |
| 942100-942500 | SQL Injection | Enable |
| 943100 | Session Fixation | Enable |
| 944100-944150 | Java Code Injection | Enable |

### Custom WAF Rules for n8n

```yaml
# Protect workflow injection
- id: N8N-001
  description: "Block potential workflow injection"
  pattern: |
    (nodes\s*:\s*\[) and (type\s*:\s*["']n8n-nodes-base\.)
  target: BODY
  action: block
  conditions:
    - uri_path_not_starts_with: /rest/workflows

# Protect credential exfiltration  
- id: N8N-002
  description: "Block credential data in responses"
  pattern: |
    ("password"\s*:\s*"[^"]+") or ("apiKey"\s*:\s*"[^"]+")
  target: RESPONSE_BODY
  action: alert
  
# Detect webhook abuse
- id: N8N-003
  description: "Rate limit suspicious webhook patterns"
  conditions:
    - uri_path_starts_with: /webhook/
    - request_count_per_ip > 100 per 60s
  action: block
```

---

## Rate Limiting Recommendations

### By Endpoint Category

| Endpoint Category | Rate Limit | Window | Notes |
|------------------|------------|--------|-------|
| `/rest/login` | 1000 req/IP **AND** 5 req/email | 5 min | Both limits should be enforced; per-email limit prevents credential stuffing |
| `/rest/password-reset/*` | 20 req/IP | 5 min | Prevent enumeration |
| `/rest/invitations/*` | 100 req/IP | 5 min | Prevent abuse |
| `/webhook/*` | 10000 req/IP | 5 min | Allow legitimate automation |
| `/rest/ai/*` | 100 req/user | 1 min | Prevent AI credit abuse |
| `/api/v1/*` | 1000 req/API-key | 1 min | Public API |
| `/rest/posthog/*` | 200 req/IP | 1 min | Analytics |
| `/rest/*` (default) | 500 req/IP | 1 min | General API |

> **Note**: n8n has built-in rate limiting for many endpoints. The table above shows
> recommended firewall-level limits. When multiple layers (WAF + Nginx + n8n built-in)
> are used, the most restrictive limit applies.

### Implementation Priority

1. **CRITICAL**: Login, password reset, owner setup
2. **HIGH**: Webhooks, invitations, SSO endpoints
3. **MEDIUM**: API endpoints, AI features
4. **LOW**: Static assets, health checks

---

## IP Allowlisting Recommendations

### Internal-Only Endpoints

These endpoints should only be accessible from internal networks:

```
/metrics              - Monitoring systems only
/rest/debug/*         - Block or admin IPs only
/rest/e2e/*           - Block or CI/CD systems only
/rest/owner/setup     - Admin IPs only (after initial setup)
/rest/orchestration/* - Internal services only
```

### Recommended IP Ranges

```nginx
# Internal monitoring
10.0.0.0/8
172.16.0.0/12
192.168.0.0/16

# Cloud provider metadata endpoints
# WARNING: Blocking these prevents SSRF attacks but may break workflows
# that legitimately need instance metadata (e.g., for AWS credentials).
# Only block if your workflows don't require metadata access:
# 169.254.169.254/32  # AWS/GCP/Azure metadata - CONSIDER BLOCKING
# 100.100.100.200/32  # Alibaba metadata - CONSIDER BLOCKING
```

---

## Security Checklist

### Pre-Production

- [ ] Block all `/rest/e2e/*` endpoints
- [ ] Block all `/rest/debug/*` endpoints
- [ ] Block all `*-test` webhook endpoints
- [ ] Configure rate limiting on login endpoint
- [ ] Configure rate limiting on password reset
- [ ] IP-restrict `/metrics` endpoint
- [ ] Enable WAF with OWASP rules
- [ ] Configure HTTPS with TLS 1.2+
- [ ] Set security headers (CSP, HSTS, X-Frame-Options)
- [ ] Configure geographic blocking if applicable

### Ongoing Monitoring

- [ ] Monitor failed login attempts
- [ ] Alert on brute force patterns
- [ ] Review webhook abuse patterns
- [ ] Audit API key usage
- [ ] Review rate limit violations
- [ ] Check for new endpoints on updates

---

## Environment Variables for Security

```bash
# Disable test endpoints in production
N8N_DISABLE_UI=false
N8N_DISABLE_PRODUCTION_MAIN_PROCESS=false

# Rate limiting (built-in)
# Note: n8n has built-in rate limiting for many endpoints

# Proxy configuration
N8N_PROXY_HOPS=1  # Set based on your proxy setup

# Metrics (restrict access separately)
N8N_METRICS=true
N8N_METRICS_PREFIX=n8n_
```

---

## Related Files

- [endpoints-inventory.json](./endpoints-inventory.json) - Complete endpoint inventory
- [endpoints-by-risk.json](./endpoints-by-risk.json) - Endpoints categorized by risk level
- [firewall-rules-nginx.conf](./firewall-rules-nginx.conf) - Nginx configuration template
- [firewall-rules-aws-waf.json](./firewall-rules-aws-waf.json) - AWS WAF rule configuration

---

*Generated on: 2026-01-30*
*n8n Version: Based on current source code analysis*
