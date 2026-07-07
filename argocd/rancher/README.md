<p align="center">
  <img src="https://raw.githubusercontent.com/rancher/ui/master/public/assets/images/logos/welcome-cow.svg" height="54" alt="Rancher"/>
  &nbsp;&nbsp;
  <img src="https://raw.githubusercontent.com/cncf/artwork/master/projects/k3s/icon/color/k3s-icon-color.svg" height="52" alt="k3s"/>
  &nbsp;&nbsp;
  <img src="https://raw.githubusercontent.com/traefik/traefik/master/docs/content/assets/img/traefik.logo.png" height="48" alt="Traefik"/>
  &nbsp;&nbsp;
  <img src="https://argo-cd.readthedocs.io/en/stable/assets/logo.png" height="48" alt="Argo CD"/>
  &nbsp;&nbsp;
  <img src="https://www.vectorlogo.zone/logos/cloudflare/cloudflare-icon.svg" height="46" alt="Cloudflare"/>
</p>

# Rancher no homelab (k3s) — umbrella chart vendorizado (igual ao homolog)

Rancher Server **2.14.3** via **Argo CD**, no mesmo modelo do ambiente corporativo: **umbrella Helm
chart com o chart vendorizado no repo** (`charts/rancher-2.14.3.tgz`) + `values.yaml`, e o
Application apontando pro **path local**. O Argo CD renderiza o chart **sem contatar o repo Helm
remoto** — foi o que resolveu o erro `repository not accessible / NoSuchKey`.

## Estrutura (igual `apps/mt-rancher` do homolog)
```
argocd/rancher/
  ├─ Chart.yaml                 # umbrella (dependency: rancher 2.14.3)
  ├─ Chart.lock                 # trava da dependência
  ├─ charts/rancher-2.14.3.tgz  # chart VENDORIZADO (não baixa do remoto)
  ├─ values.yaml                # config sob a chave "rancher:"
  └─ README.md
argocd/apps/rancher.yml         # Application → path argocd/rancher (helm valueFiles: values.yaml)
```

## Diferenças do homolog (adaptação homelab)
| Homolog (EKS) | Homelab (k3s) |
|---|---|
| nginx-internal + **HAProxy** | **Traefik** já seta `X-Forwarded-Proto` → **sem HAProxy** |
| **ESO/Secrets Manager** | `bootstrapPassword` **no `values.yaml`** (o chart cria o `bootstrap-secret`) |
| `replicas: 3` + antiAffinity + Karpenter | **`replicas: 1`**, sem scheduling especial |
| `ingress.enabled: false` (ingress próprio→HAProxy) | **`ingress.enabled: true`** + `ingressClassName: traefik` |

## Fluxo
```
Browser ─HTTPS─▶ Cloudflare (proxy) ─▶ Roteador (NAT 80/443) ─▶ Traefik (MetalLB 192.168.15.202)
                                                                     │  ingress rancher.mvps.com.br
                                                                     ▼
                                                Rancher (cattle-system, tls=external, :80, 1 réplica)
```

## Pré-requisitos
- k3s + **Traefik** + **MetalLB** (`192.168.15.202`) + **Argo CD**.
- Cloudflare: `rancher.mvps.com.br` **CNAME → mvps.com.br**, **Proxied**; SSL/TLS **Full** (se der erro, `Flexible`).
- Port-forward do roteador 80/443 → `192.168.15.202` (doc `homelab-k3s`).

## Deploy (GitOps)
1. Edite a senha em `argocd/rancher/values.yaml` (`bootstrapPassword`).
2. **Commit + push.**
3. `kubectl apply -f argocd/apps/rancher.yml` (ou o Argo CD já sincroniza).
```bash
kubectl -n cattle-system rollout status deploy/rancher     # 1/1 Running
```
4. Acesse `https://rancher.mvps.com.br` → `admin` + senha do `bootstrapPassword` → defina a definitiva.

## Validar local (antes do push)
```bash
helm template rancher argocd/rancher -f argocd/rancher/values.yaml --kube-version 1.31.0 | head
```

## Atualizar a versão do Rancher
```bash
# ajuste version/appVersion no Chart.yaml e a versão da dependência, então:
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
helm repo update
helm dependency build argocd/rancher      # re-vendoriza o novo .tgz + Chart.lock
git add argocd/rancher && git commit -m "chore(rancher): bump 2.14.x" && git push
```

## Troubleshooting
- **`repository not accessible / NoSuchKey`**: era o Argo CD tratando o repo Helm remoto como git.
  Resolvido **vendorizando o chart** (esta abordagem) — não há mais repo remoto na source.
- **Login "0 users" / admin não nasce**: `bootstrapPassword` precisa estar setado (o chart cria o
  `bootstrap-secret`); se subiu vazio, ajuste e `kubectl -n cattle-system rollout restart deploy/rancher`.
- **Cluster `local` Unavailable com `Connected=True`** (após restart): condition `Ready` travada →
  `kubectl annotate clusters.management.cattle.io local cattle.io/force-reconcile=$(date +%s) --overwrite`.
- **Error 522**: rede (NAT/port-forward ou Cloudflare), não o Rancher — ver doc `homelab-k3s`.

## ⚠️ Segurança
`bootstrapPassword` no git é **descartável** (só o 1º login). Mas se o repo for **público**, prefira
**repo privado** ou **Sealed Secrets/SOPS**. Rancher exposto: considere **Cloudflare Access**.
