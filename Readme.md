# EKS NGINX Deployment with ArgoCD & Ingress
This project provisions an Amazon EKS cluster using Terraform, deploys an NGINX application using Kubernetes manifests, and manages the deployment via ArgoCD. Optionally, it exposes the app via LoadBalancer or Ingress + custom domain.

## ✅ Prerequisites

- AWS CLI configured (`aws configure`)
- `kubectl` installed
- `terraform` installed
- `helm` installed
- GitHub account (for ArgoCD app source)

## Provision EKS Cluster using Terraform
```bash
cd eks-cluster/
terraform init
terraform apply -auto-approve
After provisioning:
aws eks --region <your-region> update-kubeconfig --name demo-eks-cluster
kubectl get nodes

## Deploy NGINX Application
Option A: kubectl apply
kubectl apply -f k8-manifests/deploy.yml
kubectl apply -f k8-manifests/svc.yml

Option B: Sync via ArgoCD(refer below argocd setup)

## Access NGINX
--> Port Forward (Local Access)
kubectl port-forward svc/nginx-service 8080:80
## Visit: http://localhost:8080

--> LoadBalancer (Public IP)
Change service.yaml:
type: LoadBalancer

kubectl apply -f k8-manifests/svc.yml
kubectl get svc nginx-service
## Access: http://<EXTERNAL-IP>

## Install ArgoCD on EKS
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

## Access ArgoCD
kubectl port-forward svc/argocd-server -n argocd 8080:443
Open: https://localhost:8080

## Default Credentials
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d
Username: admin
Password: (output above)

## Create ArgoCD Application (to auto-sync NGINX)
--> Update and apply:

apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/Mukesh-97/nginx-eks-app.git
    targetRevision: HEAD
    path: .
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true

Apply:
kubectl apply -f nginx-argocd.yml


## Install NGINX Ingress Controller
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace

## Create Ingress Resource

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-service
            port:
              number: 80

kubectl apply -f ing.yml
## if want we can Update your domain DNS to point to the Ingress Controller EXTERNAL-IP.

## Cleanup
terraform destroy -auto-approve
kubectl delete namespace argocd ingress-nginx