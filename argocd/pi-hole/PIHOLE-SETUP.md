# Pi-hole no k3s — Documentação de Setup

## Visão geral da arquitetura

```
Clientes Windows (DNS: 192.168.15.201)
         │
         │ porta 53 TCP/UDP
         ▼
MetalLB LoadBalancer → 192.168.15.201 (IP fixo, pihole-pool)
         │
         ▼
Pod Pi-hole (namespace: pihole)
         │
         │ upstream DNS
         ▼
    1.1.1.1 / 8.8.8.8
```

O CoreDNS interno do k3s permanece **separado** — resolve `.cluster.local` e não passa pelo Pi-hole.

---

## Pré-requisitos

- k3s instalado e funcionando
- ArgoCD instalado no cluster
- Repositório Git conectado ao ArgoCD (App of Apps pattern)

---

## Componentes criados

### 1. MetalLB — Load Balancer para expor o DNS na porta 53

O Pi-hole precisa expor a porta 53 (TCP + UDP) na rede local com um IP fixo. `NodePort` não funciona para isso porque o Kubernetes não consegue alocar a mesma porta para TCP e UDP simultaneamente via iptables.

**Por que MetalLB?**
- Atribui um IP real da rede local ao Service (não uma porta alta como 30053)
- Clientes DNS padrão sempre usam porta 53 — não há como mudar isso
- O IP persiste mesmo se o pod reiniciar

#### ArgoCD Application — instalação via Helm

Arquivo: `argocd/apps/metallb.yml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: metallb
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "0"
spec:
  project: default
  source:
    repoURL: https://metallb.github.io/metallb
    chart: metallb
    targetRevision: "0.16.0"
    helm:
      values: |
        frr-k8s:
          prometheus:
            serviceMonitor:
              enabled: false
  destination:
    server: https://kubernetes.default.svc
    namespace: metallb-system
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
  ignoreDifferences:
    - group: apiextensions.k8s.io
      kind: CustomResourceDefinition
      jsonPointers:
        - /spec/conversion/webhook/clientConfig/caBundle
    - group: admissionregistration.k8s.io
      kind: ValidatingWebhookConfiguration
      jsonPointers:
        - /webhooks/0/clientConfig/caBundle
```

**Decisões de design:**
- `sync-wave: "0"` — MetalLB deve subir antes da configuração de pools (wave 1)
- `ServerSideApply` **não** é usado — conflita com `--force` do ArgoCD ao atualizar CRDs
- `ignoreDifferences` nos campos `caBundle` — o MetalLB injeta esses valores no webhook após o deploy; sem isso o ArgoCD ficaria em loop de OutOfSync tentando "corrigir" esses campos
- `frr-k8s.prometheus.serviceMonitor.enabled: false` — workaround para um bug no chart 0.16.0 onde o template `service-monitor.yaml` causa nil pointer se o valor não for declarado explicitamente

#### ArgoCD Application — configuração de pools

Arquivo: `argocd/apps/metallb-config.yml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: metallb-config
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "1"
spec:
  project: default
  source:
    repoURL: https://github.com/brasleiro01/home-lab/
    targetRevision: HEAD
    path: argocd/metallb
  destination:
    server: https://kubernetes.default.svc
    namespace: metallb-system
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

**Por que dois Applications separados para o MetalLB?**

Os CRDs `IPAddressPool` e `L2Advertisement` só existem **depois** que o Helm chart do MetalLB for instalado. Se os dois estivessem no mesmo Application, o ArgoCD tentaria criar os pools antes dos CRDs existirem e falharia. A separação por `sync-wave` garante a ordem correta:

```
Wave 0: metallb (Helm) → instala CRDs + controller + speaker
Wave 1: metallb-config  → aplica IPAddressPool + L2Advertisement
```

#### Pools de IP

Arquivo: `argocd/metallb/ipaddresspool.yaml`

```yaml
# Pool geral — para outros serviços LoadBalancer
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-pool
  namespace: metallb-system
spec:
  addresses:
    - 192.168.15.202-192.168.15.210
---
# Pool dedicado ao Pi-hole — IP fixo e isolado
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: pihole-pool
  namespace: metallb-system
spec:
  addresses:
    - 192.168.15.201/32
```

| IP | Uso |
|---|---|
| `192.168.15.201` | Pi-hole DNS (fixo, pool dedicado) |
| `192.168.15.202–210` | Pool geral para outros LoadBalancer services |

> **Importante:** garantir que esses IPs estejam **fora do range do DHCP do roteador** para evitar conflitos de endereço.

#### L2 Advertisement

Arquivo: `argocd/metallb/l2advertisement.yaml`

```yaml
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: l2-advert
  namespace: metallb-system
spec:
  ipAddressPools:
    - default-pool
    - pihole-pool
