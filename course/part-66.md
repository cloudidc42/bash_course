# Part 66: Global CDN Architecture, Edge Computing 2.0 และ Multi-Region Deployment

## ภาพรวม
Part นี้ครอบคลุม global-scale infrastructure: CDN architecture, edge computing patterns และ multi-region deployment ระดับ World-Class

---

## Step 612: Global CDN Architecture — CloudFront + Edge Logic

```bash
cat > global-cdn.sh << 'SCRIPT'
#!/bin/bash
# Global CDN Architecture with CloudFront + Edge Functions

set -euo pipefail

echo "=== Global CDN Architecture ==="

# ─── 1. CloudFront Distribution with Advanced Configuration ───────────────
cat > cloudfront-distribution.tf << 'EOF'
# Terraform: CloudFront with advanced caching + edge security

resource "aws_cloudfront_distribution" "main" {
  comment         = "Global CDN for payment platform"
  enabled         = true
  is_ipv6_enabled = true
  price_class     = "PriceClass_All"
  http_version    = "http2and3"

  # S3 origin for static assets
  origin {
    domain_name              = aws_s3_bucket.assets.bucket_regional_domain_name
    origin_id                = "S3-assets"
    origin_access_control_id = aws_cloudfront_origin_access_control.s3_oac.id
    
    origin_shield {
      enabled              = true
      origin_shield_region = "ap-southeast-1"  # Closest to origin
    }
  }

  # ALB origin for API
  origin {
    domain_name = aws_lb.api.dns_name
    origin_id   = "ALB-api"
    
    custom_origin_config {
      http_port              = 80
      https_port             = 443
      origin_protocol_policy = "https-only"
      origin_ssl_protocols   = ["TLSv1.2"]
      origin_read_timeout    = 60
      origin_keepalive_timeout = 60
    }
    
    origin_shield {
      enabled              = true
      origin_shield_region = "ap-southeast-1"
    }
    
    custom_header {
      name  = "X-Origin-Verify"
      value = var.origin_verify_header
    }
  }

  # Default: static assets (aggressive caching)
  default_cache_behavior {
    target_origin_id       = "S3-assets"
    viewer_protocol_policy = "redirect-to-https"
    allowed_methods        = ["GET", "HEAD", "OPTIONS"]
    cached_methods         = ["GET", "HEAD"]
    compress               = true
    
    forwarded_values {
      query_string = false
      headers      = ["Origin", "Access-Control-Request-Headers", "Access-Control-Request-Method"]
      cookies {
        forward = "none"
      }
    }
    
    # CloudFront Functions for static assets
    function_association {
      event_type   = "viewer-request"
      function_arn = aws_cloudfront_function.security_headers.arn
    }
    
    min_ttl     = 0
    default_ttl = 86400   # 1 day
    max_ttl     = 31536000  # 1 year
  }

  # API behavior (dynamic, short cache)
  ordered_cache_behavior {
    path_pattern           = "/api/*"
    target_origin_id       = "ALB-api"
    viewer_protocol_policy = "https-only"
    allowed_methods        = ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"]
    cached_methods         = ["GET", "HEAD"]
    compress               = true
    
    cache_policy_id          = aws_cloudfront_cache_policy.api.id
    origin_request_policy_id = aws_cloudfront_origin_request_policy.api.id
    response_headers_policy_id = aws_cloudfront_response_headers_policy.security.id
    
    # Lambda@Edge for auth + routing
    lambda_function_association {
      event_type   = "viewer-request"
      lambda_arn   = aws_lambda_function.edge_auth.qualified_arn
      include_body = true
    }
    
    lambda_function_association {
      event_type = "origin-response"
      lambda_arn = aws_lambda_function.edge_cache_vary.qualified_arn
    }
    
    min_ttl     = 0
    default_ttl = 0       # No caching for API by default
    max_ttl     = 300     # Max 5 min if cache-control allows
  }

  # WAF
  web_acl_id = aws_wafv2_web_acl.cloudfront.arn

  # Geo restrictions (OFAC compliance)
  restrictions {
    geo_restriction {
      restriction_type = "blacklist"
      locations        = ["KP", "IR", "SY", "CU"]  # OFAC sanctioned
    }
  }

  viewer_certificate {
    acm_certificate_arn      = aws_acm_certificate.main.arn
    ssl_support_method       = "sni-only"
    minimum_protocol_version = "TLSv1.2_2021"
  }

  # Real-time logs to Kinesis
  logging_config {
    include_cookies = false
    bucket          = aws_s3_bucket.logs.bucket_domain_name
    prefix          = "cloudfront/"
  }
}

# Cache policy for API endpoints
resource "aws_cloudfront_cache_policy" "api" {
  name        = "api-cache-policy"
  min_ttl     = 0
  default_ttl = 0
  max_ttl     = 300

  parameters_in_cache_key_and_forwarded_to_origin {
    headers_config {
      header_behavior = "whitelist"
      headers {
        items = ["Authorization", "Accept-Language", "Accept-Encoding"]
      }
    }
    query_strings_config {
      query_string_behavior = "all"
    }
    cookies_config {
      cookie_behavior = "none"
    }
  }
}
EOF

# ─── 2. CloudFront Functions (lightweight edge logic) ─────────────────────
cat > cf-security-headers.js << 'EOF'
// CloudFront Function: Add security headers + normalize URL
function handler(event) {
    var request = event.request;
    var headers = request.headers;
    
    // Normalize URL: remove trailing slashes
    var uri = request.uri;
    if (uri.endsWith('/') && uri !== '/') {
        request.uri = uri.slice(0, -1);
    }
    
    // Handle www → apex redirect
    var host = headers['host']?.value || '';
    if (host.startsWith('www.')) {
        return {
            statusCode: 301,
            statusDescription: 'Moved Permanently',
            headers: {
                location: { value: 'https://' + host.substring(4) + uri }
            }
        };
    }
    
    // Add cache-bust for HTML files (versioned assets don't need this)
    if (uri.endsWith('.html') || uri === '/') {
        request.headers['x-html-request'] = { value: 'true' };
    }
    
    return request;
}
EOF

cat > cf-response-headers.js << 'EOF'
// CloudFront Function: Add security response headers
function handler(event) {
    var response = event.response;
    var headers = response.headers;
    
    // Security headers
    headers['strict-transport-security'] = {
        value: 'max-age=31536000; includeSubDomains; preload'
    };
    headers['x-content-type-options'] = { value: 'nosniff' };
    headers['x-frame-options'] = { value: 'DENY' };
    headers['x-xss-protection'] = { value: '1; mode=block' };
    headers['referrer-policy'] = {
        value: 'strict-origin-when-cross-origin'
    };
    headers['permissions-policy'] = {
        value: 'geolocation=(), microphone=(), camera=()'
    };
    headers['content-security-policy'] = {
        value: [
            "default-src 'self'",
            "script-src 'self' 'nonce-{NONCE}'",
            "style-src 'self' 'unsafe-inline'",
            "img-src 'self' data: https://cdn.example.com",
            "connect-src 'self' https://api.example.com",
            "frame-ancestors 'none'",
            "base-uri 'self'",
            "form-action 'self'"
        ].join('; ')
    };
    
    // Remove server info
    delete headers['server'];
    delete headers['x-powered-by'];
    
    return response;
}
EOF

# ─── 3. Lambda@Edge — JWT Auth + Geo-Routing ──────────────────────────────
cat > lambda-edge-auth.js << 'EOF'
'use strict';

const jwt = require('jsonwebtoken');
const https = require('https');

// JWT public keys cache (refreshed every hour)
let jwksCache = null;
let jwksCacheTime = 0;

async function getJWKS() {
    const now = Date.now();
    if (jwksCache && now - jwksCacheTime < 3600000) {
        return jwksCache;
    }
    
    return new Promise((resolve, reject) => {
        https.get('https://auth.example.com/.well-known/jwks.json', (res) => {
            let data = '';
            res.on('data', (chunk) => data += chunk);
            res.on('end', () => {
                jwksCache = JSON.parse(data);
                jwksCacheTime = Date.now();
                resolve(jwksCache);
            });
        }).on('error', reject);
    });
}

exports.handler = async (event) => {
    const request = event.Records[0].cf.request;
    const headers = request.headers;
    
    // Skip auth for public paths
    const publicPaths = ['/api/v2/health', '/api/v2/auth/login', '/api/v2/auth/register'];
    if (publicPaths.some(p => request.uri.startsWith(p))) {
        return request;
    }
    
    // Extract JWT from Authorization header or cookie
    let token = null;
    const authHeader = headers['authorization']?.[0]?.value;
    if (authHeader?.startsWith('Bearer ')) {
        token = authHeader.substring(7);
    }
    
    if (!token) {
        return {
            status: '401',
            statusDescription: 'Unauthorized',
            headers: {
                'www-authenticate': [{ value: 'Bearer realm="api"' }],
                'content-type': [{ value: 'application/json' }],
            },
            body: JSON.stringify({ error: 'Authentication required' }),
        };
    }
    
    try {
        const jwks = await getJWKS();
        const decoded = jwt.verify(token, jwks, {
            algorithms: ['RS256'],
            issuer: 'https://auth.example.com',
            audience: 'api.example.com',
        });
        
        // Add user context to request headers
        request.headers['x-user-id'] = [{ value: decoded.sub }];
        request.headers['x-user-tier'] = [{ value: decoded.tier || 'free' }];
        request.headers['x-user-region'] = [{ value: decoded.region || 'global' }];
        
        // Geo-based routing: Thailand users → SEA cluster
        const viewerCountry = headers['cloudfront-viewer-country']?.[0]?.value;
        if (['TH', 'SG', 'MY', 'ID', 'VN'].includes(viewerCountry)) {
            request.origin = {
                custom: {
                    domainName: 'api-sea.example.com',
                    port: 443,
                    protocol: 'https',
                    sslProtocols: ['TLSv1.2'],
                    readTimeout: 30,
                    keepaliveTimeout: 5,
                    customHeaders: {}
                }
            };
        }
        
        return request;
        
    } catch (err) {
        return {
            status: '401',
            statusDescription: 'Unauthorized',
            headers: {
                'content-type': [{ value: 'application/json' }],
            },
            body: JSON.stringify({ error: 'Invalid token' }),
        };
    }
};
EOF

echo "CloudFront configuration, CF Functions, and Lambda@Edge created"
echo "=== Step 612 Complete: Global CDN Architecture ==="
SCRIPT
chmod +x global-cdn.sh
echo "Script created: global-cdn.sh"
```

