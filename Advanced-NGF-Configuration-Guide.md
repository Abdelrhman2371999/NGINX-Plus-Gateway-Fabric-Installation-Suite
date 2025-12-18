Based on your resources, here's a comprehensive summary of advanced NGINX Gateway Fabric (NGF) configuration and traffic management features:

# 🚦 Advanced NGINX Gateway Fabric Configuration Guide

## 📋 **Overview**
This guide covers advanced NGF configuration including **MetalLB load balancer setup**, **snippets for custom NGINX directives**, and **traffic management patterns** for production environments.

## ⚡ **Quick Setup Commands**

### **1. MetalLB Load Balancer Installation**
```bash
# Install MetalLB (replaces NodePort for production)
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.15.2/config/manifests/metallb-native.yaml

# Wait for MetalLB to be ready
kubectl wait --namespace metallb-system \
  --for=condition=ready pod \
  --selector=app=metallb \
  --timeout=90s
```

### **2. MetalLB IP Address Pool Configuration**
```yaml
# metallb-ip-pool.yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: production-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.1.100-192.168.1.110  # Your IP range
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: l2-advert
  namespace: metallb-system
spec:
  ipAddressPools:
  - production-pool
```
Apply: `kubectl apply -f metallb-ip-pool.yaml`

### **3. Update NGF Service to LoadBalancer Type**
```bash
# Upgrade NGF Helm installation
helm upgrade ngf oci://ghcr.io/nginx/charts/nginx-gateway-fabric \
  --set nginx.service.type=LoadBalancer \
  --reuse-values \
  -n nginx-gateway
```

## 🔧 **NGF Snippets: Custom NGINX Directives**

### **What Are Snippets?**
Snippets allow injection of raw NGINX directives into NGF configuration at specific contexts (HTTP, Server, Location).

### **Configuration Example**
```yaml
# custom-snippets.yaml
apiVersion: gateway.nginx.org/v1alpha1
kind: NginxGateway
metadata:
  name: ngf-config
  namespace: nginx-gateway
spec:
  http:
    # HTTP-level snippets (affects all traffic)
    snippets:
      customHttpSnippet: |
        # Custom logging format
        log_format custom '$remote_addr - $remote_user [$time_local] '
                          '"$request" $status $body_bytes_sent '
                          '"$http_referer" "$http_user_agent"';
        
        # Rate limiting zone
        limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
        
  servers:
    # Server-level snippets (per Gateway listener)
    - listener: 
        port: 80
        protocol: HTTP
      snippets:
        customServerSnippet: |
          # Enable custom log format
          access_log /var/log/nginx/access.log custom;
          
          # Security headers
          add_header X-Frame-Options "SAMEORIGIN" always;
          add_header X-Content-Type-Options "nosniff" always;
          add_header Referrer-Policy "strict-origin-when-cross-origin" always;
```

### **Common Snippet Use Cases**
| Use Case | Snippet Type | Example Directive |
|----------|-------------|-------------------|
| **Custom logging** | HTTP/Server | `log_format`, `access_log` |
| **Security headers** | Server/Location | `add_header` |
| **Rate limiting** | HTTP/Location | `limit_req_zone`, `limit_req` |
| **CORS configuration** | Server/Location | `add_header 'Access-Control-Allow-*'` |
| **Buffer tuning** | HTTP/Server | `client_body_buffer_size`, `proxy_buffers` |
| **Timeout settings** | HTTP/Location | `proxy_read_timeout`, `keepalive_timeout` |

## 🚦 **Traffic Management Patterns**

### **1. Basic Routing Architecture**
```
External Traffic → MetalLB LoadBalancer → NGF Gateway → HTTPRoute → Backend Services
```

**Basic Gateway Configuration:**
```yaml
# basic-gateway.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: production-gateway
spec:
  gatewayClassName: nginx
  listeners:
  - name: web
    port: 80
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: All
  - name: secure-web
    port: 443
    protocol: HTTPS
    tls:
      mode: Terminate
      certificateRefs:
      - name: tls-certificate
    allowedRoutes:
      namespaces:
        from: All
```

