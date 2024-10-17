# Argo CD Setup and Usage Guide

## 1. Install Argo CD

First, create a namespace for Argo CD:

```bash
kubectl create namespace argocd
```

Then, apply the Argo CD installation manifest:

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Wait for all Argo CD components to be ready:

```bash
kubectl wait --for=condition=Available deployment --all -n argocd
```

## 2. Access the Argo CD API Server

By default, the Argo CD API server is not exposed. To access it, you can either port-forward or create an ingress for that you need a domain so try to create service by using below yaml file:



jj
`argocd-loadbalancer.yaml`:
```yaml

apiVersion: v1
kind: Service
metadata:
  name: argocd-server-loadbalancer
  namespace: argocd
  annotations:
    # If using an on-premises cluster, you might need to specify the type of load balancer
    # service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
spec:
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 8080
      name: http
    - port: 443
      targetPort: 8080
      name: https
  selector:
    app.kubernetes.io/name: argocd-server

```

```bash

kubectl apply -f argocd-loadbalancer.yaml

```
The API server can now be accessed at https://localhost:8080

## 3. Log in to Argo CD

The initial password for the `admin` account is auto-generated. Retrieve it with:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

Use this password to log in to the Argo CD UI or CLI.

## 4. Install Argo CD CLI (optional but recommended)

Download the Argo CD CLI from the [official GitHub releases page](https://github.com/argoproj/argo-cd/releases) and add it to your PATH.

## 5. Create a Sample Application

Let's create a simple "guestbook" application to deploy. Create a new GitHub repository and add the following files:

`deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: guestbook-ui
spec:
  replicas: 1
  revisionHistoryLimit: 3
  selector:
    matchLabels:
      app: guestbook-ui
  template:
    metadata:
      labels:
        app: guestbook-ui
    spec:
      containers:
      - image: gcr.io/heptio-images/ks-guestbook-demo:0.2
        name: guestbook-ui
        ports:
        - containerPort: 80
```

`service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: guestbook-ui
spec:
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: guestbook-ui
```

## 6. Create an Argo CD Application

Now, let's create an Argo CD Application that references your GitHub repository. You can do this via the UI or CLI. Here's how to do it via CLI:

```bash
argocd app create guestbook \
  --repo https://github.com/PureLogic-Material/-Sample-APP-WITH-ARGO-CD.git \
  --path . \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default
```

Replace `https://github.com/yourusername/yourrepo.git` with your actual repository URL.

## 7. Sync the Application

After creating the application, you need to sync it to deploy:

```bash
argocd app sync guestbook
```

## 8. View the Application

You can now view your application in the Argo CD UI or use the CLI:

```bash
argocd app get guestbook
```

## 9. Make Changes

To see Argo CD in action, make a change to your deployment (e.g., increase the number of replicas) and commit it to your Git repository. Argo CD will detect the change and show that the application is out of sync. You can then either manually sync or set up auto-sync for automatic deployment.

This guide provides a basic setup. Argo CD offers many more features like RBAC, SSO integration, and advanced deployment strategies that you can explore as you become more familiar with the tool.
