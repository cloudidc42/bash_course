# Part 47: Global Load Balancing and Traffic Management

## Module 4: Professional Level — Global Infrastructure

### ขั้นตอนที่ 541: AWS Global Accelerator and CloudFront

**Global load balancing** สำหรับ multi-region deployment

```bash
#!/bin/bash
# global-load-balancer.sh - Global Load Balancing and Traffic Management

set -euo pipefail

AWS_REGION="${AWS_REGION:-ap-southeast-1}"
APP_NAME="${APP_NAME:-production-app}"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }

create_global_accelerator() {
    local accelerator_name="${1:-${APP_NAME}-accelerator}"
    
    log "Creating AWS Global Accelerator: ${accelerator_name}"
    
    local accelerator_arn=$(aws globalaccelerator create-accelerator \
        --name "${accelerator_name}" \
        --ip-address-type IPV4 \
        --enabled \
        --tags "Key=Environment,Value=production" "Key=ManagedBy,Value=automation" \
        --region us-west-2 \
        --query 'Accelerator.AcceleratorArn' \
        --output text)
    
    log "Accelerator created: ${accelerator_arn}"
    
    local listener_arn=$(aws globalaccelerator create-listener \
        --accelerator-arn "${accelerator_arn}" \
        --protocol TCP \
        --port-ranges "FromPort=443,ToPort=443" "FromPort=80,ToPort=80" \
        --client-affinity SOURCE_IP \
        --region us-west-2 \
        --query 'Listener.ListenerArn' \
        --output text)
    
    log "Listener created: ${listener_arn}"
    
    local regions=("us-east-1" "ap-southeast-1" "eu-west-1")
    local weights=(100 100 100)
    
    for i in "${!regions[@]}"; do
        local region="${regions[$i]}"
        local weight="${weights[$i]}"
        
        local alb_arn=$(aws elbv2 describe-load-balancers \
            --names "${APP_NAME}-alb" \
            --region "${region}" \
            --query 'LoadBalancers[0].LoadBalancerArn' \
            --output text 2>/dev/null || echo "")
        
        if [[ -n "${alb_arn}" && "${alb_arn}" != "None" ]]; then
            aws globalaccelerator create-endpoint-group \
                --listener-arn "${listener_arn}" \
                --endpoint-group-region "${region}" \
                --traffic-dial-percentage "${weight}" \
                --endpoint-configurations "EndpointId=${alb_arn},Weight=128,ClientIPPreservationEnabled=true" \
                --health-check-protocol HTTPS \
                --health-check-path "/health" \
                --health-check-interval-seconds 10 \
                --threshold-count 3 \
                --region us-west-2
            
            log "Endpoint group added: ${region} (weight: ${weight})"
        fi
    done
    
    local accelerator_info=$(aws globalaccelerator describe-accelerator \
        --accelerator-arn "${accelerator_arn}" \
        --region us-west-2 \
        --query 'Accelerator.IpSets[*].IpAddresses[]' \
        --output text)
    
    log "Global Accelerator IPs: ${accelerator_info}"
    echo "${accelerator_arn}"
}

create_cloudfront_distribution() {
    local origin_domain="${1:-api.company.com}"
    local distribution_name="${2:-${APP_NAME}-cdn}"
    
    log "Creating CloudFront distribution for: ${origin_domain}"
    
    cat <<JSON > /tmp/cloudfront-config.json
{
    "CallerReference": "${distribution_name}-$(date +%s)",
    "Comment": "${distribution_name} CDN Distribution",
    "DefaultRootObject": "",
    "Origins": {
        "Quantity": 2,
        "Items": [
            {
                "Id": "api-origin",
                "DomainName": "${origin_domain}",
                "CustomOriginConfig": {
                    "HTTPSPort": 443,
                    "OriginProtocolPolicy": "https-only",
                    "OriginSSLProtocols": {"Quantity": 2, "Items": ["TLSv1.2", "TLSv1.3"]},
                    "OriginReadTimeout": 30,
                    "OriginKeepaliveTimeout": 5
                },
                "ConnectionAttempts": 3,
                "ConnectionTimeout": 10
            },
            {
                "Id": "s3-static-origin",
                "DomainName": "${APP_NAME}-static.s3.ap-southeast-1.amazonaws.com",
                "S3OriginConfig": {
                    "OriginAccessIdentity": ""
                },
                "ConnectionAttempts": 3,
                "ConnectionTimeout": 10
            }
        ]
    },
    "OriginGroups": {
        "Quantity": 1,
        "Items": [
            {
                "Id": "api-failover-group",
                "FailoverCriteria": {
                    "StatusCodes": {
                        "Quantity": 4,
                        "Items": [500, 502, 503, 504]
                    }
                },
                "Members": {
                    "Quantity": 2,
                    "Items": [
                        {"OriginId": "api-origin"},
                        {"OriginId": "s3-static-origin"}
                    ]
                }
            }
        ]
    },
    "DefaultCacheBehavior": {
        "TargetOriginId": "api-origin",
        "ViewerProtocolPolicy": "redirect-to-https",
        "AllowedMethods": {
            "Quantity": 7,
            "Items": ["GET", "HEAD", "OPTIONS", "PUT", "POST", "PATCH", "DELETE"],
            "CachedMethods": {"Quantity": 3, "Items": ["GET", "HEAD", "OPTIONS"]}
        },
        "CachePolicyId": "4135ea2d-6df8-44a3-9df3-4b5a84be39ad",
        "OriginRequestPolicyId": "b689b0a8-53d0-40ab-baf2-68738e2966ac",
        "ResponseHeadersPolicyId": "67f7725c-6f97-4210-82d7-5512b31e9d03",
        "Compress": true,
        "FunctionAssociations": {
            "Quantity": 1,
            "Items": [
                {
                    "FunctionARN": "arn:aws:cloudfront::ACCOUNT:function/security-headers",
                    "EventType": "viewer-response"
                }
            ]
        }
    },
    "CacheBehaviors": {
        "Quantity": 2,
        "Items": [
            {
                "PathPattern": "/api/*",
                "TargetOriginId": "api-origin",
                "ViewerProtocolPolicy": "https-only",
                "AllowedMethods": {
                    "Quantity": 7,
                    "Items": ["GET", "HEAD", "OPTIONS", "PUT", "POST", "PATCH", "DELETE"],
                    "CachedMethods": {"Quantity": 2, "Items": ["GET", "HEAD"]}
                },
                "CachePolicyId": "4135ea2d-6df8-44a3-9df3-4b5a84be39ad",
                "TTL": 0,
                "Compress": true
            },
            {
                "PathPattern": "/static/*",
                "TargetOriginId": "s3-static-origin",
                "ViewerProtocolPolicy": "redirect-to-https",
                "AllowedMethods": {
                    "Quantity": 2,
                    "Items": ["GET", "HEAD"],
                    "CachedMethods": {"Quantity": 2, "Items": ["GET", "HEAD"]}
                },
                "DefaultTTL": 86400,
                "MaxTTL": 604800,
                "Compress": true
            }
        ]
    },
    "PriceClass": "PriceClass_All",
    "Enabled": true,
    "HttpVersion": "http2and3",
    "IsIPV6Enabled": true,
    "WebACLId": "arn:aws:wafv2:us-east-1:ACCOUNT:global/webacl/production-waf/WEBACL-ID"
}
JSON
    
    local distribution_id=$(aws cloudfront create-distribution \
        --distribution-config file:///tmp/cloudfront-config.json \
        --query 'Distribution.Id' \
        --output text)
    
    log "CloudFront distribution created: ${distribution_id}"
    echo "${distribution_id}"
}

configure_waf() {
    local web_acl_name="${1:-production-waf}"
    
    log "Creating WAF WebACL: ${web_acl_name}"
    
    aws wafv2 create-web-acl \
        --name "${web_acl_name}" \
        --scope CLOUDFRONT \
        --region us-east-1 \
        --default-action '{"Allow": {}}' \
        --description "Production WAF for ${APP_NAME}" \
        --rules '[
            {
                "Name": "AWSManagedRulesCommonRuleSet",
                "Priority": 1,
                "OverrideAction": {"None": {}},
                "VisibilityConfig": {
                    "SampledRequestsEnabled": true,
                    "CloudWatchMetricsEnabled": true,
                    "MetricName": "CommonRuleSet"
                },
                "Statement": {
                    "ManagedRuleGroupStatement": {
                        "VendorName": "AWS",
                        "Name": "AWSManagedRulesCommonRuleSet"
                    }
                }
            },
            {
                "Name": "AWSManagedRulesKnownBadInputsRuleSet",
                "Priority": 2,
                "OverrideAction": {"None": {}},
                "VisibilityConfig": {
                    "SampledRequestsEnabled": true,
                    "CloudWatchMetricsEnabled": true,
                    "MetricName": "KnownBadInputs"
                },
                "Statement": {
                    "ManagedRuleGroupStatement": {
                        "VendorName": "AWS",
                        "Name": "AWSManagedRulesKnownBadInputsRuleSet"
                    }
                }
            },
            {
                "Name": "RateLimitRule",
                "Priority": 10,
                "Action": {"Block": {}},
                "VisibilityConfig": {
                    "SampledRequestsEnabled": true,
                    "CloudWatchMetricsEnabled": true,
                    "MetricName": "RateLimit"
                },
                "Statement": {
                    "RateBasedStatement": {
                        "Limit": 2000,
                        "AggregateKeyType": "IP"
                    }
                }
            },
            {
                "Name": "GeoBlockRule",
                "Priority": 20,
                "Action": {"Block": {}},
                "VisibilityConfig": {
                    "SampledRequestsEnabled": true,
                    "CloudWatchMetricsEnabled": true,
                    "MetricName": "GeoBlock"
                },
                "Statement": {
                    "GeoMatchStatement": {
                        "CountryCodes": ["CN", "RU", "KP"]
                    }
                }
            }
        ]' \
        --visibility-config '{
            "SampledRequestsEnabled": true,
            "CloudWatchMetricsEnabled": true,
            "MetricName": "'"${web_acl_name}"'"
        }'
    
    log "WAF WebACL created: ${web_acl_name}"
}

setup_route53_health_checks() {
    local domain="${1:-company.com}"
    
    log "Setting up Route53 health checks and failover for: ${domain}"
    
    local primary_check_id=$(aws route53 create-health-check \
        --caller-reference "primary-$(date +%s)" \
        --health-check-config '{
            "Type": "HTTPS",
            "FullyQualifiedDomainName": "api-primary.'"${domain}"'",
            "RequestInterval": 10,
            "FailureThreshold": 2,
            "MeasureLatency": true,
            "EnableSNI": true,
            "Regions": ["us-east-1", "eu-west-1", "ap-southeast-1"],
            "AlarmIdentifier": {
                "Region": "us-east-1",
                "Name": "api-primary-alarm"
            },
            "InsufficientDataHealthStatus": "Unhealthy"
        }' \
        --query 'HealthCheck.Id' \
        --output text)
    
    local secondary_check_id=$(aws route53 create-health-check \
        --caller-reference "secondary-$(date +%s)" \
        --health-check-config '{
            "Type": "HTTPS",
            "FullyQualifiedDomainName": "api-secondary.'"${domain}"'",
            "RequestInterval": 10,
            "FailureThreshold": 2,
            "MeasureLatency": true,
            "EnableSNI": true
        }' \
        --query 'HealthCheck.Id' \
        --output text)
    
    local hosted_zone_id=$(aws route53 list-hosted-zones-by-name \
        --dns-name "${domain}" \
        --query 'HostedZones[0].Id' \
        --output text | awk -F'/' '{print $3}')
    
    aws route53 change-resource-record-sets \
        --hosted-zone-id "${hosted_zone_id}" \
        --change-batch '{
            "Changes": [
                {
                    "Action": "UPSERT",
                    "ResourceRecordSet": {
                        "Name": "api.'"${domain}"'",
                        "Type": "A",
                        "SetIdentifier": "primary",
                        "Failover": "PRIMARY",
                        "HealthCheckId": "'"${primary_check_id}"'",
                        "AliasTarget": {
                            "HostedZoneId": "Z1H1FL5HABSF5",
                            "DNSName": "accelerator.awsglobalaccelerator.com",
                            "EvaluateTargetHealth": true
                        }
                    }
                },
                {
                    "Action": "UPSERT",
                    "ResourceRecordSet": {
                        "Name": "api.'"${domain}"'",
                        "Type": "A",
                        "SetIdentifier": "secondary",
                        "Failover": "SECONDARY",
                        "HealthCheckId": "'"${secondary_check_id}"'",
                        "AliasTarget": {
                            "HostedZoneId": "Z35SXDOTRQ7X7K",
                            "DNSName": "api-secondary.'"${domain}"'.elb.amazonaws.com",
                            "EvaluateTargetHealth": true
                        }
                    }
                }
            ]
        }'
    
    log "Route53 health checks and failover configured for: ${domain}"
}

monitor_global_traffic() {
    log "=== Global Traffic Monitoring ==="
    
    echo "--- CloudFront Distribution Status ---"
    aws cloudfront list-distributions \
        --query 'DistributionList.Items[*].[Id,Status,DomainName,Origins.Items[0].DomainName]' \
        --output table 2>/dev/null || echo "CloudFront not available"
    
    echo ""
    echo "--- Global Accelerator Status ---"
    aws globalaccelerator list-accelerators \
        --region us-west-2 \
        --query 'Accelerators[*].[Name,Status,IpSets[0].IpAddresses[0]]' \
        --output table 2>/dev/null || echo "Global Accelerator not available"
    
    echo ""
    echo "--- Route53 Health Check Status ---"
    aws route53 list-health-checks \
        --query 'HealthChecks[*].[Id,HealthCheckConfig.FullyQualifiedDomainName,HealthCheckConfig.Type]' \
        --output table 2>/dev/null || echo "Route53 not available"
}

case "${1:-help}" in
    "global-accelerator") create_global_accelerator "${2:-}" ;;
    "cloudfront") create_cloudfront_distribution "$2" "${3:-}" ;;
    "waf") configure_waf "${2:-production-waf}" ;;
    "route53-failover") setup_route53_health_checks "${2:-company.com}" ;;
    "monitor") monitor_global_traffic ;;
    *) echo "Usage: $0 {global-accelerator|cloudfront|waf|route53-failover|monitor}" ;;
esac
```

