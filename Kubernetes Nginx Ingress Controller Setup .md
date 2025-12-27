# **Kubernetes Nginx Ingress Controller Setup Guide**

## **Overview**
This guide provides step-by-step instructions for installing and configuring an Nginx Ingress Controller on a Kubernetes cluster master node. The setup allows external access to services running in your Kubernetes cluster.

## **Prerequisites**
- Kubernetes cluster with master node
- `kubectl` configured to access the cluster
- Master node IP address
- Cluster admin permissions

## **Installation Steps**

### **1. Create Deployment YAML**

Create `nginx-ingress.yaml`:
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ingress-nginx
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-ingress
  namespace: ingress-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-ingress
  template:
    metadata:
      labels:
        app: nginx-ingress
    spec:
      tolerations:
      - key: "node-role.kubernetes.io/master"
        operator: "Exists"
        effect: "NoSchedule"
      - key: "node-role.kubernetes.io/control-plane"
        operator: "Exists"
        effect: "NoSchedule"
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
        - containerPort: 443
        volumeMounts:
        - name: nginx-config
          mountPath: /etc/nginx/nginx.conf
          subPath: nginx.conf
      volumes:
      - name: nginx-config
        configMap:
          name: nginx-config
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
  namespace: ingress-nginx
data:
  nginx.conf: |
    events {}
    http {
      server {
        listen 80;
        location / {
          return 200 "NGINX Ingress Controller is working!\n";
        }
        location /healthz {
          return 200 "healthy\n";
        }
      }
    }
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-ingress-service
  namespace: ingress-nginx
spec:
  type: NodePort
  selector:
    app: nginx-ingress
  ports:
  - name: http
    port: 80
    targetPort: 80
  - name: https
    port: 443
    targetPort: 443
```

### **2. Apply the Configuration**
```bash
kubectl apply -f nginx-ingress.yaml
```

### **3. Check Installation Status**
```bash
kubectl get all -n ingress-nginx
```
Expected output:
```
NAME                                 READY   STATUS    RESTARTS   AGE
pod/nginx-ingress-xxxxxxxxxx-xxxxx   1/1     Running   0          1m

NAME                            TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
service/nginx-ingress-service   NodePort   10.XXX.XXX.XXX  <none>        80:3XXXX/TCP,443:3XXXX/TCP   1m

NAME                            READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx-ingress   1/1     1            1           1m
```

## **Accessing the Ingress Controller**

### **Get Access Information**
```bash
# Get Master Node IP
MASTER_IP=$(kubectl get node k8s-master -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}')

# Get NodePort
NODE_PORT=$(kubectl get svc nginx-ingress-service -n ingress-nginx -o jsonpath='{.spec.ports[0].nodePort}')

echo "Access URL: http://$MASTER_IP:$NODE_PORT"
```

### **Test the Ingress Controller**
```bash
# Basic test
curl http://$MASTER_IP:$NODE_PORT

# Health check
curl http://$MASTER_IP:$NODE_PORT/healthz
```

## **Troubleshooting**

### **Common Issues and Solutions**

#### **1. Pod Stuck in "Pending" State**
**Problem**: Pod cannot be scheduled due to node taints
```bash
# Check taints on master node
kubectl describe node k8s-master | grep -i taint

# Remove taints if present
kubectl taint nodes k8s-master node-role.kubernetes.io/control-plane-
kubectl taint nodes k8s-master node-role.kubernetes.io/master-
```

#### **2. "Connection Refused" Error**
**Problem**: Using wrong port number
```bash
# Always use NodePort, not port 80
# Correct: http://<IP>:<NodePort>
# Wrong: http://<IP>:80

# Get the actual NodePort
kubectl get svc nginx-ingress-service -n ingress-nginx
```

#### **3. Pod Running but Not Accessible**
**Problem**: Service selector doesn't match pod labels
```bash
# Check service selector
kubectl describe svc nginx-ingress-service -n ingress-nginx | grep Selector

# Check pod labels
kubectl get pod -n ingress-nginx --show-labels

# They must match: app=nginx-ingress
```

### **Diagnostic Commands**
```bash
# Check pod logs
kubectl logs -n ingress-nginx deployment/nginx-ingress

# Check service details
kubectl describe svc nginx-ingress-service -n ingress-nginx

# Check endpoints
kubectl get endpoints -n ingress-nginx

# Test from inside cluster
kubectl run test-curl --image=curlimages/curl --restart=Never --rm -it -- curl http://nginx-ingress-service.ingress-nginx:80
```

## **Using Ingress Rules**

### **Create a Sample Application**
```yaml
# sample-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: web
        image: nginx:alpine
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 80
```

### **Create Ingress Resource**
```yaml
# web-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /web
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

### **Apply and Test**
```bash
# Deploy application and ingress
kubectl apply -f sample-app.yaml
kubectl apply -f web-ingress.yaml

# Test the ingress route
curl http://$MASTER_IP:$NODE_PORT/web
```

## **Configuration Options**

### **Custom Nginx Configuration**
Edit the ConfigMap to modify nginx settings:
```bash
kubectl edit configmap nginx-config -n ingress-nginx
```

### **Change NodePort**
To use a specific NodePort:
```bash
kubectl edit svc nginx-ingress-service -n ingress-nginx
```
Change:
```yaml
ports:
- name: http
  port: 80
  targetPort: 80
  nodePort: 30080  # Your desired port (30000-32767)
```

### **Scale the Deployment**
```bash
# Increase replicas for high availability
kubectl scale deployment nginx-ingress -n ingress-nginx --replicas=2
```

## **Maintenance**

### **Update Nginx Image**
```bash
kubectl set image deployment/nginx-ingress nginx=nginx:latest -n ingress-nginx
```

### **Delete Everything**
```bash
kubectl delete namespace ingress-nginx
```

### **Monitor Resources**
```bash
# Monitor pod resource usage
kubectl top pods -n ingress-nginx

# Check events
kubectl get events -n ingress-nginx --sort-by='.lastTimestamp'
```

## **Security Considerations**

1. **Use specific image tags** instead of `:latest`
2. **Configure resource limits** in production
3. **Enable TLS** for HTTPS traffic
4. **Use NetworkPolicies** to restrict access
5. **Regularly update** nginx and base images

## **Quick Reference**

| Command | Purpose |
|---------|---------|
| `kubectl get all -n ingress-nginx` | Check all resources |
| `kubectl logs -n ingress-nginx deployment/nginx-ingress` | View logs |
| `kubectl describe pod -n ingress-nginx` | Detailed pod info |
| `curl http://<IP>:<NodePort>` | Test access |
| `kubectl edit svc -n ingress-nginx` | Modify service |

## **Success Indicators**
- ✅ Pod status shows `Running`
- ✅ Service has a `NodePort` assigned
- ✅ `curl http://<IP>:<NodePort>` returns "NGINX Ingress Controller is working!"
- ✅ `curl http://<IP>:<NodePort>/healthz` returns "healthy"

## **Next Steps**
1. Configure HTTPS/TLS certificates
2. Set up monitoring and alerts
3. Configure custom ingress rules for applications
4. Implement authentication/authorization
5. Set up logging and analytics

---

**Note**: This setup is for development/testing. For production, consider using the official Nginx Ingress Controller with proper security configurations.
