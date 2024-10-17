# SSL Setup Guide with cert-manager and GoDaddy

## 1. Install cert-manager

First, install cert-manager in your cluster:

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.11.0/cert-manager.yaml

OR

kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.13.0/cert-manager.yaml
```

Wait for cert-manager to be ready:

```bash
kubectl wait --for=condition=ready pod -l app.kubernetes.io/instance=cert-manager -n cert-manager
```

## 2. Create a GoDaddy API Key

1. Log in to your GoDaddy account
2. Go to https://developer.godaddy.com/keys
3. Create a production API key and secret
4. Note down the API key and secret

## 3. Create a Kubernetes Secret for GoDaddy API Key

```bash
kubectl create secret generic godaddy-api-key \
  --from-literal=key=h1eM8yRupYbE_8cvhHXe6572TxNP6GCrrd3 \
  --from-literal=secret=UnSJpvg522f4kTAnmgVy2G \
  -n cert-manager

  OR

  cat <<EOF > secret.yml
apiVersion: v1
kind: Secret
metadata:
  name: godaddy-api-key
type: Opaque
stringData:
  token: h1eM8yRupYbE_8cvhHXe6572TxNP6GCrrd3:UnSJpvg522f4kTAnmgVy2G
EOF
```
```bash
kubectl apply -f secret.yml -n cert-manager
```

Replace `<your-godaddy-api-key>` and `<your-godaddy-api-secret>` with your actual GoDaddy API key and secret.

## 4. Install cert-manager-webhook-godaddy

We need to install a webhook that allows cert-manager to interact with GoDaddy's API:

```bash
# export DOMAIN=test.mujahidhussain.store  # replace with your domain
# helm install -n cert-manager godaddy-webhook ./deploy/charts/godaddy-webhook --set groupName=$DOMAIN

OR

# helm repo add cert-manager-webhook-godaddy https://github.com/snowdrop/godaddy-webhook
# helm repo add godaddy-webhook https://snowdrop.github.io/godaddy-webhook
# helm repo update
# helm install cert-manager-webhook-godaddy cert-manager-webhook-godaddy/cert-manager-webhook-godaddy

export DOMAIN=test.mujahidhussain.store 
helm repo add godaddy-webhook https://snowdrop.github.io/godaddy-webhook
helm repo update
helm install godaddy-webhook godaddy-webhook/godaddy-webhook -n cert-manager --set groupName=$DOMAIN


# To uninstall 
helm uninstall godaddy-webhook -n cert-manager 
```

## 5. Create a ClusterIssuer

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
        webhook:
          groupName: acme.mycompany.com
          solverName: godaddy
          config:
            apiKeySecretRef:
              name: godaddy-api-key
              key: token
```
```bash
cat <<EOF > cluster-issuer.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    # ACME Server
    # prod : https://acme-v02.api.letsencrypt.org/directory
    # staging : https://acme-staging-v02.api.letsencrypt.org/directory
    server: https://acme-v02.api.letsencrypt.org/directory
    # ACME Email address
    email: rajazainjanjua333@gmail.com
    privateKeySecretRef:
      name: letsencrypt-prod-account-key
    solvers:
    - selector:
        dnsZones:
        - 'test.mujahidhussain.store'
      dns01:
        webhook:
          config:
            apiKeySecretRef:
              name: godaddy-api-key
              key: key
            apiSecretSecretRef:
              name: godaddy-api-key
              key: secret
            production: true
            ttl: 600
          groupName: acme.mycompany.com
          solverName: godaddy
EOF

```

Apply it:

```bash
kubectl apply -f cluster-issuer.yaml
```

## 6. Update your Ingress

Update your existing Ingress or create a new one. Here's an example:

```bash
cat <<EOF > ingress-config.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: guestbook-ui-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  tls:
  - hosts:
    - test.mujahidhussain.store
    secretName: guestbook-ui-tls
  rules:
  - host: test.mujahidhussain.store
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: guestbook-ui
            port: 
              number: 80
EOF
```

Apply this Ingress:

```bash
kubectl apply -f ingress-config.yaml
```

## 7. Verify the Certificate
```bash
cat <<EOF > certificate-config.yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: guestbook-ui-cert
spec:
  secretName: guestbook-ui-tls
  renewBefore: 240h
  dnsNames:
  - 'test.mujahidhussain.store'
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  duration: 2160h # 90 days
  privateKey:
    rotationPolicy: Always
EOF

```

Apply it:

```bash
kubectl apply -f certificate-config.yaml

Check the status of your certificate:

```bash
kubectl get certificate
```

When it's ready, you should see "True" under the READY column.

## 8. Update DNS in GoDaddy

Ensure your domain in GoDaddy points to your Ingress Controller's external IP. You can find this IP with:

```bash
kubectl get svc -n ingress-nginx
```

Look for the EXTERNAL-IP of the ingress-nginx-controller service.

## 9. Test Your SSL Setup

Visit https://yourdomain.com in your browser. You should see a valid SSL certificate.
https://test.mujahidhussain.store/
curl -k https://test.mujahidhussain.store/