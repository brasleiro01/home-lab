# Design — Migração de `argocd/uptime-kuma` para Helm chart wrapper + ServiceMonitor

Data: 2026-07-10

## Contexto

O `argocd/uptime-kuma/` hoje é um manifesto raw (Namespace + PVC + Deployment +
Service + Ingress) aplicado via ArgoCD (`argocd/apps/uptime-kuma.yml`), rodando
`louislam/uptime-kuma:1` em produção real no homelab, com dados reais em um PVC
(`uptime-kuma-data`, `local-path`, 2Gi). Já existe também um
`argocd/uptime-kuma/servicemonitor.yaml`, com instruções para criar
manualmente o Secret `uptime-kuma-metrics-auth` (usuário/API key) usado pelo
scrape do Prometheus (kube-prometheus-stack).

O objetivo é migrar essa app para o mesmo padrão de "chart Helm wrapper
vendorizado" já usado em outras apps deste repositório (`argocd/rancher/`,
`argocd/observability/*`): `Chart.yaml` com uma `dependency` no chart
comunitário `dirsigler/uptime-kuma-helm`, `values.yaml`, chart vendorizado em
`charts/*.tgz`. Ao mesmo tempo, atualizar a imagem de `1` para `2.3.0`
(decisão explícita do usuário — bundlar upgrade de versão com a migração de
infraestrutura).

Esta é uma migração de uma app **real, em produção, com dados reais** — não
uma app nova. Os riscos abaixo vêm de documentação oficial consultada antes
deste design, não de suposição.

## Riscos identificados e mitigação

### 1. Migração de dados v1→v2 do Uptime Kuma

Fonte oficial: https://github.com/louislam/uptime-kuma/wiki/Migration-From-v1-To-v2

- A migração do banco SQLite roda automaticamente no start do container v2,
  mas a documentação recomenda (repetidamente) **parar a instância e fazer
  backup do diretório `/app/data` antes** de atualizar — a feature de
  backup/restore via JSON do próprio Kuma **foi removida na v2**.
- A migração pode levar de minutos a horas dependendo do volume de dados
  monitorados, e **não deve ser interrompida** — se for, a recuperação é
  restaurar o backup e tentar de novo.
- **Mitigação:** este spec documenta um runbook manual de backup (ver seção
  "Runbook de corte") que o usuário deve executar antes de mergear
  `homolog` → `main`. Não há como automatizar isso a partir desta sessão —
  não há acesso direto ao cluster real.

### 2. Seletor de Deployment imutável

O Deployment atual usa `spec.selector.matchLabels: {app: uptime-kuma}`. O
chart novo usa `app.kubernetes.io/name`/`app.kubernetes.io/instance`
(convenção padrão de Helm). `spec.selector` é imutável em Deployments do
Kubernetes — o ArgoCD **não conseguirá aplicar o novo Deployment por cima do
antigo** (mesmo nome, seletor diferente); o sync desse recurso específico vai
falhar com erro de campo imutável.

- **Mitigação:** runbook documenta rodar
  `kubectl delete deployment uptime-kuma -n uptime-kuma` uma vez, manualmente,
  após o merge — o ArgoCD recria o Deployment com o seletor novo no sync
  seguinte. Deletar um Deployment não afeta o PVC (recursos independentes).

### 3. Preservar o PVC existente (não perder monitors/histórico)

- `values.yaml` usa `volume.existingClaim: uptime-kuma-data` — o chart, quando
  `existingClaim` está setado, **não cria um PVC próprio** (confirmado lendo
  `templates/pvc.yaml` do chart vendorizado), só monta o PVC existente pelo
  nome.
- O `Namespace` e o `PersistentVolumeClaim` atuais continuam declarados
  **dentro do próprio wrapper chart** (`templates/namespace-and-pvc.yaml`),
  com o conteúdo idêntico ao manifesto raw de hoje — não são gerados pela
  subchart. Isso evita que o ArgoCD faça `prune` neles quando o manifesto raw
  antigo deixar de existir no git (se esses recursos simplesmente
  desaparecessem do manifesto renderizado, o `syncPolicy.automated.prune: true`
  os deletaria — e deletar um PVC provisionado por `local-path` normalmente
  apaga os dados em disco).

### 4. ServiceMonitor existente

**REVISÃO (2026-07-10, durante a execução):** o desenho original mantinha o
`ServiceMonitor` como template raw preservado dentro do wrapper chart (ver
histórico da Task 2 abaixo). Ao tentar remover o arquivo antigo duplicado da
raiz (`argocd/uptime-kuma/servicemonitor.yaml`), o usuário pediu para trocar
de abordagem: usar o **mecanismo nativo do chart**
(`uptime-kuma.serviceMonitor.enabled: true`) em vez de manter um
`ServiceMonitor` raw à mão. Esta seção documenta o desenho final,
substituindo a versão anterior.

