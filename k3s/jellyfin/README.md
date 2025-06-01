# Jellyfin Setup with Cloudflare Tunnel

This document describes how to set up Jellyfin media server with Cloudflare Tunnel for secure external access.

## Prerequisites

- A Kubernetes cluster (k3s in this case)
- A Cloudflare account
- A domain name managed by Cloudflare
- `cloudflared` CLI tool installed locally

## Setup Steps

### 1. Jellyfin Deployment

The Jellyfin service is deployed in the `jellyfin` namespace with the following components:

- Service exposing ports:
  - 8096 (HTTP)
  - 8920 (HTTPS)
  - 7359 (Auto-discovery, UDP)
  - 1900 (DLNA, UDP)

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: jellyfin
  namespace: jellyfin
spec:
  selector:
    app: jellyfin
  ports:
    - name: http
      port: 8096
      targetPort: 8096
      nodePort: 30080
    - name: https
      port: 8920
      targetPort: 8920
      nodePort: 30081
    - name: auto-discovery
      port: 7359
      targetPort: 7359
      protocol: UDP
    - name: dlna
      port: 1900
      targetPort: 1900
      protocol: UDP
  type: NodePort
```

### 2. Cloudflare Tunnel Setup

#### 2.1 Create Tunnel

```bash
# Create a new tunnel
cloudflared tunnel create HL_K8_JF

# Note the tunnel ID from the output
# Example: 92e9e906-7554-4b35-bf7d-39a76d05f1cb
```

#### 2.2 Kubernetes Resources

Create the following Kubernetes resources in the `cloudflare` namespace:

1. **Namespace**:
```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: cloudflare
```

2. **ConfigMap**:
```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cloudflared-config
  namespace: cloudflare
data:
  config.yaml: |
    tunnel: <your-tunnel-id>
    credentials-file: /etc/cloudflared/credentials.json
    ingress:
      - hostname: jellyfin.your-domain.com
        service: http://jellyfin.jellyfin:8096
      - service: http_status:404
```

3. **Deployment**:
```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cloudflared
  namespace: cloudflare
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cloudflared
  template:
    metadata:
      labels:
        app: cloudflared
    spec:
      containers:
      - name: cloudflared
        image: cloudflare/cloudflared:2024.2.0
        args:
        - tunnel
        - --config
        - /etc/cloudflared/config/config.yaml
        - run
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
        securityContext:
          capabilities:
            add:
            - NET_ADMIN
        volumeMounts:
        - name: config
          mountPath: /etc/cloudflared/config
          readOnly: true
        - name: cert
          mountPath: /etc/cloudflared
          readOnly: true
      volumes:
      - name: config
        configMap:
          name: cloudflared-config
      - name: cert
        secret:
          secretName: cloudflared-cert
```

#### 2.3 Create Kubernetes Secret

```bash
# Create secret from tunnel credentials
kubectl create secret generic cloudflared-cert \
  --namespace cloudflare \
  --from-file=credentials.json=/home/mohammad/.cloudflared/<tunnel-id>.json
```

#### 2.4 Apply Kubernetes Resources

```bash
# Apply all resources
kubectl apply -f k3s/cloudflare/namespace.yaml
kubectl apply -f k3s/cloudflare/configmap.yaml
kubectl apply -f k3s/cloudflare/deployment.yaml
```

### 3. DNS Configuration

1. Go to Cloudflare DNS settings
2. Add a CNAME record:
   - Name: jellyfin (or your preferred subdomain)
   - Target: <your-tunnel-id>.cfargotunnel.com
   - Proxy status: Proxied

## Verification

1. Check tunnel status:
```bash
kubectl get pods -n cloudflare
kubectl logs -n cloudflare <pod-name>
```

2. Access Jellyfin:
   - Open https://jellyfin.your-domain.com in your browser
   - You should see the Jellyfin login page

## Troubleshooting

1. If the tunnel fails to connect:
   - Check the pod logs for errors
   - Verify the credentials file is correct
   - Ensure the tunnel ID matches in both ConfigMap and DNS settings

2. If Jellyfin is not accessible:
   - Verify the service is running: `kubectl get pods -n jellyfin`
   - Check service endpoints: `kubectl get endpoints -n jellyfin`
   - Verify DNS propagation: `dig jellyfin.your-domain.com`

## Security Notes

- The tunnel provides secure access without exposing your internal network
- All traffic is encrypted through Cloudflare's network
- No need to open ports on your firewall
- Access can be further restricted using Cloudflare Access policies 