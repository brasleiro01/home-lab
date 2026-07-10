# Migração uptime-kuma para Helm Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Migrar `argocd/uptime-kuma/` de manifests raw (Deployment/Service/Ingress/PVC/Namespace soltos) para um chart Helm wrapper vendorizado (`dirsigler/uptime-kuma-helm` 4.1.0), atualizando a imagem de `louislam/uptime-kuma:1` para `2.3.0`, preservando o PVC de dados existente e corrigindo o ServiceMonitor para continuar funcionando com o novo Service.

**Architecture:** Chart wrapper em `argocd/uptime-kuma/` com uma `dependency` no chart comunitário `uptime-kuma`, mais um `templates/` próprio (fora da subchart) que recria Namespace+PVC e o ServiceMonitor exatamente como já existem hoje — evitando que o ArgoCD faça prune desses recursos com dados reais. O `Application` existente (`argocd/apps/uptime-kuma.yml`) ganha só um bloco `helm.valueFiles`.

**Tech Stack:** Helm v4, ArgoCD (`argoproj.io/v1alpha1` `Application`), Prometheus Operator (`ServiceMonitor` CRD), git.

## Global Constraints

- Repositório: `home-lab` (`https://github.com/brasleiro01/home-lab.git`), branch de trabalho `homolog` (padrão já usado no repo — 34 PRs seguiram esse fluxo homolog→main). Commits vão para `homolog`; **não fazer merge/push para `main` como parte deste plano** — isso é decisão manual do usuário, feita depois, seguindo o runbook.
- **App real, em produção, com dados reais** — não é uma app nova. Qualquer mudança que arrisque o PVC `uptime-kuma-data` (namespace `uptime-kuma`, `local-path`, 2Gi) precisa preservá-lo.
- Chart: `uptime-kuma` versão `4.1.0` (repositório clássico `https://helm.irsigler.cloud`, requer `helm repo add` antes de `helm dependency build` — mesmo chart já usado e testado em `homelab/projeto/apps/kuma`).
- `values.yaml`: `image.tag: "2.3.0"` (upgrade de versão combinado com a migração — decisão explícita do usuário), `volume.existingClaim: uptime-kuma-data` (reaproveita o PVC, não cria um novo), `ingress.className: traefik`, host `uptime.mvps.com.br` (domínio real, não mudar), `resources` idênticos aos atuais (`50m/128Mi` request, `500m/512Mi` limit), `serviceMonitor.enabled: false` (mantém o ServiceMonitor raw, não o nativo do chart).
- `templates/namespace-and-pvc.yaml` e `templates/servicemonitor.yaml` no wrapper chart precisam existir para que o `syncPolicy.automated.prune: true` do Application não delete esses recursos.
- **Correção obrigatória no ServiceMonitor:** o Service gerado pelo chart usa labels `app.kubernetes.io/name: uptime-kuma` / `app.kubernetes.io/instance: uptime-kuma`, não `app: uptime-kuma`. O `spec.selector.matchLabels` do `ServiceMonitor` precisa usar as labels novas, senão o Prometheus para de coletar métricas silenciosamente. O resto do ServiceMonitor (Secret `uptime-kuma-metrics-auth`, porta `http`, path `/metrics`, interval `30s`) fica idêntico ao atual.
- Fora de escopo: executar o corte em produção (backup, merge pra main, `kubectl delete deployment`) — fica documentado como runbook manual para o usuário, esta sessão não tem acesso ao cluster real.
- Fora de escopo: qualquer alteração em `homelab/projeto/` (repositório separado, não relacionado a este trabalho).

---

### Task 1: Chart Helm wrapper (`argocd/uptime-kuma/Chart.yaml` + `values.yaml`)

**Files:**
- Create: `argocd/uptime-kuma/Chart.yaml`
- Create: `argocd/uptime-kuma/values.yaml`
- Create (gerado por `helm dependency build`): `argocd/uptime-kuma/Chart.lock`, `argocd/uptime-kuma/charts/uptime-kuma-4.1.0.tgz`
- Delete: `argocd/uptime-kuma/uptime-kuma.yaml` (o manifesto raw antigo — seu conteúdo é preservado nas Tasks 1 e 2 dentro da nova estrutura)

