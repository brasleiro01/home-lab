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

O `servicemonitor.yaml` atual já funciona (referencia o Secret
`uptime-kuma-metrics-auth`, já criado manualmente pelo usuário). Em vez de
usar o mecanismo nativo `serviceMonitor.enabled` do chart (que espera um
Secret com outro nome, `uptime-kuma-metrics-basic-auth`, e outro
`selector.matchLabels`), o spec mantém o `ServiceMonitor` **exatamente como
está hoje**, também como template raw dentro do wrapper chart — zero risco de
quebrar o scrape do Prometheus que já está funcionando.

## Estrutura de arquivos

```
argocd/uptime-kuma/
  Chart.yaml
  values.yaml
  Chart.lock                          <- gerado por helm dependency build
  charts/uptime-kuma-4.1.0.tgz        <- vendorizado
  templates/
    namespace-and-pvc.yaml            <- Namespace + PVC, idênticos aos de hoje
    servicemonitor.yaml               <- ServiceMonitor, idêntico ao de hoje
argocd/apps/uptime-kuma.yml           <- Application existente, só ganha helm.valueFiles
```

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
- `serviceMonitor.enabled: false` (mantém o ServiceMonitor raw existente, ver
  risco 4)
- Probes: mantém o default do chart (inclui o binário `extra/healthcheck` do
  próprio Kuma para liveness, com delay de 180s) — não replica os probes
  simplificados (`httpGet` em `/`) do manifesto raw atual, por serem menos
  corretos que o healthcheck oficial do projeto.

### `templates/namespace-and-pvc.yaml`

Cópia exata do `Namespace` e `PersistentVolumeClaim` do
`argocd/uptime-kuma/uptime-kuma.yaml` atual (nome `uptime-kuma-data`,
namespace `uptime-kuma`, `ReadWriteOnce`, `storageClassName: local-path`,
`2Gi`).

### `templates/servicemonitor.yaml`

Cópia exata do `argocd/uptime-kuma/servicemonitor.yaml` atual.

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
2. Merge do PR `homolog` → `main`.
3. Acompanhar o sync no ArgoCD — o Deployment vai falhar por seletor
   imutável (risco 2).
4. Rodar `kubectl delete deployment uptime-kuma -n uptime-kuma`.
5. ArgoCD recria o Deployment com a imagem `2.3.0`; acompanhar os logs do pod
   novo para confirmar que a migração do banco terminou sem erro
   (`kubectl logs -f -n uptime-kuma deploy/uptime-kuma`).
6. Confirmar acesso via `https://uptime.mvps.com.br` e que o scrape do
   Prometheus continua funcionando (o ServiceMonitor não muda).

## Fora de escopo

- Automatizar o backup ou o corte a partir desta sessão — sem acesso ao
  cluster real do homelab.
- Trocar o mecanismo do ServiceMonitor para o nativo do chart — mantém o que
  já funciona.
- Qualquer alteração em `homelab/projeto/` (o esqueleto GitOps criado
  anteriormente nesta mesma conversa, para fins de aprendizado — não é
  afetado por este trabalho no repositório real).