**สิ่งที่เรียนรู้:**
- CloudFront distribution: Origin Shield (reduce origin load), http2and3, geo restriction (OFAC compliance)
- CloudFront Functions (lightweight, <1ms): URL normalization, security response headers (HSTS, CSP, X-Frame-Options)
- Lambda@Edge (full Node.js): JWKS-based JWT validation with 1-hour cache, geo-based origin routing (SEA cluster)
- Cache policy: whitelist Authorization/Accept-Language headers, max 5-min TTL for API
- WAF integration for DDoS protection

---

## Step 613: Multi-Region Active-Active Deployment

```bash
cat > multi-region-deployment.sh << 'SCRIPT'
#!/bin/bash
# Multi-Region Active-Active with Global Load Balancing

set -euo pipefail

echo "=== Multi-Region Active-Active Deployment ==="

# ─── 1. Route 53 Global Load Balancing ────────────────────────────────────
cat > route53-global-routing.tf << 'EOF'
# Route53 with latency-based routing + health checks

# Health checks for each region
resource "aws_route53_health_check" "ap_southeast_1" {
  fqdn              = "api-sea.example.com"
  port              = 443
  type              = "HTTPS"
  resource_path     = "/health"
  failure_threshold = 2
  request_interval  = 10    # seconds
  
  regions = ["ap-southeast-1", "ap-northeast-1", "eu-west-1"]  # Check from 3 regions
  
  cloudwatch_alarm_name   = "api-sea-health"
  cloudwatch_alarm_region = "us-east-1"
  insufficient_data_health_status = "Unhealthy"
  
  tags = { Name = "api-sea-health" }
}

resource "aws_route53_health_check" "us_east_1" {
  fqdn              = "api-us.example.com"
  port              = 443
  type              = "HTTPS"
  resource_path     = "/health"
  failure_threshold = 2
  request_interval  = 10
  
  regions = ["us-east-1", "us-west-2", "eu-west-1"]
}

# Primary record: latency-based routing SEA
resource "aws_route53_record" "api_sea" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "api.example.com"
  type    = "A"
  
  set_identifier = "sea"
  
  latency_routing_policy {
    region = "ap-southeast-1"
  }
  
  alias {
    name                   = aws_lb.api_sea.dns_name
    zone_id                = aws_lb.api_sea.zone_id
    evaluate_target_health = true
  }
  
  health_check_id = aws_route53_health_check.ap_southeast_1.id
}

# Primary record: latency-based routing US
resource "aws_route53_record" "api_us" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "api.example.com"
  type    = "A"
  
  set_identifier = "us"
  
  latency_routing_policy {
    region = "us-east-1"
  }
  
  alias {
    name                   = aws_lb.api_us.dns_name
    zone_id                = aws_lb.api_us.zone_id
    evaluate_target_health = true
  }
  
  health_check_id = aws_route53_health_check.us_east_1.id
}

# Global Accelerator (anycast IPs, AWS backbone)
resource "aws_globalaccelerator_accelerator" "main" {
  name            = "payment-platform"
  ip_address_type = "DUAL_STACK"
  enabled         = true
  
  attributes {
    flow_logs_enabled   = true
    flow_logs_s3_bucket = aws_s3_bucket.ga_logs.id
    flow_logs_s3_prefix = "global-accelerator/"
  }
}

resource "aws_globalaccelerator_listener" "https" {
  accelerator_arn = aws_globalaccelerator_accelerator.main.id
  client_affinity = "SOURCE_IP"
  protocol        = "TCP"
  
  port_range {
    from_port = 443
    to_port   = 443
  }
}

resource "aws_globalaccelerator_endpoint_group" "sea" {
  listener_arn                  = aws_globalaccelerator_listener.https.id
  endpoint_group_region         = "ap-southeast-1"
  traffic_dial_percentage       = 100
  health_check_path             = "/health"
  health_check_interval_seconds = 10
  threshold_count               = 2
  
  endpoint_configuration {
    endpoint_id                    = aws_lb.api_sea.arn
    weight                         = 100
    client_ip_preservation_enabled = true
  }
}
EOF

# ─── 2. Database Multi-Region Replication ──────────────────────────────────
cat > multi-region-db.tf << 'EOF'
# Aurora Global Database (primary: ap-southeast-1, secondary: us-east-1, eu-west-1)
resource "aws_rds_global_cluster" "main" {
  global_cluster_identifier = "payment-global-db"
  engine                    = "aurora-postgresql"
  engine_version            = "16.4"
  database_name             = "payments"
  storage_encrypted         = true
}

resource "aws_rds_cluster" "primary" {
  cluster_identifier        = "payment-primary-sea"
  engine                    = "aurora-postgresql"
  engine_version            = "16.4"
  global_cluster_identifier = aws_rds_global_cluster.main.id
  
  availability_zones = ["ap-southeast-1a", "ap-southeast-1b", "ap-southeast-1c"]
  
  db_subnet_group_name   = aws_db_subnet_group.sea.name
  vpc_security_group_ids = [aws_security_group.aurora.id]
  
  # Backtrack: point-in-time recovery without restore
  backtrack_window = 72  # 72 hours
  
  backup_retention_period = 30
  preferred_backup_window = "03:00-04:00"
  
  # Enhanced monitoring
  enable_cloudwatch_logs_exports = ["postgresql"]
  monitoring_interval            = 1  # Enhanced monitoring every 1 second
  
  serverlessv2_scaling_configuration {
    max_capacity = 256   # ACU (max ~512GB RAM)
    min_capacity = 0.5   # Scale to near-zero
  }
  
  tags = { Region = "primary", Tier = "critical" }
}

resource "aws_rds_cluster" "secondary_us" {
  cluster_identifier        = "payment-secondary-us"
  engine                    = "aurora-postgresql"
  engine_version            = "16.4"
  global_cluster_identifier = aws_rds_global_cluster.main.id
  
  availability_zones = ["us-east-1a", "us-east-1b", "us-east-1c"]
  
  # Read replica region
  is_primary_cluster = false
  
  serverlessv2_scaling_configuration {
    max_capacity = 64
    min_capacity = 0.5
  }
  
  depends_on = [aws_rds_cluster.primary]
}
EOF

# ─── 3. Multi-Region Failover Automation ───────────────────────────────────
cat > failover_automation.py << 'PYEOF'
#!/usr/bin/env python3
"""
Multi-Region Failover Automation.
Detects regional failures and coordinates global failover.
"""

import asyncio
import logging
import json
import time
from dataclasses import dataclass, field
from typing import Dict, List, Optional
from enum import Enum

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class RegionStatus(Enum):
    HEALTHY = "healthy"
    DEGRADED = "degraded"
    FAILED = "failed"

@dataclass
class RegionHealth:
    region: str
    status: RegionStatus
    api_latency_ms: float
    api_success_rate: float
    db_replication_lag_ms: float
    last_checked: float = field(default_factory=time.time)

class MultiRegionFailoverController:
    REGIONS = {
        "ap-southeast-1": {
            "api": "https://api-sea.example.com",
            "priority": 1,  # Primary
        },
        "us-east-1": {
            "api": "https://api-us.example.com",
            "priority": 2,
        },
        "eu-west-1": {
            "api": "https://api-eu.example.com",
            "priority": 3,
        }
    }
    
    FAILURE_THRESHOLD = {
        "api_success_rate": 0.95,   # Below 95% = failure
        "api_latency_ms": 5000,     # Above 5s = failure
        "consecutive_failures": 3,
    }

    def __init__(self):
        self.health: Dict[str, RegionHealth] = {}
        self.failure_counts: Dict[str, int] = {}
        self.active_primary: str = "ap-southeast-1"

    async def check_region_health(self, region: str) -> RegionHealth:
        """Check health of a specific region."""
        import random
        
        # Simulate health check (real: query CloudWatch + Prometheus)
        success_rate = random.uniform(0.98, 1.0)
        latency = random.uniform(50, 200)
        
        status = RegionStatus.HEALTHY
        if success_rate < 0.95 or latency > 5000:
            status = RegionStatus.FAILED
        elif success_rate < 0.99 or latency > 2000:
            status = RegionStatus.DEGRADED
        
        return RegionHealth(
            region=region,
            status=status,
            api_latency_ms=latency,
            api_success_rate=success_rate,
            db_replication_lag_ms=random.uniform(10, 200),
        )

    async def monitor_regions(self):
        """Continuous regional health monitoring."""
        logger.info("Starting multi-region health monitor")
        
        while True:
            checks = await asyncio.gather(*[
                self.check_region_health(r)
                for r in self.REGIONS
            ])
            
            for health in checks:
                self.health[health.region] = health
                
                if health.status == RegionStatus.FAILED:
                    self.failure_counts[health.region] = \
                        self.failure_counts.get(health.region, 0) + 1
                    
                    if (self.failure_counts[health.region] >= 
                            self.FAILURE_THRESHOLD["consecutive_failures"]):
                        await self._handle_regional_failure(health.region)
                else:
                    self.failure_counts[health.region] = 0
                    
                    # Recovery: restore primary if it recovers
                    if (health.region == "ap-southeast-1" and 
                            self.active_primary != "ap-southeast-1" and
                            health.status == RegionStatus.HEALTHY):
                        await self._restore_primary(health.region)
            
            await asyncio.sleep(10)  # Check every 10 seconds

    async def _handle_regional_failure(self, failed_region: str):
        """Execute failover procedure when a region fails."""
        if failed_region != self.active_primary:
            logger.warning(f"Secondary region {failed_region} failed - monitoring only")
            return
        
        logger.critical(f"PRIMARY REGION FAILURE: {failed_region}")
        
        # Find best healthy secondary
        candidates = [
            (region, config["priority"])
            for region, config in self.REGIONS.items()
            if region != failed_region
            and self.health.get(region, RegionHealth(
                region, RegionStatus.FAILED, 0, 0, 0
            )).status == RegionStatus.HEALTHY
        ]
        
        if not candidates:
            logger.critical("NO HEALTHY REGIONS AVAILABLE - FULL OUTAGE")
            await self._trigger_pagerduty_critical("FULL_OUTAGE")
            return
        
        # Select lowest priority number (highest priority)
        next_primary = min(candidates, key=lambda x: x[1])[0]
        
        logger.info(f"Failing over to {next_primary}")
        
        # Execute failover steps
        await self._promote_aurora_secondary(next_primary)
        await self._update_route53(next_primary)
        await self._update_global_accelerator(next_primary)
        await self._notify_teams(failed_region, next_primary)
        
        self.active_primary = next_primary
        logger.info(f"Failover complete: primary is now {next_primary}")

    async def _promote_aurora_secondary(self, region: str):
        """Promote Aurora secondary to primary."""
        logger.info(f"Promoting Aurora secondary in {region}...")
        # boto3: rds.promote_read_replica_db_cluster(...)
        await asyncio.sleep(0.1)  # ~30-60 seconds in production
        logger.info(f"Aurora promoted in {region}")

    async def _update_route53(self, new_primary: str):
        """Update Route53 weights to route traffic to new primary."""
        logger.info(f"Updating Route53 to route traffic to {new_primary}")
        # boto3: route53.change_resource_record_sets(...)
        await asyncio.sleep(0.05)

    async def _update_global_accelerator(self, new_primary: str):
        """Update Global Accelerator endpoint weights."""
        logger.info(f"Updating Global Accelerator endpoint weights")
        await asyncio.sleep(0.05)

    async def _restore_primary(self, region: str):
        """Restore original primary when it recovers."""
        logger.info(f"Primary region {region} recovered - restoring as primary")
        self.active_primary = region

    async def _trigger_pagerduty_critical(self, event: str):
        logger.critical(f"PagerDuty P0 triggered: {event}")

    async def _notify_teams(self, failed: str, new_primary: str):
        logger.info(f"Notified: Slack + PagerDuty - failover {failed} → {new_primary}")

async def demo():
    controller = MultiRegionFailoverController()
    
    print("=== Multi-Region Failover Demo ===\n")
    
    # Simulate one round of health checks
    checks = await asyncio.gather(*[
        controller.check_region_health(r)
        for r in controller.REGIONS
    ])
    
    for health in checks:
        controller.health[health.region] = health
        icon = {"healthy": "✓", "degraded": "⚠", "failed": "✗"}[health.status.value]
        print(f"  {icon} {health.region}: {health.status.value} "
              f"(latency={health.api_latency_ms:.0f}ms, "
              f"success={health.api_success_rate:.2%})")
    
    print(f"\nActive primary: {controller.active_primary}")

asyncio.run(demo())
PYEOF

python3 failover_automation.py

echo "=== Step 613 Complete: Multi-Region Active-Active ==="
SCRIPT
chmod +x multi-region-deployment.sh
bash multi-region-deployment.sh 2>/dev/null || true
echo "Script created: multi-region-deployment.sh"
```

