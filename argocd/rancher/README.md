<p align="center">
  <img src="https://raw.githubusercontent.com/rancher/ui/master/public/assets/images/logos/welcome-cow.svg" height="54" alt="Rancher"/>
  &nbsp;&nbsp;
  <img src="https://raw.githubusercontent.com/cncf/artwork/master/projects/k3s/icon/color/k3s-icon-color.svg" height="52" alt="k3s"/>
  &nbsp;&nbsp;
  <img src="https://raw.githubusercontent.com/traefik/traefik/master/docs/content/assets/img/traefik.logo.png" height="48" alt="Traefik"/>
  &nbsp;&nbsp;
  <img src="https://argo-cd.readthedocs.io/en/stable/assets/logo.png" height="48" alt="Argo CD"/>
  &nbsp;&nbsp;
  <img src="https://upload.wikimedia.org/wikipedia/commons/9/94/Cloudflare_Logo.svg" height="36" alt="Cloudflare"/>
</p>

# Rancher no homelab (k3s) — 100% GitOps com Argo CD + Traefik + Cloudflare

Rancher Server **2.14.3** instalado **inteiramente por GitOps** (Argo CD), exposto pelo **Traefik**
(ingress do k3s) em `rancher.mvps.com.br` (Cloudflare). **Nenhum passo manual de `kubectl`** — o
Argo CD aplica o chart **e** o Secret de bootstrap (Application **multi-source**).

> Adaptação da versão corporativa (EKS/nginx). **O que mudou pro homelab:**
> | Corporativo (EKS) | Homelab (k3s) |
> |---|---|
> | nginx-internal + **HAProxy** (NLB L4 não mandava `X-Forwarded-Proto`) | **Traefik** já seta `X-Forwarded-Proto` → **sem HAProxy** |
> | Senha via **ESO/AWS Secrets Manager** | **Secret versionado** no repo (`bootstrap-secret.yaml`), aplicado pelo Argo CD |
> | `replicas: 3` + antiAffinity + Karpenter | **`replicas: 1`** (single-node), sem scheduling especial |
> | Chart vendorizado no repo | **Helm remoto** (canal stable), como o metallb |

## Como funciona (multi-source)
O `argocd/apps/rancher.yml` tem **duas sources** num único Application:
1. **Helm** → chart `rancher` 2.14.3 (de `releases.rancher.com/server-charts/stable`) com os values inline.
2. **Git** → `path: argocd/rancher` deste repo → aplica o `bootstrap-secret.yaml` (o `.example` e este README são ignorados pelo Argo CD).

O Secret tem `sync-wave: "-1"` → é aplicado **antes** do Deployment, então o Rancher já sobe com a senha.

```
Browser ─HTTPS─▶ Cloudflare (proxy) ─▶ Roteador (NAT 80/443) ─▶ Traefik (MetalLB 192.168.15.202)
                                                                     │  ingress rancher.mvps.com.br
                                                                     ▼
                                                Rancher (cattle-system, tls=external, :80, 1 réplica)
```

## Pré-requisitos (já no homelab)
- k3s + **Traefik** + **MetalLB** (`192.168.15.202`) + **Argo CD**.
- **Cloudflare**: `rancher.mvps.com.br` **CNAME → mvps.com.br**, **Proxied**; SSL/TLS mode **Full** (se der erro, teste `Flexible`).
- Port-forward do roteador 80/443 → `192.168.15.202` (ver doc `homelab-k3s`).

## Deploy (GitOps — só git)
1. **Defina a senha** em `argocd/rancher/bootstrap-secret.yaml` (troque `TROQUE_ESTA_SENHA`).
2. **Commit + push.**
3. O Argo CD sincroniza o app `rancher` (ou aplique o Application uma vez: `kubectl apply -f argocd/apps/rancher.yml`). Ele instala o chart + o secret.
```bash
kubectl -n cattle-system rollout status deploy/rancher     # 1/1 Running
```
4. Acesse `https://rancher.mvps.com.br` → login `admin` + a senha do secret → **defina a senha definitiva**.

## Verificação
```bash
kubectl -n cattle-system get pods
kubectl get ingress -n cattle-system                 # host rancher.mvps.com.br (classe traefik)
kubectl get clusters.management.cattle.io            # 'local' deve ficar Ready
```

## Troubleshooting
- **Login "0 users" / admin não nasce**: o pod precisa do `CATTLE_BOOTSTRAP_PASSWORD` no 1º boot. O
  `sync-wave: -1` do secret garante isso; se subiu antes, `kubectl -n cattle-system rollout restart deploy/rancher`.
- **Mixed-content**: não deve ocorrer (Traefik + Cloudflare setam o proto). Se ocorrer, cheque o **SSL mode** no Cloudflare.
- **Cluster `local` Unavailable com `Connected=True`** (após restart): condition `Ready` travada →
  `kubectl annotate clusters.management.cattle.io local cattle.io/force-reconcile=$(date +%s) --overwrite`.
- **Error 522**: é rede (NAT/port-forward ou Cloudflare), não o Rancher — ver doc `homelab-k3s`.

## ⚠️ Segurança (repo público)
- O **`bootstrapPassword` é descartável**: só vale no 1º login; depois que você define a senha real
  do admin no Rancher, ele não concede mais acesso. Por isso versioná-lo é baixo risco.
- **Mas os demais secrets do repo** (ex.: `k8s-monitor/04-secrets.yaml` tem chave do Gemini, webhook do
  Discord, senha do Postgres) **ficam expostos se o repo for PÚBLICO.** Recomendo **repo privado** ou
  migrar pra **Sealed Secrets/SOPS** (aí o secret vai criptografado no git e o Argo CD aplica igual).
- Rancher exposto na internet: considere **Cloudflare Access (Zero Trust)** na frente.

## Arquivos
- `../apps/rancher.yml` — Argo CD Application **multi-source** (Helm 2.14.3 + este path).
- `bootstrap-secret.yaml` — Secret da senha de bootstrap (versionado).
- `bootstrap-secret.yaml.example` — modelo.