O chart nativo gera o `ServiceMonitor` a partir dos mesmos helpers usados
pelo `Service` (`uptime-kuma.selectorLabels`), então o `selector.matchLabels`
sempre bate automaticamente com o Service real — elimina de vez a classe de
bug do risco original (selector desatualizado). Testado localmente
(`helm template ... --api-versions monitoring.coreos.com/v1`): renderiza
`name: uptime-kuma`, `namespace: uptime-kuma`, `port: http`, `path:
/metrics`, com `interval: 30s` (setado explicitamente em `values.yaml` para
não mudar a frequência de scrape que já existe hoje — o default do chart é
`60s`).

**Consequência operacional:** o mecanismo nativo referencia um Secret com
nome diferente do atual: `uptime-kuma-metrics-basic-auth` (gerado como
`<fullname>-metrics-basic-auth`), em vez do `uptime-kuma-metrics-auth`
manual que já existe e funciona hoje. **O usuário precisa criar esse novo
Secret antes do corte**, copiando o valor do Secret antigo (sem expor a API
key em texto — ver comando exato no runbook). Isso substitui o item
"Trocar o mecanismo do ServiceMonitor... mantém o que já funciona" que
antes estava em "Fora de escopo".

**REVISÃO 2 (2026-07-10, achado na revisão da Task 4 — reverte a Revisão 1):**
a revisão da Task 4 (README) descobriu, extraindo e lendo o chart vendorizado
por completo, um arquivo que não tinha sido inspecionado durante o desenho
original: `templates/servicemonitor.auth.secret.yaml`. Esse template **cria
um Secret de verdade** a partir do valor literal de
`.Values.serviceMonitor.basicAuth` (`stringData` direto dos valores do
`values.yaml`) — **não** é uma referência a um Secret externo já existente,
como eu tinha assumido ao ler só o `servicemonitor.yaml`. Isso significa que
o mecanismo nativo do chart exige colocar a credencial real da API key **em
texto plano no `values.yaml`, commitado no git** — e este repositório
(`brasleiro01/home-lab`) é **público** no GitHub. Colocar a API key real ali
vazaria a credencial publicamente.

Além disso, o nome do Secret que o `ServiceMonitor` nativo referencia é fixo
(`{{ fullname }}-metrics-basic-auth`, sem ponto de override em `values.yaml`)
— não dá pra apontar o mecanismo nativo para um Secret com outro nome.

**Decisão final:** voltar para um `ServiceMonitor` raw (à mão, dentro de
`templates/`, mesma abordagem da Revisão 0 original), mas referenciando um
`ExternalSecret` que o usuário já mantém no cluster
(`sm-mt-observability`, gerenciado fora do git via um External Secrets
Operator) em vez do antigo `uptime-kuma-metrics-auth`. O `values.yaml` volta
a ter `serviceMonitor.enabled: false`. Os nomes exatos das chaves
(`username`/`password` ou outros) dentro do Secret gerado por esse
`ExternalSecret` **não foram confirmados pelo usuário** — o
`templates/servicemonitor.yaml` final usa `username`/`password` como
placeholder, com um comentário pedindo confirmação/ajuste antes do corte.

Como consequência: `templates/servicemonitor.yaml` (criado na Task 2) e o
`argocd/uptime-kuma/servicemonitor.yaml` antigo da raiz são **ambos
removidos** — nenhum ServiceMonitor fica declarado como arquivo à mão, tudo
vem do `values.yaml`.

## Estrutura de arquivos

```
argocd/uptime-kuma/
  Chart.yaml
  values.yaml                         <- inclui uptime-kuma.serviceMonitor.enabled: false
  Chart.lock                          <- gerado por helm dependency build
  charts/uptime-kuma-4.1.0.tgz        <- vendorizado
  templates/
    namespace-and-pvc.yaml            <- Namespace + PVC, idênticos aos de hoje
    servicemonitor.yaml               <- ServiceMonitor raw à mão, referencia o
                                          Secret do ExternalSecret sm-mt-observability
argocd/apps/uptime-kuma.yml           <- Application existente, só ganha helm.valueFiles
```

(`templates/servicemonitor.yaml` existe — ver REVISÃO 2 na seção "Risco 4"
acima: o mecanismo nativo do chart via `serviceMonitor.enabled` foi
descartado por exigir credencial em texto plano em `values.yaml`, que é
público. O ServiceMonitor final é um manifesto raw à mão, com
`serviceMonitor.enabled: false` em `values.yaml`.)

### `Chart.yaml`

`apiVersion: v2`, `name: uptime-kuma-homelab` (segue o padrão observado em
`argocd/rancher` e `argocd/observability/loki`, que sufixam `-homelab`),
`appVersion: "2.3.0"`, `dependencies` apontando pra `uptime-kuma` `4.1.0` em
`https://helm.irsigler.cloud` (mesmo chart comunitário já usado e testado em
`homelab/projeto/apps/kuma`).

