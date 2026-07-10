# Uptime Kuma no homelab (k3s) — chart wrapper vendorizado

Uptime Kuma **2.3.0** via Argo CD, no mesmo modelo das outras apps deste repo
(`argocd/rancher`, `argocd/observability/*`): **chart Helm vendorizado no
repo** (`charts/uptime-kuma-4.1.0.tgz`) + `values.yaml`. O `Namespace`, o
`PersistentVolumeClaim` e o `ServiceMonitor` ficam como templates raw dentro
do próprio wrapper (não vêm da subchart) — o `ServiceMonitor` referencia o
Secret gerado pelo `ExternalSecret sm-mt-observability` (fora do git), não o
mecanismo nativo `serviceMonitor.enabled` do chart, que fica desligado
(`false`) porque criaria um Secret a partir de valor literal em
`values.yaml`, e este repositório é público.

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

## ServiceMonitor: referencia o ExternalSecret sm-mt-observability

O `ServiceMonitor` é um manifesto raw à mão (`templates/servicemonitor.yaml`),
não gerado pelo mecanismo nativo do chart. Isso garante que não precisamos
colocar credenciais em texto plano no `values.yaml` — um risco inaceitável
num repositório público. Em vez disso, o manifesto referencia um Secret
existente gerado pelo `ExternalSecret` named `sm-mt-observability` (gerenciado
fora do git via External Secrets Operator). O `selector.matchLabels` usa
`app.kubernetes.io/name: uptime-kuma` / `app.kubernetes.io/instance: uptime-kuma`
(rótulos padrão do chart Helm) e é mantido sincronizado manualmente com os
rótulos reais do `Service` gerado.

**Nota importante:** os nomes exatos das chaves dentro do Secret
(`username`/`password` ou outros) precisam ser confirmados/ajustados antes
do corte — o manifesto usa `username`/`password` como placeholder.

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
2. **Confirme que o ExternalSecret `sm-mt-observability` já existe** na
   namespace `uptime-kuma` e tem produzido seu Secret:
   ```bash
   kubectl get secret sm-mt-observability -n uptime-kuma
   ```
   Se o Secret existir, confirme os nomes de chave dentro dele (ex.:
   `kubectl get secret sm-mt-observability -n uptime-kuma -o jsonpath='{.data}' | jq keys`).
   Se os nomes forem diferentes de `username`/`password`, edite
   `templates/servicemonitor.yaml` antes do merge pra ajustar as chaves.
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
   métrica equivalente) — agora via o Secret `sm-mt-observability`.

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