**Interfaces:**
- Produces: chart Helm válido em `argocd/uptime-kuma`, usado pela Task 2 (templates preservados) e pela Task 3 (`Application`).

- [ ] **Step 1: Registrar o repositório Helm do chart (não é OCI)**

Run: `helm repo add uptime-kuma https://helm.irsigler.cloud`

Expected: `"uptime-kuma" has been added to your repositories` (ou, se já registrado de sessão anterior, `"uptime-kuma" already exists with the same configuration, skipping` — também válido).

- [ ] **Step 2: Criar `argocd/uptime-kuma/Chart.yaml`**

```yaml
apiVersion: v2
name: uptime-kuma-homelab
description: Umbrella chart do Uptime Kuma para o homelab (k3s + Traefik), migrado de manifests raw.
type: application
version: 0.1.0
appVersion: "2.3.0"
dependencies:
  - name: uptime-kuma
    version: "4.1.0"
    repository: "https://helm.irsigler.cloud"
```

- [ ] **Step 3: Criar `argocd/uptime-kuma/values.yaml`**

```yaml
uptime-kuma:
  image:
    tag: "2.3.0"

  volume:
    enabled: true
    existingClaim: uptime-kuma-data

  ingress:
    enabled: true
    className: traefik
    hosts:
      - host: "uptime.mvps.com.br"
        paths:
          - path: /
            pathType: Prefix

  resources:
    requests:
      cpu: 50m
      memory: 128Mi
    limits:
      cpu: 500m
      memory: 512Mi

  serviceMonitor:
    enabled: false
```

- [ ] **Step 4: Deletar o manifesto raw antigo**

Run: `git rm argocd/uptime-kuma/uptime-kuma.yaml`

(Seu conteúdo — Namespace, PVC, Deployment, Service, Ingress — está preservado: Namespace+PVC voltam na Task 2 como template raw; Deployment/Service/Ingress agora vêm da subchart Helm via `values.yaml` acima.)

- [ ] **Step 5: Vendorizar o chart de dependência**

Run: `cd /home/marcus/homelab/home-lab && helm dependency build argocd/uptime-kuma`

Expected (últimas linhas): `Saving 1 charts`, `Downloading uptime-kuma from repo https://helm.irsigler.cloud`, `Deleting outdated charts` — e os arquivos `argocd/uptime-kuma/Chart.lock` e `argocd/uptime-kuma/charts/uptime-kuma-4.1.0.tgz` passam a existir.

- [ ] **Step 6: Commit**

```bash
cd /home/marcus/homelab/home-lab
git add argocd/uptime-kuma/Chart.yaml argocd/uptime-kuma/values.yaml argocd/uptime-kuma/Chart.lock argocd/uptime-kuma/charts/uptime-kuma-4.1.0.tgz
git commit -m "feat(uptime-kuma): migrate to helm chart wrapper, bump to 2.3.0"
```

(O `git rm` do Step 4 fica staged nesse mesmo commit — confirme com `git status` antes de commitar que `argocd/uptime-kuma/uptime-kuma.yaml` aparece como deleted.)

---

### Task 2: Recursos preservados (`templates/namespace-and-pvc.yaml` + `templates/servicemonitor.yaml`)

> **REVISÃO (executada, ver ledger):** esta task foi implementada como escrito
> abaixo e passou na revisão, mas logo em seguida o usuário pediu para trocar
> o ServiceMonitor pelo mecanismo nativo do chart. `templates/servicemonitor.yaml`
> foi removido num commit seguinte (junto com o `servicemonitor.yaml` antigo da
> raiz), e `values.yaml` (Task 1) ganhou `serviceMonitor.enabled: true`. Ver a
> seção "### 4. ServiceMonitor existente" no spec de design para o racional
> completo. O texto abaixo é o desenho original — mantido por registro
> histórico, não reflete o estado final do ServiceMonitor.

**Files:**
- Create: `argocd/uptime-kuma/templates/namespace-and-pvc.yaml`
- Create: `argocd/uptime-kuma/templates/servicemonitor.yaml`

**Interfaces:**
- Consumes: `argocd/uptime-kuma/Chart.yaml` (Task 1) — estes templates vivem dentro do mesmo chart wrapper, fora da subchart `uptime-kuma`.
- Produces: os recursos Namespace/PVC/ServiceMonitor continuam existindo no manifesto renderizado (evita prune pelo ArgoCD).