**สิ่งที่เรียนรู้:**
- Route53 latency-based routing: health check จาก 3 regions, auto-failover เมื่อ health check fail
- AWS Global Accelerator: anycast IPs บน AWS backbone, SOURCE_IP client affinity
- Aurora Global Database: primary (SEA) + secondaries (US/EU), Serverless v2 (0.5-256 ACU)
- Aurora Backtrack: point-in-time recovery ใน 72 ชั่วโมง (ไม่ต้อง restore snapshot)
- Failover automation: 3-strike failure detection, automated Aurora promotion + Route53 + GA update

---

## Step 614: Edge Computing 2.0 — Cloudflare Workers + Durable Objects

```bash
cat > edge-computing-2.sh << 'SCRIPT'
#!/bin/bash
# Edge Computing 2.0: Cloudflare Workers + Durable Objects + KV

set -euo pipefail

echo "=== Edge Computing 2.0: Cloudflare Workers ==="

# ─── 1. Cloudflare Workers: API Gateway at Edge ────────────────────────────
cat > worker-api-gateway.js << 'EOF'
/**
 * Cloudflare Worker: Edge API Gateway
 * Features: JWT auth, rate limiting, geo-routing, request transformation
 */

import { verify } from '@tsndr/cloudflare-worker-jwt';

const RATE_LIMIT_WINDOW = 60;     // seconds
const RATE_LIMIT_REQUESTS = 1000; // per window

// Regional origin mapping
const REGION_ORIGINS = {
    'TH': 'https://api-sea.example.com',
    'SG': 'https://api-sea.example.com',
    'MY': 'https://api-sea.example.com',
    'US': 'https://api-us.example.com',
    'GB': 'https://api-eu.example.com',
    'DE': 'https://api-eu.example.com',
    'DEFAULT': 'https://api-us.example.com',
};

export default {
    async fetch(request, env, ctx) {
        const url = new URL(request.url);
        const country = request.cf?.country || 'DEFAULT';
        
        // 1. Skip auth for public endpoints
        const isPublic = ['/health', '/api/v2/auth/login'].some(
            path => url.pathname.startsWith(path)
        );
        
        // 2. JWT Authentication
        if (!isPublic) {
            const authResult = await authenticateRequest(request, env);
            if (!authResult.success) {
                return new Response(
                    JSON.stringify({ error: authResult.error }),
                    { status: 401, headers: { 'Content-Type': 'application/json' } }
                );
            }
            request = authResult.enrichedRequest;
        }
        
        // 3. Rate limiting (per IP using Durable Objects)
        const clientIP = request.headers.get('CF-Connecting-IP') || '0.0.0.0';
        const rateLimitResult = await checkRateLimit(env, clientIP);
        if (!rateLimitResult.allowed) {
            return new Response(
                JSON.stringify({ error: 'Rate limit exceeded' }),
                {
                    status: 429,
                    headers: {
                        'Content-Type': 'application/json',
                        'Retry-After': String(rateLimitResult.retryAfter),
                        'X-RateLimit-Limit': String(RATE_LIMIT_REQUESTS),
                        'X-RateLimit-Remaining': '0',
                    }
                }
            );
        }
        
        // 4. Geo-based routing
        const origin = REGION_ORIGINS[country] || REGION_ORIGINS['DEFAULT'];
        
        // 5. Build proxied request
        const originUrl = new URL(url.pathname + url.search, origin);
        const proxyRequest = new Request(originUrl, {
            method: request.method,
            headers: {
                ...Object.fromEntries(request.headers),
                'X-Forwarded-For': clientIP,
                'X-CF-Country': country,
                'X-CF-Ray': request.headers.get('CF-Ray') || '',
                // Remove external headers, add internal
                'X-Origin-Verify': env.ORIGIN_VERIFY_TOKEN,
            },
            body: request.body,
        });
        
        // 6. Forward to origin with timeout
        const response = await Promise.race([
            fetch(proxyRequest),
            new Promise((_, reject) =>
                setTimeout(() => reject(new Error('Origin timeout')), 30000)
            )
        ]).catch(() => new Response('Gateway Timeout', { status: 504 }));
        
        // 7. Add edge headers to response
        const modifiedResponse = new Response(response.body, response);
        modifiedResponse.headers.set('X-Edge-Region', country);
        modifiedResponse.headers.set('X-Cache-Status',
            response.headers.get('CF-Cache-Status') || 'MISS'
        );
        
        return modifiedResponse;
    }
};

async function authenticateRequest(request, env) {
    const authHeader = request.headers.get('Authorization') || '';
    
    if (!authHeader.startsWith('Bearer ')) {
        return { success: false, error: 'Missing Bearer token' };
    }
    
    const token = authHeader.substring(7);
    
    try {
        const isValid = await verify(token, env.JWT_SECRET, { algorithm: 'HS256' });
        if (!isValid) {
            return { success: false, error: 'Invalid token' };
        }
        
        // Decode payload (without verification since already verified)
        const [, payload] = token.split('.');
        const decoded = JSON.parse(atob(payload));
        
        // Add user context to request
        const headers = new Headers(request.headers);
        headers.set('X-User-ID', decoded.sub || '');
        headers.set('X-User-Tier', decoded.tier || 'free');
        
        return {
            success: true,
            enrichedRequest: new Request(request, { headers })
        };
    } catch {
        return { success: false, error: 'Token verification failed' };
    }
}

async function checkRateLimit(env, clientIP) {
    // Use Durable Object for distributed rate limiting
    const id = env.RATE_LIMITER.idFromName(clientIP);
    const rateLimiter = env.RATE_LIMITER.get(id);
    
    const result = await rateLimiter.fetch('https://internal/check', {
        method: 'POST',
        body: JSON.stringify({
            window: RATE_LIMIT_WINDOW,
            limit: RATE_LIMIT_REQUESTS,
        }),
    });
    
    return result.json();
}
EOF

# ─── 2. Durable Object: Rate Limiter ──────────────────────────────────────
cat > durable-object-rate-limiter.js << 'EOF'
/**
 * Cloudflare Durable Object: Distributed Rate Limiter
 * State persisted in Durable Object storage (strong consistency per key)
 */

export class RateLimiter {
    constructor(state, env) {
        this.state = state;
    }

    async fetch(request) {
        const { window, limit } = await request.json();
        
        const now = Math.floor(Date.now() / 1000);
        const windowStart = Math.floor(now / window) * window;
        const key = `count:${windowStart}`;
        
        // Atomic increment using Durable Object storage
        const current = (await this.state.storage.get(key)) || 0;
        
        if (current >= limit) {
            const retryAfter = windowStart + window - now;
            return new Response(JSON.stringify({
                allowed: false,
                retryAfter,
                current,
                limit,
            }), { headers: { 'Content-Type': 'application/json' } });
        }
        
        // Increment counter
        await this.state.storage.put(key, current + 1);
        
        // Auto-expire old windows
        this.state.storage.delete(`count:${windowStart - window}`);
        
        return new Response(JSON.stringify({
            allowed: true,
            current: current + 1,
            remaining: limit - current - 1,
            limit,
        }), { headers: { 'Content-Type': 'application/json' } });
    }
}
EOF

# ─── 3. Wrangler Configuration ─────────────────────────────────────────────
cat > wrangler.toml << 'EOF'
name = "api-gateway-edge"
main = "worker-api-gateway.js"
compatibility_date = "2025-09-01"
compatibility_flags = ["nodejs_compat"]

[vars]
ENVIRONMENT = "production"

# Durable Objects
[[durable_objects.bindings]]
name = "RATE_LIMITER"
class_name = "RateLimiter"
script_name = "api-gateway-edge"  # Same worker

[[migrations]]
tag = "v1"
new_classes = ["RateLimiter"]

# KV Namespaces
[[kv_namespaces]]
binding = "CONFIG"
id = "xxxx"

# R2 Storage
[[r2_buckets]]
binding = "ASSETS"
bucket_name = "platform-assets"

# Routes
[[routes]]
pattern = "api.example.com/*"
zone_name = "example.com"

# Secrets (set via wrangler secret put)
# JWT_SECRET, ORIGIN_VERIFY_TOKEN

[build]
command = "npm run build"

[env.staging]
vars = { ENVIRONMENT = "staging" }
routes = [{ pattern = "staging-api.example.com/*", zone_name = "example.com" }]
EOF

cat > worker-test.js << 'EOF'
// Unit tests for Cloudflare Worker
import { unstable_dev } from 'wrangler';
import { describe, expect, it, beforeAll, afterAll } from 'vitest';

describe('API Gateway Worker', () => {
    let worker;
    
    beforeAll(async () => {
        worker = await unstable_dev('worker-api-gateway.js', {
            experimental: { disableExperimentalWarning: true },
            vars: {
                JWT_SECRET: 'test-secret-32-chars-minimum!!!',
                ORIGIN_VERIFY_TOKEN: 'test-verify-token',
            }
        });
    });
    
    afterAll(async () => {
        await worker.stop();
    });
    
    it('should allow health endpoint without auth', async () => {
        const res = await worker.fetch('https://api.example.com/health');
        expect(res.status).not.toBe(401);
    });
    
    it('should reject requests without Bearer token', async () => {
        const res = await worker.fetch('https://api.example.com/api/v2/payments');
        expect(res.status).toBe(401);
    });
    
    it('should add geo-routing headers', async () => {
        const res = await worker.fetch(
            'https://api.example.com/health',
            { headers: { 'CF-IPCountry': 'TH' } }
        );
        expect(res.headers.get('X-Edge-Region')).toBe('TH');
    });
});
EOF

echo "Cloudflare Worker + Durable Objects created"
echo "=== Step 614 Complete: Edge Computing 2.0 ==="
SCRIPT
chmod +x edge-computing-2.sh
echo "Script created: edge-computing-2.sh"
```

