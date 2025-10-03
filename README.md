Kubernetes Cluster Setup
🖥️ Hosts
Host	Role	CPU	MEM	IP	DNS
CKA-01	cp	4	4	192.168.15.102	192.168.15.1
CKA-02	work	4	12	192.168.15.103	192.168.15.1
🔀 Arquitetura do Cluster
flowchart LR
    U[Users/Clients] --> |HTTP/HTTPS| I[Ingress NGINX]
    I --> |Regras de Roteamento| S[Services (LoadBalancer/ClusterIP)]
    S --> |IP Pool| M[MetalLB]
    M --> |Distribui IP externo| N[Nodes]
    N --> P[Pods/Applications]

1️⃣ Preparação dos Nodes
sudo apt update -y
sudo apt upgrade -y
sudo vim /etc/fstab   # comentar linha do swap
sudo systemctl stop ufw
sudo systemctl disable ufw
sudo systemctl mask ufw

2️⃣ Deploy do MetalLB
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.15.2/config/manifests/metallb-native.yaml

apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.15.240-192.168.15.250
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: l2adv
  namespace: metallb-system

3️⃣ Cert-Manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.18.2/cert-manager.yaml

4️⃣ Ingress-NGINX
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.13.3/deploy/static/provider/cloud/deploy.yaml

5️⃣ Instalar Helm
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh

6️⃣ Rancher
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
kubectl create namespace cattle-system

helm install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --set hostname=rancher.mvps.com.br \
  --set bootstrapPassword=admin


Editar ingress e adicionar:

ingressClassName: nginx

7️⃣ ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d


Ingress:

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server-ingress
  namespace: argocd
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
spec:
  ingressClassName: nginx
  rules:
  - host: argocd.mvps.com.br
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: argocd-server
            port:
              name: https
  tls:
  - hosts:
    - argocd.mvps.com.br
    secretName: argocd-server-tls