- [ ] **Step 1: Criar `argocd/uptime-kuma/templates/namespace-and-pvc.yaml`**

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: uptime-kuma
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: uptime-kuma-data
  namespace: uptime-kuma
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: local-path
  resources:
    requests:
      storage: 2Gi
```

- [ ] **Step 2: Criar `argocd/uptime-kuma/templates/servicemonitor.yaml`**

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: uptime-kuma
  namespace: uptime-kuma
  labels:
    app: uptime-kuma
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: uptime-kuma
      app.kubernetes.io/instance: uptime-kuma
  endpoints:
    - port: http
      path: /metrics
      interval: 30s
      basicAuth:
        username:
          name: uptime-kuma-metrics-auth
          key: username
        password:
          name: uptime-kuma-metrics-auth
          key: password
```

(Nota: `spec.selector.matchLabels` usa `app.kubernetes.io/name`/`instance` — **não** `app: uptime-kuma` como no manifesto antigo — porque é isso que o Service gerado pela subchart Helm carrega. `metadata.labels` do próprio ServiceMonitor pode continuar `app: uptime-kuma`, é só um label descritivo do objeto, não afeta o selector.)

- [ ] **Step 3: Lint do chart completo**

Run: `cd /home/marcus/homelab/home-lab && helm lint argocd/uptime-kuma`

Expected (última linha): `1 chart(s) linted, 0 chart(s) failed`

- [ ] **Step 4: Renderizar o chart e conferir todos os recursos**

Run: `cd /home/marcus/homelab/home-lab && helm template uptime-kuma argocd/uptime-kuma --namespace uptime-kuma | grep -E "^kind:" | sort | uniq -c`

Expected:
```
      1 kind: Deployment
      1 kind: Ingress
      1 kind: Namespace
      1 kind: PersistentVolumeClaim
      1 kind: Service
      1 kind: ServiceMonitor
```

(Pode aparecer também `1 kind: Pod` — é o pod de teste `helm.sh/hook: test` que vem por padrão do chart upstream, inerte em deploy normal, não é um problema.)

- [ ] **Step 5: Conferir que o PVC renderizado é o preservado (não um novo da subchart) e que o Deployment monta ele**

Run:
```bash
cd /home/marcus/homelab/home-lab
helm template uptime-kuma argocd/uptime-kuma --namespace uptime-kuma > /tmp/uk-verify.yaml
grep -c "kind: PersistentVolumeClaim" /tmp/uk-verify.yaml
grep -A2 "# Source: uptime-kuma-homelab/templates/namespace-and-pvc.yaml" /tmp/uk-verify.yaml
grep -A2 "claimName:" /tmp/uk-verify.yaml
rm /tmp/uk-verify.yaml
```

Expected: a contagem de `PersistentVolumeClaim` é `1` (só o preservado, a subchart não cria um segundo porque `volume.existingClaim` está setado); o bloco `# Source: .../templates/namespace-and-pvc.yaml` aparece seguido de `apiVersion: v1` / `kind: Namespace`; e `claimName: uptime-kuma-data` aparece no volume do Deployment.

- [ ] **Step 6: Conferir a imagem e o host do ingress renderizados**

Run: `cd /home/marcus/homelab/home-lab && helm template uptime-kuma argocd/uptime-kuma --namespace uptime-kuma | grep -E "image:|host:"`

Expected: deve conter `image: "louislam/uptime-kuma:2.3.0"` e `host: uptime.mvps.com.br` (ou `"uptime.mvps.com.br"`, dependendo de como o template renderiza aspas).

- [ ] **Step 7: Commit**

```bash
cd /home/marcus/homelab/home-lab
git add argocd/uptime-kuma/templates/namespace-and-pvc.yaml argocd/uptime-kuma/templates/servicemonitor.yaml
git commit -m "feat(uptime-kuma): preserve namespace, pvc and servicemonitor as raw templates"
```

---

### Task 3: Atualizar o Application do ArgoCD

**Files:**
- Modify: `argocd/apps/uptime-kuma.yml`

**Interfaces:**
- Consumes: `argocd/uptime-kuma` (path do chart, inalterado — já apontava pra lá).

- [ ] **Step 1: Adicionar o bloco `helm.valueFiles` ao `Application` existente**