### ขั้นตอนที่ 542: Nginx Ingress Advanced Traffic Management

**Nginx Ingress** สำหรับ advanced traffic routing, canary deployments

```bash
#!/bin/bash
# nginx-ingress-advanced.sh - Advanced Traffic Management with Nginx Ingress

set -euo pipefail

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }

install_nginx_ingress_enterprise() {
    log "Installing Nginx Ingress Controller with enterprise config..."
    
    helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
    helm repo update
    
    cat <<EOF > /tmp/nginx-ingress-values.yaml
controller:
  replicaCount: 3
  
  resources:
    requests:
      cpu: "500m"
      memory: "512Mi"
    limits:
      cpu: "2"
      memory: "2Gi"
  
  config:
    # Performance
    worker-processes: "auto"
    worker-connections: "16384"
    worker-rlimit-nofile: "131072"
    
    # Timeouts
    proxy-connect-timeout: "10"
    proxy-send-timeout: "120"
    proxy-read-timeout: "120"
    
    # Buffer sizes
    proxy-buffer-size: "16k"
    proxy-buffers: "8 16k"
    proxy-busy-buffers-size: "32k"
    
    # SSL/TLS
    ssl-protocols: "TLSv1.2 TLSv1.3"
    ssl-ciphers: "ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256"
    ssl-prefer-server-ciphers: "on"
    ssl-session-cache: "shared:SSL:10m"
    ssl-session-timeout: "1d"
    ssl-session-tickets: "off"
    
    # Security headers
    hsts: "true"
    hsts-max-age: "31536000"
    hsts-include-subdomains: "true"
    hsts-preload: "true"
    
    # Logging
    log-format-upstream: '{"time": "$time_iso8601", "remote_addr": "$proxy_protocol_addr", "x_forwarded_for": "$proxy_add_x_forwarded_for", "request_id": "$req_id", "remote_user": "$remote_user", "bytes_sent": $bytes_sent, "request_time": $request_time, "status": $status, "vhost": "$host", "request_proto": "$server_protocol", "path": "$uri", "request_query": "$args", "request_length": $request_length, "duration": $request_time, "method": "$request_method", "http_referrer": "$http_referer", "http_user_agent": "$http_user_agent"}'
    
    # Rate limiting global
    limit-req-status-code: "429"
    limit-conn-status-code: "429"
    
    # CORS
    enable-cors: "false"
    
    # Compression
    use-gzip: "true"
    gzip-level: "6"
    gzip-types: "text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript"
    
    # ModSecurity WAF
    enable-modsecurity: "true"
    enable-owasp-modsecurity-crs: "true"
    modsecurity-snippet: |
      SecRuleEngine On
      SecAuditLog /dev/stdout
      SecAuditLogFormat JSON
  
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchExpressions:
              - key: app.kubernetes.io/name
                operator: In
                values: [ingress-nginx]
          topologyKey: kubernetes.io/hostname
  
  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: DoNotSchedule
      labelSelector:
        matchLabels:
          app.kubernetes.io/name: ingress-nginx
  
  metrics:
    enabled: true
    serviceMonitor:
      enabled: true
      namespace: monitoring
  
  service:
    externalTrafficPolicy: Local
    annotations:
      service.beta.kubernetes.io/aws-load-balancer-type: nlb
      service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
      service.beta.kubernetes.io/aws-load-balancer-backend-protocol: tcp
  
  autoscaling:
    enabled: true
    minReplicas: 3
    maxReplicas: 20
    targetCPUUtilizationPercentage: 70
    targetMemoryUtilizationPercentage: 80

defaultBackend:
  enabled: true
  replicaCount: 2
  resources:
    requests:
      cpu: "50m"
      memory: "64Mi"
EOF
    
    helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
        --namespace ingress-nginx \
        --create-namespace \
        -f /tmp/nginx-ingress-values.yaml \
        --wait
    
    log "Nginx Ingress Controller installed"
}

create_canary_deployment() {
    local app_name="${1:-api}"
    local namespace="${2:-default}"
    local canary_weight="${3:-10}"
    local stable_version="${4:-v1}"
    local canary_version="${5:-v2}"
    
    log "Creating canary deployment: ${app_name} (${canary_weight}% to ${canary_version})"
    
    cat <<EOF | kubectl apply -f -
# Stable (Production) Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ${app_name}-stable
  namespace: ${namespace}
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/rewrite-target: /\$2
    nginx.ingress.kubernetes.io/use-regex: "true"
spec:
  rules:
    - host: api.company.com
      http:
        paths:
          - path: /(api/)(.*) 
            pathType: Prefix
            backend:
              service:
                name: ${app_name}-${stable_version}-svc
                port:
                  number: 80
---
# Canary Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ${app_name}-canary
  namespace: ${namespace}
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "${canary_weight}"
    nginx.ingress.kubernetes.io/canary-by-header: "X-Canary"
    nginx.ingress.kubernetes.io/canary-by-header-value: "true"
    nginx.ingress.kubernetes.io/canary-by-cookie: "canary"
spec:
  rules:
    - host: api.company.com
      http:
        paths:
          - path: /api/
            pathType: Prefix
            backend:
              service:
                name: ${app_name}-${canary_version}-svc
                port:
                  number: 80
EOF
    
    log "Canary deployment configured: ${canary_weight}% traffic to ${canary_version}"
}

setup_rate_limiting() {
    local app_name="${1:-api}"
    local namespace="${2:-default}"
    local rps_limit="${3:-100}"
    local burst_limit="${4:-200}"
    
    log "Configuring rate limiting for: ${app_name} (${rps_limit} rps)"
    
    cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ${app_name}-ingress
  namespace: ${namespace}
  annotations:
    kubernetes.io/ingress.class: nginx
    
    # Rate limiting
    nginx.ingress.kubernetes.io/limit-rps: "${rps_limit}"
    nginx.ingress.kubernetes.io/limit-rpm: "$(( rps_limit * 60 ))"
    nginx.ingress.kubernetes.io/limit-burst-multiplier: "$(( burst_limit / rps_limit ))"
    nginx.ingress.kubernetes.io/limit-connections: "50"
    
    # Connection timeouts
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "10"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
    
    # Circuit breaker  
    nginx.ingress.kubernetes.io/upstream-fail-timeout: "30"
    nginx.ingress.kubernetes.io/upstream-max-fails: "5"
    
    # CORS
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://company.com"
    nginx.ingress.kubernetes.io/cors-allow-methods: "GET, POST, PUT, DELETE, OPTIONS"
    nginx.ingress.kubernetes.io/cors-allow-headers: "Authorization, Content-Type, X-Request-ID"
    nginx.ingress.kubernetes.io/cors-max-age: "3600"
    
    # Security
    nginx.ingress.kubernetes.io/configuration-snippet: |
      more_set_headers "X-Frame-Options: DENY";
      more_set_headers "X-Content-Type-Options: nosniff";
      more_set_headers "X-XSS-Protection: 1; mode=block";
      more_set_headers "Content-Security-Policy: default-src 'self'";
      more_set_headers "Referrer-Policy: strict-origin-when-cross-origin";
    
    # Auth
    nginx.ingress.kubernetes.io/auth-url: "https://auth.company.com/validate"
    nginx.ingress.kubernetes.io/auth-response-headers: "X-Auth-User-Id, X-Auth-User-Email, X-Auth-Roles"
    
    # Sticky session
    nginx.ingress.kubernetes.io/affinity: cookie
    nginx.ingress.kubernetes.io/session-cookie-name: route
    nginx.ingress.kubernetes.io/session-cookie-expires: "172800"
    nginx.ingress.kubernetes.io/session-cookie-max-age: "172800"
    nginx.ingress.kubernetes.io/session-cookie-change-on-failure: "true"
    
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
    - hosts:
        - api.company.com
      secretName: api-tls
  rules:
    - host: api.company.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: ${app_name}-svc
                port:
                  number: 80
EOF
    
    log "Rate limiting configured for: ${app_name}"
}

configure_nginx_lua_plugin() {
    log "Configuring Nginx Lua plugins for advanced traffic processing..."
    
    kubectl create configmap nginx-lua-plugins \
        --from-literal=request-transformer.lua='
-- Request transformer plugin
local req_body = kong.request.get_body()
local headers = kong.request.get_headers()

-- Add tracing headers
local trace_id = headers["x-trace-id"] or require("kong.tools.utils").uuid()
kong.service.request.set_header("X-Trace-ID", trace_id)
kong.service.request.set_header("X-Request-Start", tostring(ngx.now() * 1000))

-- JWT validation
local auth = headers["authorization"]
if auth then
    local token = auth:match("Bearer (.+)")
    if token then
        -- Validate and decode JWT
        local claims = decode_jwt(token)
        if claims then
            kong.service.request.set_header("X-User-ID", claims.sub)
            kong.service.request.set_header("X-User-Roles", table.concat(claims.roles or {}, ","))
        end
    end
end
' \
        -n ingress-nginx \
        --dry-run=client -o yaml | kubectl apply -f -
    
    log "Nginx Lua plugins configured"
}

monitor_ingress_traffic() {
    log "=== Nginx Ingress Traffic Metrics ==="
    
    kubectl exec -n ingress-nginx \
        "$(kubectl get pod -n ingress-nginx -l 'app.kubernetes.io/name=ingress-nginx' -o name | head -1 | cut -d/ -f2)" -- \
        nginx -T 2>/dev/null | grep -E "worker_processes|worker_connections" | head -5
    
    echo ""
    echo "--- Ingress Resources ---"
    kubectl get ingress -A \
        -o custom-columns="NAMESPACE:.metadata.namespace,NAME:.metadata.name,HOST:.spec.rules[0].host,TLS:.spec.tls[0].secretName" | head -20
    
    echo ""
    echo "--- Traffic Distribution ---"
    kubectl get ingress -A \
        -o jsonpath='{range .items[?(@.metadata.annotations.nginx\.ingress\.kubernetes\.io/canary=="true")]}{.metadata.name}: {.metadata.annotations.nginx\.ingress\.kubernetes\.io/canary-weight}%{"\n"}{end}' | head -10
}

case "${1:-help}" in
    "install") install_nginx_ingress_enterprise ;;
    "canary") create_canary_deployment "$2" "${3:-default}" "${4:-10}" "${5:-v1}" "${6:-v2}" ;;
    "rate-limit") setup_rate_limiting "$2" "${3:-default}" "${4:-100}" "${5:-200}" ;;
    "lua-plugins") configure_nginx_lua_plugin ;;
    "monitor") monitor_ingress_traffic ;;
    *) echo "Usage: $0 {install|canary|rate-limit|lua-plugins|monitor}" ;;
esac
```