```

O modo L2 é usado por ser universalmente compatível em redes Ethernet domésticas, sem necessidade de configuração especial no roteador (ao contrário do modo BGP).

---

### 2. Pi-hole — Deployment

Arquivo: `argocd/pi-hole/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pihole
  namespace: pihole
spec:
  replicas: 1
  selector:
    matchLabels:
      app: pihole
  template:
    metadata:
      labels:
        app: pihole
        name: pihole
    spec:
      containers:
        - name: pihole
          image: pihole/pihole:latest
          imagePullPolicy: Always
          env:
            - name: TZ
              value: "America/Sao_Paulo"
            - name: WEBPASSWORD
              value: "admin123"
          volumeMounts:
            - name: etc-pihole-volume
              mountPath: "/etc/pihole"
            - name: etc-dnsmasq-volume
              mountPath: "/etc/dnsmasq.d"
      volumes:
        - name: etc-pihole-volume
          hostPath:
            path: /data/pihole/etc-pihole
            type: DirectoryOrCreate
        - name: etc-dnsmasq-volume
          hostPath:
            path: /data/pihole/dnsmasq.d
            type: DirectoryOrCreate
```

> **Pendências de segurança no deployment:**
> - `WEBPASSWORD` hardcoded — migrar para um Kubernetes Secret
> - `image: latest` — fixar em uma versão específica para builds reproduzíveis

### 3. Pi-hole — Service

Arquivo: `argocd/pi-hole/service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: pihole-svc
  namespace: pihole
  labels:
    app: pihole
  annotations:
    metallb.universe.tf/loadBalancerIPs: "192.168.15.201"
    metallb.universe.tf/address-pool: "pihole-pool"
spec:
  type: LoadBalancer
  selector:
    app: pihole
  ports:
    - name: dns-tcp
      port: 53
      targetPort: 53
      protocol: TCP
    - name: dns-udp
      port: 53
      targetPort: 53
      protocol: UDP
    - name: http
      port: 80
      targetPort: 80
      protocol: TCP
```

**Mudanças em relação à configuração original:**

| Campo | Antes | Depois | Motivo |
|---|---|---|---|
| `type` | `NodePort` | `LoadBalancer` | NodePort não funciona para porta 53 TCP+UDP |
| `nodePort` | `30053` | removido | MetalLB usa a porta 53 padrão |
| annotation `loadBalancerIPs` | — | `192.168.15.201` | IP fixo do pool dedicado |

### 4. Pi-hole — Ingress

Arquivo: `argocd/pi-hole/ingress.yaml`

Expõe o painel web em `pihole.mvps.com.br` via Traefik com TLS gerenciado pelo cert-manager.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: pihole-ingress
  namespace: pihole
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: traefik
  rules:
    - host: pihole.mvps.com.br
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: pihole-svc
                port:
                  number: 80
  tls:
    - hosts:
        - pihole.mvps.com.br
      secretName: pihole-tls
```

---

## Configuração dos clientes DNS

### Opção A — Via roteador (recomendado)

Configurar o servidor DNS do DHCP no roteador para `192.168.15.201`. Todos os dispositivos da rede recebem automaticamente.

### Opção B — Estático por máquina Windows

```
Painel de Controle
  → Central de Rede e Compartilhamento
  → Alterar as configurações do adaptador
  → Clique direito no adaptador → Propriedades
  → Protocolo TCP/IP Versão 4 → Propriedades
    DNS preferencial:  192.168.15.201
    DNS alternativo:   1.1.1.1
```

O DNS alternativo (`1.1.1.1`) garante que a internet continue funcionando caso o Pi-hole caia.

---

## Verificação pós-deploy

```bash
# MetalLB pods rodando
kubectl get pods -n metallb-system

# Pools de IP criados
kubectl get ipaddresspool -n metallb-system

# Service do Pi-hole com EXTERNAL-IP = 192.168.15.201
kubectl get svc -n pihole

# Testar resolução DNS pelo Pi-hole
nslookup google.com 192.168.15.201
```

---

## Estrutura de arquivos

```
argocd/
├── apps/
│   ├── metallb.yml          # ArgoCD App: instala MetalLB via Helm (wave 0)
│   └── metallb-config.yml   # ArgoCD App: pools e L2 advertisement (wave 1)
├── metallb/
│   ├── ipaddresspool.yaml   # Pools: pihole (201) e default (202-210)
│   └── l2advertisement.yaml # Anuncia pools via L2 na rede local
└── pi-hole/
    ├── deployment.yaml      # Pod do Pi-hole com volumes hostPath
    ├── service.yaml         # LoadBalancer com IP fixo 192.168.15.201
    ├── ingress.yaml         # Painel web em pihole.mvps.com.br
    └── PIHOLE-SETUP.md      # Este arquivo
```