Substituir o conteúdo de `argocd/apps/uptime-kuma.yml` por:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: uptime-kuma
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/brasleiro01/home-lab/
    targetRevision: HEAD
    path: argocd/uptime-kuma
    helm:
      valueFiles:
        - values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: uptime-kuma
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

(Única mudança real: o bloco `helm: valueFiles: [values.yaml]` sob `spec.source`. Todo o resto — `repoURL`, `targetRevision: HEAD`, `path`, `destination`, `syncPolicy` — é idêntico ao `Application` que já existe hoje.)

- [ ] **Step 2: Validar sintaxe e campos obrigatórios**

Run:
```bash
cd /home/marcus/homelab/home-lab
python3 -c "
import yaml
with open('argocd/apps/uptime-kuma.yml') as f:
    doc = yaml.safe_load(f)
assert doc['apiVersion'] == 'argoproj.io/v1alpha1'
assert doc['kind'] == 'Application'
assert doc['spec']['source']['path'] == 'argocd/uptime-kuma'
assert doc['spec']['source']['helm']['valueFiles'] == ['values.yaml']
assert doc['spec']['destination']['namespace'] == 'uptime-kuma'
assert doc['spec']['syncPolicy']['automated']['prune'] is True
print('OK')
"
```

Expected: `OK`

- [ ] **Step 3: Commit**

```bash
cd /home/marcus/homelab/home-lab
git add argocd/apps/uptime-kuma.yml
git commit -m "feat(uptime-kuma): point application at helm chart wrapper"
```

---

### Task 4: README com o runbook de corte

**Files:**
- Create: `argocd/uptime-kuma/README.md`

**Interfaces:**
- Consumes: caminhos e comandos definidos nas Tasks 1-3 (referencia o chart, o `Application`, e os passos manuais de corte).

- [ ] **Step 1: Criar `argocd/uptime-kuma/README.md`**