### **2. Advanced Routing (from Lab 2)**
**Path-based routing with multiple backends:**
```yaml
# advanced-routing.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: api-route
spec:
  parentRefs:
  - name: production-gateway
  hostnames:
  - "api.example.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /v1/users
    backendRefs:
    - name: user-service
      port: 8080
  - matches:
    - path:
        type: PathPrefix
        value: /v1/orders
    backendRefs:
    - name: order-service
      port: 8080
  - matches:
    - path:
        type: PathPrefix
        value: /admin
    filters:
    - type: RequestHeaderModifier
      requestHeaderModifier:
        add:
        - name: X-Admin-Access
          value: "true"
    backendRefs:
    - name: admin-service
      port: 8080
```

### **3. Traffic Splitting (from Lab 5)**
**Canary deployments and A/B testing:**
```yaml
# traffic-splitting.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: canary-release
spec:
  parentRefs:
  - name: production-gateway
  hostnames:
  - "app.example.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    # 90% traffic to stable version
    - name: app-stable
      port: 8080
      weight: 90
    # 10% traffic to canary version  
    - name: app-canary
      port: 8080
      weight: 10
    filters:
    - type: RequestHeaderModifier
      requestHeaderModifier:
        add:
        - name: X-Traffic-Split
          value: "90-10-stable-canary"
```

### **4. Header-based Routing**
```yaml
# header-routing.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: mobile-api-route
spec:
  parentRefs:
  - name: production-gateway
  rules:
  - matches:
    - headers:
      - type: Exact
        name: User-Agent
        value: "Mobile-App/v2.0"
    backendRefs:
    - name: mobile-optimized-api
      port: 8080
  - matches:
    - headers:
      - type: RegularExpression
        name: User-Agent
        value: ".*Android.*"
    backendRefs:
    - name: android-specific-api
      port: 8080
```

## 🎯 **Production-Ready Architecture**

### **Complete Architecture Diagram**
```
┌─────────────────────────────────────────────────────────────┐
│                    External Users                           │
└───────────────────────────┬─────────────────────────────────┘
                            │
                    ┌───────▼────────┐
                    │   DNS: app.com │
                    └───────┬────────┘
                            │
                    ┌───────▼────────┐
                    │   MetalLB VIP  │
                    │  192.168.1.100 │
                    └───────┬────────┘
                            │
                    ┌───────▼─────────────────────────────────┐
                    │      NGF Gateway                        │
                    │  • TLS Termination                      │
                    │  • Rate Limiting                        │
                    │  • Security Headers                     │
                    │  • Access Logging                       │
                    └───────┬─────────────────────────────────┘
                            │
                    ┌───────▼─────────────┐
                    │   HTTPRoute Rules    │
                    │  • Path-based routing│
                    │  • Traffic splitting │
                    │  • Header filtering  │
                    └───────┬─────────────┘
                            │
┌───────────────────┬───────┼──────────────────────────┬───┐
│                   │       │                          │   │
│       ┌───────────▼──┐  ┌─▼─────────┐  ┌─────────────▼──┐|
│       │   Service A  │  │ Service B │  │   Service C    │|
│       │  (v1-stable) │  │ (v2-canary│  │  (admin-only)  │|
│       │    90%       │  │   10%)    │  │                │|
│       └──────────────┘  └───────────┘  └────────────────┘|
│                                                          │
└──────────────────────────────────────────────────────────┘
```

## ⚙️ **Configuration Management Workflow**

### **1. Development Pipeline**
```bash
# 1. Create configuration files
vim gateway.yaml
vim httproute.yaml
vim snippets.yaml

# 2. Apply configurations
kubectl apply -f gateway.yaml -f httproute.yaml -f snippets.yaml

# 3. Verify configuration
kubectl get gateway,httproute -A
kubectl describe nginxgateway ngf-config -n nginx-gateway

# 4. Test routing
curl -H "Host: api.example.com" http://192.168.1.100/v1/users
curl -H "Host: app.example.com" http://192.168.1.100/
```