### ขั้นตอนที่ 543: Service Mesh Traffic Management with Istio

**Istio advanced traffic management** สำหรับ microservices

```bash
#!/bin/bash
# istio-traffic-management.sh - Advanced Istio Traffic Management

set -euo pipefail

ISTIO_NAMESPACE="${ISTIO_NAMESPACE:-istio-system}"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }

configure_advanced_traffic_routing() {
    local service_name="${1:-productpage}"
    local namespace="${2:-default}"
    
    log "Configuring advanced traffic routing for: ${service_name}"
    
    cat <<EOF | kubectl apply -f -
# Virtual Service with advanced routing
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: ${service_name}-routing
  namespace: ${namespace}
spec:
  hosts:
    - ${service_name}
    - ${service_name}.company.com
  gateways:
    - istio-system/main-gateway
    - mesh
  http:
    # JWT-based routing
    - match:
        - headers:
            end-user:
              exact: premium-user
      route:
        - destination:
            host: ${service_name}
            subset: v3-premium
          weight: 100
      timeout: 30s
      retries:
        attempts: 3
        perTryTimeout: 10s
        retryOn: 5xx,reset,connect-failure
    
    # Header-based canary routing
    - match:
        - headers:
            x-canary:
              exact: "true"
      route:
        - destination:
            host: ${service_name}
            subset: canary
          weight: 100
    
    # Cookie-based routing
    - match:
        - headers:
            cookie:
              regex: "^(.*?;)?(canary=true)(;.*)?$"
      route:
        - destination:
            host: ${service_name}
            subset: canary
    
    # A/B testing: 20% to v2
    - route:
        - destination:
            host: ${service_name}
            subset: stable
          weight: 80
        - destination:
            host: ${service_name}
            subset: canary
          weight: 20
      timeout: 15s
      fault:
        delay:
          percentage:
            value: 0.1
          fixedDelay: 5s
      mirror:
        host: ${service_name}-shadow
        subset: stable
      mirrorPercentage:
        value: 10.0
---
# Destination Rule with circuit breaker
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: ${service_name}-dr
  namespace: ${namespace}
spec:
  host: ${service_name}
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 1000
        connectTimeout: 30ms
        tcpKeepalive:
          time: 7200s
          interval: 75s
      http:
        h2UpgradePolicy: UPGRADE
        http1MaxPendingRequests: 1024
        http2MaxRequests: 1024
        maxRequestsPerConnection: 10
        maxRetries: 3
        useClientProtocol: false
    outlierDetection:
      consecutiveGatewayErrors: 5
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
      minHealthPercent: 50
    loadBalancer:
      simple: LEAST_CONN
      localityLbSetting:
        enabled: true
        failover:
          - from: ap-southeast-1
            to: us-east-1
          - from: us-east-1
            to: eu-west-1
    tls:
      mode: ISTIO_MUTUAL
  subsets:
    - name: stable
      labels:
        version: stable
      trafficPolicy:
        loadBalancer:
          simple: ROUND_ROBIN
    - name: canary
      labels:
        version: canary
      trafficPolicy:
        connectionPool:
          http:
            http1MaxPendingRequests: 10
            maxRequestsPerConnection: 2
    - name: v3-premium
      labels:
        version: v3
        tier: premium
EOF
    
    log "Advanced traffic routing configured for: ${service_name}"
}

create_circuit_breaker() {
    local service_name="$1"
    local namespace="${2:-default}"
    
    log "Configuring circuit breaker for: ${service_name}"
    
    cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: ${service_name}-circuit-breaker
  namespace: ${namespace}
spec:
  host: ${service_name}
  trafficPolicy:
    outlierDetection:
      splitExternalLocalOriginErrors: true
      consecutiveLocalOriginFailures: 5
      consecutiveGatewayErrors: 5
      consecutive5xxErrors: 3
      interval: 10s
      baseEjectionTime: 30s
      maxEjectionPercent: 100
      minHealthPercent: 0
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 100
        maxRequestsPerConnection: 1
EOF
    
    log "Circuit breaker configured for: ${service_name}"
}

setup_mutual_tls() {
    local namespace="${1:-default}"
    local mode="${2:-STRICT}"
    
    log "Configuring mTLS (${mode}) for namespace: ${namespace}"
    
    cat <<EOF | kubectl apply -f -
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: ${namespace}
spec:
  mtls:
    mode: ${mode}
---
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-authenticated
  namespace: ${namespace}
spec:
  action: ALLOW
  rules:
    - from:
        - source:
            principals:
              - cluster.local/ns/default/sa/frontend
              - cluster.local/ns/default/sa/api-gateway
      to:
        - operation:
            methods: ["GET", "POST", "PUT", "DELETE"]
    - from:
        - source:
            namespaces: ["monitoring"]
      to:
        - operation:
            paths: ["/metrics"]
EOF
    
    log "mTLS configured for namespace: ${namespace}"
}

create_istio_gateway() {
    local gateway_name="${1:-main-gateway}"
    local hostname="${2:-*.company.com}"
    
    log "Creating Istio Gateway: ${gateway_name}"
    
    cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: ${gateway_name}
  namespace: ${ISTIO_NAMESPACE}
spec:
  selector:
    istio: ingressgateway
  servers:
    - port:
        number: 443
        name: https
        protocol: HTTPS
      tls:
        mode: SIMPLE
        credentialName: wildcard-tls
        minProtocolVersion: TLSV1_2
        cipherSuites:
          - ECDHE-RSA-AES256-GCM-SHA384
          - ECDHE-RSA-AES128-GCM-SHA256
      hosts:
        - "${hostname}"
    - port:
        number: 80
        name: http
        protocol: HTTP
      tls:
        httpsRedirect: true
      hosts:
        - "${hostname}"
EOF
    
    log "Istio Gateway created: ${gateway_name}"
}

monitor_istio_traffic() {
    log "=== Istio Traffic Overview ==="
    
    echo "--- VirtualServices ---"
    kubectl get virtualservice -A \
        -o custom-columns="NAMESPACE:.metadata.namespace,NAME:.metadata.name,HOSTS:.spec.hosts[*],GATEWAYS:.spec.gateways[*]"
    
    echo ""
    echo "--- DestinationRules ---"
    kubectl get destinationrule -A \
        -o custom-columns="NAMESPACE:.metadata.namespace,NAME:.metadata.name,HOST:.spec.host,TLS:.spec.trafficPolicy.tls.mode"
    
    echo ""
    echo "--- PeerAuthentication ---"
    kubectl get peerauthentication -A \
        -o custom-columns="NAMESPACE:.metadata.namespace,NAME:.metadata.name,MODE:.spec.mtls.mode"
    
    echo ""
    echo "--- Istio Proxy Status ---"
    istioctl proxy-status 2>/dev/null | head -20 || \
        kubectl get pod -A -l security.istio.io/tlsMode -o custom-columns="NAMESPACE:.metadata.namespace,NAME:.metadata.name,STATUS:.status.phase" | head -20
}

case "${1:-help}" in
    "advanced-routing") configure_advanced_traffic_routing "${2:-api}" "${3:-default}" ;;
    "circuit-breaker") create_circuit_breaker "$2" "${3:-default}" ;;
    "mtls") setup_mutual_tls "${2:-default}" "${3:-STRICT}" ;;
    "gateway") create_istio_gateway "${2:-main-gateway}" "${3:-*.company.com}" ;;
    "monitor") monitor_istio_traffic ;;
    *) echo "Usage: $0 {advanced-routing|circuit-breaker|mtls|gateway|monitor}" ;;
esac
```

---

## สรุป Part 47

ในส่วนนี้เราได้เรียนรู้:

| ขั้นตอน | หัวข้อ | เครื่องมือหลัก |
|---------|--------|----------------|
| 541 | Global Load Balancing | AWS Global Accelerator, CloudFront, WAF, Route53 Failover |
| 542 | Nginx Ingress Traffic Mgmt | Canary Deployments, Rate Limiting, ModSecurity WAF |
| 543 | Istio Advanced Routing | VirtualService, DestinationRule, Circuit Breaker, mTLS |

### ขั้นตอนต่อไป: Part 48 - Enterprise Security Architecture