```markdown
# Uptime Kuma no homelab (k3s) — chart wrapper vendorizado

Uptime Kuma **2.3.0** via Argo CD, no mesmo modelo das outras apps deste repo
(`argocd/rancher`, `argocd/observability/*`): **chart Helm vendorizado no
repo** (`charts/uptime-kuma-4.1.0.tgz`) + `values.yaml`. O `Namespace` e o
`PersistentVolumeClaim` ficam como template raw dentro do próprio wrapper
(não vêm da subchart); o `ServiceMonitor` vem do mecanismo nativo do chart
(`uptime-kuma.serviceMonitor.enabled: true` em `values.yaml`).

## Por que Namespace/PVC ficam fora da subchart

Esta app já tinha dados reais (monitors configurados, histórico) num PVC
(`uptime-kuma-data`) antes da migração para Helm. Se esse recurso fosse
gerado pela subchart em vez de mantido explicitamente no manifesto, o
`syncPolicy.automated.prune: true` do Application deletaria o PVC assim que
o manifesto raw antigo saísse do git — e um PVC `local-path` deletado
normalmente apaga os dados em disco. Por isso:
- `values.yaml` usa `volume.existingClaim: uptime-kuma-data` (a subchart
  monta o PVC existente, não cria um novo).
- `templates/namespace-and-pvc.yaml` recria o Namespace e o PVC exatamente
  como existiam no manifesto raw — continuam "vivos" no manifesto
  renderizado, o prune não mexe neles.

## ServiceMonitor: mecanismo nativo do chart, não um arquivo à mão

Diferente do Namespace/PVC, o `ServiceMonitor` vem de
`uptime-kuma.serviceMonitor.enabled: true` no `values.yaml` — o chart gera o
recurso usando os mesmos helpers do `Service`, então o `selector.matchLabels`
sempre bate automaticamente (sem risco de ficar desatualizado se o chart
mudar labels no futuro). `interval: "30s"` é setado explicitamente pra manter
a mesma frequência de scrape que já existia (o default do chart é `60s`).

**Consequência: nome de Secret diferente.** O mecanismo nativo espera um
Secret chamado `uptime-kuma-metrics-basic-auth` (com chaves
`username`/`password`), não o `uptime-kuma-metrics-auth` que já existe hoje.
Isso precisa ser criado manualmente antes do corte — ver runbook abaixo.

## ⚠️ Runbook de corte (execução manual, uma vez, ao mergear pra main)

Esta migração troca a imagem de `louislam/uptime-kuma:1` para `2.3.0` — uma
migração de banco de dados (SQLite) roda automaticamente no primeiro start
do container v2. A [documentação oficial](https://github.com/louislam/uptime-kuma/wiki/Migration-From-v1-To-v2)
recomenda fortemente backup antes, e não é possível interromper a migração
no meio.

1. **Backup do diretório de dados, antes de mergear:**
   ```bash
   kubectl -n uptime-kuma exec deploy/uptime-kuma -- tar czf - -C /app/data . \
     > uptime-kuma-data-backup-$(date +%Y%m%d).tar.gz
   ```
2. **Criar o novo Secret do ServiceMonitor**, copiando o valor do Secret
   antigo (sem expor a API key em texto):
   ```bash
   kubectl -n uptime-kuma get secret uptime-kuma-metrics-auth -o jsonpath='{.data.username}' | base64 -d > /tmp/u
   kubectl -n uptime-kuma get secret uptime-kuma-metrics-auth -o jsonpath='{.data.password}' | base64 -d > /tmp/p
   kubectl -n uptime-kuma create secret generic uptime-kuma-metrics-basic-auth \
     --from-file=username=/tmp/u --from-file=password=/tmp/p
   rm /tmp/u /tmp/p
   ```
   (O Secret antigo `uptime-kuma-metrics-auth` pode ficar — não atrapalha,
   só deixa de ser referenciado. Remover é opcional.)
3. Merge do PR `homolog` → `main`.
4. Acompanhe o sync no Argo CD. **O Deployment vai falhar** com um erro de
   campo imutável (`spec.selector` mudou de `{app: uptime-kuma}` para
   `{app.kubernetes.io/name/instance: uptime-kuma}` — Kubernetes não permite
   trocar o seletor de um Deployment existente).
5. Rode, uma vez:
   ```bash
   kubectl delete deployment uptime-kuma -n uptime-kuma
   ```
   Isso **não afeta o PVC** — só remove o Deployment antigo pra o Argo CD
   poder criar o novo, com o seletor correto.
6. O Argo CD recria o Deployment com a imagem `2.3.0`. Acompanhe os logs pra
   confirmar que a migração do banco terminou sem erro:
   ```bash
   kubectl logs -f -n uptime-kuma deploy/uptime-kuma
   ```
7. Confirme o acesso em `https://uptime.mvps.com.br` e que os monitors e o
   histórico continuam lá.
8. Confirme que o Prometheus voltou a coletar `up{job="uptime-kuma"}` (ou a
   métrica equivalente) — agora via o Secret novo `uptime-kuma-metrics-basic-auth`.

## Validar local (antes do push)

```bash
helm dependency build argocd/uptime-kuma
helm lint argocd/uptime-kuma
helm template uptime-kuma argocd/uptime-kuma --namespace uptime-kuma --api-versions monitoring.coreos.com/v1 | less
```

## Atualizar a versão do Uptime Kuma no futuro

```bash
# ajuste appVersion no Chart.yaml e image.tag no values.yaml, então:
helm dependency build argocd/uptime-kuma   # re-vendoriza se a versão do CHART também mudou
git add argocd/uptime-kuma && git commit -m "chore(uptime-kuma): bump to X.Y.Z"
```
```

- [ ] **Step 2: Conferir que os caminhos citados no README existem de fato**

Run:
```bash
cd /home/marcus/homelab/home-lab
test -f argocd/uptime-kuma/Chart.yaml && \
test -f argocd/uptime-kuma/values.yaml && \
test -f argocd/uptime-kuma/templates/namespace-and-pvc.yaml && \
test -f argocd/apps/uptime-kuma.yml && \
! test -f argocd/uptime-kuma/templates/servicemonitor.yaml && \
! test -f argocd/uptime-kuma/servicemonitor.yaml && \
echo OK
```

Expected: `OK` (as duas últimas checagens confirmam que os arquivos de ServiceMonitor à mão foram removidos — o mecanismo agora é nativo do chart, via `values.yaml`).

- [ ] **Step 3: Commit**

```bash
cd /home/marcus/homelab/home-lab
git add argocd/uptime-kuma/README.md
git commit -m "docs(uptime-kuma): add cutover runbook and migration rationale"
```