### **2. Validation Commands**
```bash
# Check NGF configuration
kubectl logs -l app.kubernetes.io/instance=ngf -n nginx-gateway

# Check MetalLB status
kubectl get pods -n metallb-system
kubectl get svc -n nginx-gateway

# Test external access
curl -v http://192.168.1.100
curl -v https://app.example.com

# Monitor traffic split
watch -n 2 'kubectl logs deployment/app-stable --tail=5'
```

## 🛡️ **Security Best Practices**

### **1. Snippet Security Considerations**
```nginx
# DO: Restrict sensitive configurations
add_header X-Content-Type-Options "nosniff";
add_header X-Frame-Options "DENY";

# DON'T: Expose sensitive information
# server_tokens on;  # Avoid - reveals NGINX version

# DO: Implement rate limiting
limit_req_zone $binary_remote_addr zone=auth:10m rate=5r/m;

# Location block for login
location /login {
    limit_req zone=auth burst=10 nodelay;
    proxy_pass http://auth-service;
}
```

### **2. Recommended Snippets for Production**
```yaml
# production-snippets.yaml
spec:
  http:
    snippets:
      main: |
        # Security headers template
        map $upstream_http_content_type $content_security_policy {
          default "default-src 'self'; script-src 'self' 'unsafe-inline';";
          "~*text/html" "default-src 'self'; script-src 'self' 'unsafe-inline' https://trusted.cdn.com;";
        }
        
        # Bot protection
        map $http_user_agent $limit_bots {
          default 0;
          ~*(googlebot|bingbot|Slurp|DuckDuckBot) 0;
          ~*(http|scan|spider|crawl|bot) 1;
        }
        
  servers:
    - listener:
        port: 443
        protocol: HTTPS
      snippets:
        main: |
          # SSL hardening
          ssl_protocols TLSv1.2 TLSv1.3;
          ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512;
          ssl_prefer_server_ciphers off;
          ssl_session_cache shared:SSL:10m;
          
          # HSTS
          add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
```

## 🚨 **Troubleshooting Guide**

| Issue | Check | Solution |
|-------|-------|----------|
| **MetalLB no IP assigned** | `kubectl get svc -n nginx-gateway` | Verify IP pool range matches network |
| **Snippets not applied** | `kubectl describe nginxgateway` | Check snippet syntax and indentation |
| **Traffic not splitting** | `kubectl describe httproute` | Verify weight sums to 100 |
| **HTTPS not working** | `kubectl get secret tls-certificate` | Ensure TLS certificate exists |
| **Routes not matching** | `kubectl logs -l app=ngf` | Check hostname and path matching |

## 📊 **Monitoring Configuration**

### **Enable NGF Metrics**
```yaml
# metrics-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ngf-metrics
  namespace: nginx-gateway
data:
  nginx-metrics.conf: |
    server {
      listen 9113;
      location /stub_status {
        stub_status;
      }
      location /metrics {
        vhost_traffic_status_display;
        vhost_traffic_status_display_format html;
      }
    }
```

## 🔄 **Rollback Procedures**

```bash
# 1. List configuration history
kubectl get httproute --sort-by=.metadata.creationTimestamp

# 2. Rollback to previous version
kubectl rollout undo httproute/my-route

# 3. Disable problematic snippets
kubectl patch nginxgateway ngf-config -n nginx-gateway \
  --type='json' -p='[{"op": "remove", "path": "/spec/http/snippets"}]'

# 4. Emergency fallback to stable route
kubectl apply -f emergency-route.yaml
```

---

**Key Takeaways:**
1. **MetalLB** provides production-grade LoadBalancer for on-prem/cloud environments
2. **Snippets** enable fine-grained NGINX customization while maintaining Gateway API abstraction
3. **Traffic splitting** supports canary releases and A/B testing natively
4. **Advanced routing** allows complex traffic management based on paths, headers, and weights
5. **Always test snippets** in staging before production deployment

This configuration transforms NGF from a basic ingress controller into a full-featured application delivery platform suitable for production workloads.
