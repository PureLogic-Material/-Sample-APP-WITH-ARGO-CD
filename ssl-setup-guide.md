# SSL Setup Guide with cert-manager and Cloudflare

## 1. Install cert-manager

First, we'll install cert-manager in your cluster:

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.11.0/cert-manager.yaml
```

Wait for cert-manager to be ready:

```bash
kubectl wait --for=condition=ready pod -l app.kubernetes.io/instance=cert-manager -n cert-manager
```

## 2. Create a Cloudflare API Token

1. Log in to your Cloudflare account
2. Go to "My Profile" > "API Tokens"
3. Click "Create Token"
4. Use the "Edit zone DNS" template
5. Under "Zone Resources", select your domain
6. Create the token and copy it

## 3. Create a Kubernetes Secret for Cloudflare API Token

```bash
kubectl create secret generic cloudflare-api-token-secret \
  --from-literal=api-token=<your-cloudflare-api-token> \
  -n cert-manager
```

Replace `<your-cloudflare-api-token>` with the token you just created.

## 4. Create a ClusterIssuer

Save this as `cluster-issuer.yaml`:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: your-email@example.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod-account-key
    solvers:
    - dns01:
        cloudflare:
          email: your-cloudflare-email@example.com
          apiTokenSecretRef:
            name: cloudflare-api-token-secret
            key: api-token
```

Apply it:

```bash
kubectl apply -f cluster-issuer.yaml
```

## 5. Update your Ingress

Update your existing Ingress or create a new one. Here's an example:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  tls:
  - hosts:
    - yourdomain.com
    secretName: yourdomain-tls
  rules:
  - host: yourdomain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: your-service-name
            port: 
              number: 80
```

Apply this Ingress:

```bash
kubectl apply -f your-ingress.yaml
```

## 6. Verify the Certificate

Check the status of your certificate:

```bash
kubectl get certificate
```

When it's ready, you should see "True" under the READY column.

## 7. Update DNS in Cloudflare

Ensure your domain in Cloudflare points to your Ingress Controller's external IP. You can find this IP with:

```bash
kubectl get svc -n ingress-nginx
```

Look for the EXTERNAL-IP of the ingress-nginx-controller service.

## 8. Test Your SSL Setup

Visit https://yourdomain.com in your browser. You should see a valid SSL certificate.