### `values.yaml` (chave `uptime-kuma:`)

- `image.tag: "2.3.0"`
- `volume.enabled: true`, `volume.existingClaim: uptime-kuma-data`
- `ingress.enabled: true`, `ingress.className: traefik`, host
  `uptime.mvps.com.br` (mantém o domínio real já configurado no
  Cloudflare/DNS — não troca)
- `resources.requests`: `cpu: 50m, memory: 128Mi`; `resources.limits`:
  `cpu: 500m, memory: 512Mi` (mesmos valores já usados em produção — a
  migração não muda dimensionamento)
- `serviceMonitor.enabled: false` (mecanismo nativo do chart descartado — ver
  REVISÃO 2 no risco 4 acima; o scrape real vem de `templates/servicemonitor.yaml`,
  um manifesto raw à mão referenciando o Secret do `ExternalSecret sm-mt-observability`)
- Probes: mantém o default do chart (inclui o binário `extra/healthcheck` do
  próprio Kuma para liveness, com delay de 180s) — não replica os probes
  simplificados (`httpGet` em `/`) do manifesto raw atual, por serem menos
  corretos que o healthcheck oficial do projeto.
- `strategy.type: Recreate` — o manifesto raw antigo já setava isso
  explicitamente (comentário: "PVC RWO: evita 2 pods disputando o volume");
  o chart Helm não define `strategy` por padrão (usaria `RollingUpdate`), o
  que arriscaria dois pods montando o mesmo PVC `ReadWriteOnce` e escrevendo
  no mesmo arquivo SQLite durante um rollout. Adicionado de volta em
  `values.yaml` na revisão final.

### `templates/namespace-and-pvc.yaml`

Cópia exata do `Namespace` e `PersistentVolumeClaim` do
`argocd/uptime-kuma/uptime-kuma.yaml` atual (nome `uptime-kuma-data`,
namespace `uptime-kuma`, `ReadWriteOnce`, `storageClassName: local-path`,
`2Gi`).

### `argocd/apps/uptime-kuma.yml`

Mesmo `Application` de hoje (`repoURL`, `targetRevision: HEAD`, `path:
argocd/uptime-kuma`, `destination.namespace: uptime-kuma`, `syncPolicy`
inalterados) — só ganha um bloco `helm.valueFiles: [values.yaml]`, já que o
`path` passa a apontar para um chart Helm em vez de manifests puros.

## Runbook de corte (execução manual do usuário, fora desta sessão)

Documentado no README/PR, não executado por esta sessão (sem acesso ao
cluster real):

1. **Backup:** copiar o diretório `/app/data` do pod atual do Uptime Kuma
   para fora do cluster (ex.: `kubectl cp` ou `kubectl exec ... tar`), antes
   de qualquer merge para `main`.
2. **Criar o novo Secret** que o ServiceMonitor nativo do chart espera,
   copiando o valor do Secret antigo (sem expor a API key em texto):
   ```bash
   kubectl -n uptime-kuma get secret uptime-kuma-metrics-auth -o jsonpath='{.data.username}' | base64 -d > /tmp/u
   kubectl -n uptime-kuma get secret uptime-kuma-metrics-auth -o jsonpath='{.data.password}' | base64 -d > /tmp/p
   kubectl -n uptime-kuma create secret generic uptime-kuma-metrics-basic-auth \
     --from-file=username=/tmp/u --from-file=password=/tmp/p
   rm /tmp/u /tmp/p
   ```
   (O Secret antigo `uptime-kuma-metrics-auth` pode ficar — não atrapalha,
   só não é mais referenciado.)
3. Merge do PR `homolog` → `main`.
4. Acompanhar o sync no ArgoCD — o Deployment vai falhar por seletor
   imutável (risco 2).
5. Rodar `kubectl delete deployment uptime-kuma -n uptime-kuma`.
6. ArgoCD recria o Deployment com a imagem `2.3.0`; acompanhar os logs do pod
   novo para confirmar que a migração do banco terminou sem erro
   (`kubectl logs -f -n uptime-kuma deploy/uptime-kuma`).
7. Confirmar acesso via `https://uptime.mvps.com.br` e que o scrape do
   Prometheus continua funcionando (`up{job="uptime-kuma"}` ou métrica
   equivalente) — agora com o ServiceMonitor gerado pelo chart.

## Fora de escopo

- Automatizar o backup, a criação do novo Secret, ou o corte a partir desta
  sessão — sem acesso ao cluster real do homelab, tudo fica documentado como
  runbook manual.
- Deletar o Secret antigo `uptime-kuma-metrics-auth` — fica órfão mas
  inofensivo, remoção é opcional e fica a critério do usuário.
- Qualquer alteração em `homelab/projeto/` (o esqueleto GitOps criado
  anteriormente nesta mesma conversa, para fins de aprendizado — não é
  afetado por este trabalho no repositório real).