**สิ่งที่เรียนรู้:**
- Cloudflare Workers: Edge API gateway (JWT auth + rate limiting + geo-routing) ใน <1ms cold start
- Durable Objects: strongly-consistent distributed state (rate limiter per IP key), atomic storage operations
- Window-based rate limiting: `Math.floor(now/window) * window` สำหรับ sliding window buckets
- Wrangler config: KV namespaces, R2 storage, Durable Object migrations, multi-environment
- Worker testing: `unstable_dev` สำหรับ unit tests

---

## Step 615: FinOps — Cloud Cost Optimization Automation

```bash
cat > finops-automation.sh << 'SCRIPT'
#!/bin/bash
# FinOps: Automated Cloud Cost Optimization

set -euo pipefail

echo "=== FinOps Cost Optimization Platform ==="

# ─── 1. Cost Intelligence Dashboard ────────────────────────────────────────
cat > cost_optimizer.py << 'PYEOF'
#!/usr/bin/env python3
"""
FinOps Cost Optimizer: Automated recommendations and actions.
Integrates with AWS Cost Explorer, Compute Optimizer, and Spot Advisor.
"""

import json
import logging
from datetime import datetime, timedelta
from dataclasses import dataclass, field
from typing import List, Dict, Optional
from enum import Enum

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class SavingsType(Enum):
    RIGHTSIZING = "rightsizing"
    RESERVED_INSTANCES = "reserved_instances"
    SAVINGS_PLANS = "savings_plans"
    SPOT_MIGRATION = "spot_migration"
    IDLE_RESOURCE = "idle_resource"
    STORAGE_TIERING = "storage_tiering"
    DATA_TRANSFER = "data_transfer"

@dataclass
class CostRecommendation:
    resource_id: str
    resource_type: str
    current_monthly_cost: float
    projected_monthly_cost: float
    savings_type: SavingsType
    action: str
    confidence: float  # 0.0 - 1.0
    risk: str  # LOW, MEDIUM, HIGH
    auto_apply: bool = False  # Can be applied automatically
    tags: Dict[str, str] = field(default_factory=dict)

    @property
    def monthly_savings(self) -> float:
        return self.current_monthly_cost - self.projected_monthly_cost

    @property
    def annual_savings(self) -> float:
        return self.monthly_savings * 12

class FinOpsOptimizer:
    # Auto-apply thresholds (only apply if savings > threshold and risk is LOW)
    AUTO_APPLY_MIN_SAVINGS = 100     # $100/month minimum
    AUTO_APPLY_MAX_RISK = "LOW"

    def __init__(self, dry_run: bool = True):
        self.dry_run = dry_run
        self.recommendations: List[CostRecommendation] = []

    def analyze_ec2_rightsizing(self) -> List[CostRecommendation]:
        """Analyze EC2 instances for rightsizing opportunities."""
        # Real impl: boto3 Compute Optimizer API
        # computeoptimizer.get_ec2_instance_recommendations()
        
        # Simulated recommendations
        recs = [
            CostRecommendation(
                resource_id="i-1234567890abcdef0",
                resource_type="EC2",
                current_monthly_cost=800.0,    # m5.4xlarge
                projected_monthly_cost=200.0,  # m5.xlarge (CPU avg 15%)
                savings_type=SavingsType.RIGHTSIZING,
                action="Downsize from m5.4xlarge to m5.xlarge (avg CPU: 15%)",
                confidence=0.87,
                risk="LOW",
                auto_apply=True,
                tags={"team": "payments", "env": "production"},
            ),
            CostRecommendation(
                resource_id="i-0987654321fedcba0",
                resource_type="EC2",
                current_monthly_cost=400.0,    # c5.2xlarge
                projected_monthly_cost=100.0,  # t3.medium (burst)
                savings_type=SavingsType.RIGHTSIZING,
                action="Migrate to t3.medium (burst workload pattern detected)",
                confidence=0.72,
                risk="MEDIUM",
                auto_apply=False,            # Require approval (MEDIUM risk)
                tags={"team": "analytics"},
            ),
        ]
        return recs

    def analyze_savings_plans(self) -> List[CostRecommendation]:
        """Recommend Savings Plans based on usage patterns."""
        # Real impl: boto3 cost-explorer get-savings-plans-purchase-recommendation()
        
        recs = [
            CostRecommendation(
                resource_id="account-123456789",
                resource_type="SavingsPlan-Compute",
                current_monthly_cost=50000.0,
                projected_monthly_cost=35000.0,  # 30% discount with 1yr Compute SP
                savings_type=SavingsType.SAVINGS_PLANS,
                action="Purchase $35k/month Compute Savings Plan (1-year, No Upfront)",
                confidence=0.94,
                risk="LOW",
                auto_apply=False,  # Financial commitment → require approval
                tags={"type": "commitment"},
            )
        ]
        return recs

    def analyze_idle_resources(self) -> List[CostRecommendation]:
        """Find idle or unused resources."""
        recs = [
            CostRecommendation(
                resource_id="vol-0abc123def456789",
                resource_type="EBS-Volume",
                current_monthly_cost=50.0,
                projected_monthly_cost=0.0,
                savings_type=SavingsType.IDLE_RESOURCE,
                action="Delete unattached EBS volume (0 I/O for 30 days)",
                confidence=0.99,
                risk="LOW",
                auto_apply=True,
                tags={"team": "unknown"},
            ),
            CostRecommendation(
                resource_id="i-idle001",
                resource_type="EC2",
                current_monthly_cost=200.0,
                projected_monthly_cost=0.0,
                savings_type=SavingsType.IDLE_RESOURCE,
                action="Terminate EC2 instance (0% CPU for 14 days, no network traffic)",
                confidence=0.95,
                risk="MEDIUM",
                auto_apply=False,
                tags={"team": "dev", "env": "dev"},
            ),
        ]
        return recs

    def analyze_spot_opportunities(self) -> List[CostRecommendation]:
        """Find workloads suitable for Spot instances."""
        recs = [
            CostRecommendation(
                resource_id="batch-job-fleet",
                resource_type="EC2-Spot",
                current_monthly_cost=3000.0,   # On-demand batch workers
                projected_monthly_cost=600.0,  # 80% savings on Spot
                savings_type=SavingsType.SPOT_MIGRATION,
                action="Migrate batch processing to Spot with Karpenter spot-first policy",
                confidence=0.88,
                risk="LOW",  # Fault-tolerant batch workload
                auto_apply=False,
                tags={"team": "data", "workload-type": "batch"},
            )
        ]
        return recs

    def analyze_all(self):
        """Run all analyses and collect recommendations."""
        analyzers = [
            self.analyze_ec2_rightsizing,
            self.analyze_savings_plans,
            self.analyze_idle_resources,
            self.analyze_spot_opportunities,
        ]
        
        for analyzer in analyzers:
            self.recommendations.extend(analyzer())
        
        # Sort by annual savings descending
        self.recommendations.sort(key=lambda r: r.annual_savings, reverse=True)

    def get_auto_applicable(self) -> List[CostRecommendation]:
        """Get recommendations that can be auto-applied."""
        return [
            r for r in self.recommendations
            if r.auto_apply and
            r.monthly_savings >= self.AUTO_APPLY_MIN_SAVINGS and
            r.confidence >= 0.85
        ]

    def apply_recommendation(self, rec: CostRecommendation) -> bool:
        """Apply a recommendation (or simulate in dry-run mode)."""
        if self.dry_run:
            logger.info(f"[DRY RUN] Would apply: {rec.action}")
            return True
        
        logger.info(f"Applying: {rec.action}")
        
        # Real impl: call AWS APIs based on savings_type
        if rec.savings_type == SavingsType.IDLE_RESOURCE:
            if rec.resource_type == "EBS-Volume":
                # boto3: ec2.delete_volume(VolumeId=rec.resource_id)
                pass
        elif rec.savings_type == SavingsType.RIGHTSIZING:
            if rec.resource_type == "EC2":
                # Stop, modify instance type, start
                # ec2.modify_instance_attribute(InstanceId=rec.resource_id, InstanceType=new_type)
                pass
        
        return True

    def generate_report(self) -> Dict:
        total_monthly = sum(r.monthly_savings for r in self.recommendations)
        by_type = {}
        for rec in self.recommendations:
            t = rec.savings_type.value
            by_type[t] = by_type.get(t, 0) + rec.monthly_savings
        
        return {
            "generated_at": datetime.utcnow().isoformat(),
            "total_monthly_savings": round(total_monthly, 2),
            "total_annual_savings": round(total_monthly * 12, 2),
            "recommendations_count": len(self.recommendations),
            "auto_applicable_count": len(self.get_auto_applicable()),
            "savings_by_type": {k: round(v, 2) for k, v in by_type.items()},
            "top_recommendations": [
                {
                    "resource": r.resource_id,
                    "action": r.action,
                    "monthly_savings": round(r.monthly_savings, 2),
                    "risk": r.risk,
                    "auto_apply": r.auto_apply,
                    "confidence": r.confidence,
                }
                for r in self.recommendations[:5]
            ]
        }

optimizer = FinOpsOptimizer(dry_run=True)
optimizer.analyze_all()

report = optimizer.generate_report()
print("=== FinOps Cost Optimization Report ===\n")
print(f"Total Monthly Savings: ${report['total_monthly_savings']:,.2f}")
print(f"Total Annual Savings:  ${report['total_annual_savings']:,.2f}")
print(f"Total Recommendations: {report['recommendations_count']}")
print(f"Auto-Applicable:       {report['auto_applicable_count']}")

print("\nSavings by Type:")
for savings_type, amount in sorted(
    report["savings_by_type"].items(), key=lambda x: x[1], reverse=True
):
    print(f"  {savings_type}: ${amount:,.2f}/month")

print("\nTop Recommendations:")
for i, rec in enumerate(report["top_recommendations"], 1):
    print(f"  {i}. [{rec['risk']}] {rec['action'][:60]}...")
    print(f"     Savings: ${rec['monthly_savings']:,.2f}/month, "
          f"Confidence: {rec['confidence']:.0%}, "
          f"Auto: {'Yes' if rec['auto_apply'] else 'No'}")

# Apply auto-applicable recommendations
auto_recs = optimizer.get_auto_applicable()
print(f"\nApplying {len(auto_recs)} auto-applicable recommendations (dry-run)...")
for rec in auto_recs:
    optimizer.apply_recommendation(rec)
PYEOF

python3 cost_optimizer.py

echo "=== Step 615 Complete: FinOps Automation ==="
SCRIPT
chmod +x finops-automation.sh
bash finops-automation.sh
echo "Script created: finops-automation.sh"
```

**สิ่งที่เรียนรู้:**
- FinOps recommendations: EC2 rightsizing (CPU avg 15% → downsize 75%), Savings Plans (30% discount), Spot migration (80% savings), idle resource cleanup
- Auto-apply logic: savings > $100/month + confidence > 85% + risk = LOW
- CostRecommendation dataclass: monthly/annual savings, confidence score, risk level
- Sorted by annual savings (highest impact first)
- Dry-run mode: simulate actions without making changes

---

## สรุป Part 66

| Step | หัวข้อ | เทคโนโลยีหลัก |
|------|--------|----------------|
| 612 | Global CDN | CloudFront + Lambda@Edge + CF Functions, JWT auth, geo-routing |
| 613 | Multi-Region Active-Active | Route53 latency routing, Aurora Global DB, Failover automation |
| 614 | Edge Computing 2.0 | Cloudflare Workers, Durable Objects rate limiter, Wrangler config |
| 615 | FinOps Automation | Cost Optimizer, rightsizing, Savings Plans, auto-apply recommendations |

**ขั้นตอนต่อไป: Part 67 — Database Engineering at Scale, NewSQL, Distributed Transactions และ Data Mesh**
