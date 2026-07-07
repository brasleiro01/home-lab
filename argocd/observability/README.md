<p align="center">
  <img src="https://raw.githubusercontent.com/cncf/artwork/master/projects/prometheus/icon/color/prometheus-icon-color.svg" height="50" alt="Prometheus"/>
  &nbsp;&nbsp;
  <img src="https://www.vectorlogo.zone/logos/grafana/grafana-icon.svg" height="50" alt="Grafana / Loki / Tempo / Alloy"/>
  &nbsp;&nbsp;
  <img src="https://cdn.simpleicons.org/minio/C72E49" height="44" alt="MinIO"/>
  &nbsp;&nbsp;
  <img src="https://argo-cd.readthedocs.io/en/stable/assets/logo.png" height="46" alt="Argo CD"/>
</p>

# Stack de Observabilidade — homelab (k3s)

Adaptação do projeto corporativo (`projetos/observabilidade`) para o homelab, **sem AWS**:
- **S3 → MinIO** (buckets `loki` e `tempo` self-hosted no cluster);
- **gp3 → `local-path`** (storageClass default do k3s);
- **nginx-internal → Traefik** + **Cloudflare** (Grafana em `grafana.mvps.com.br`);
- **IRSA → chaves estáticas** do MinIO;
- charts **vendorizados** (umbrella + `charts/*.tgz`), **igual ao Rancher** — o Argo CD renderiza local, sem contatar repo Helm remoto.

## Componentes
| App | Chart | Versão | Função |
|---|---|---|---|
| **minio** | `minio/minio` | 5.4.0 | S3 self-hosted; buckets `loki`, `tempo` |
| **kube-prometheus-stack** | `prometheus-community/kube-prometheus-stack` | 87.2.1 | Prometheus + Grafana + Alertmanager + exporters |
| **loki** | `grafana/loki` | 7.0.0 | logs (SingleBinary, backend S3→MinIO) |
| **tempo** | `grafana/tempo` | 1.24.4 | traces (backend S3→MinIO, OTLP 4317/4318) |
| **alloy** | `grafana/alloy` | 1.10.0 | DaemonSet: coleta logs dos pods → Loki |

## Arquitetura
```
 pods ──logs──▶ Alloy ──▶ Loki ──┐
 apps ──OTLP──▶ Tempo ───────────┤ (chunks/índice/traces em S3)
                 │               ▼
                 │            MinIO  (buckets: loki, tempo)   [local-path PVC]
                 │
                 └─ metrics-generator ──remote_write──▶ Prometheus  (service graph + span metrics/RED)
 pods/nodes ──scrape──────────────────────────────────▶ Prometheus
                                                            │
                                                            ▼
                        Grafana  ◀── datasources: Prometheus + Loki + Tempo (correlacionados)
                           │  Ingress Traefik → Cloudflare: grafana.mvps.com.br
```

### APM / correlação trace ↔ log ↔ métrica
- **Tempo `metrics-generator`** processa os spans e gera **service graph** + **span metrics** (RED: rate/errors/duration),
  fazendo **`remote_write`** para o Prometheus (que precisa de `enableRemoteWriteReceiver: true`).
- **Datasources correlacionados** (ConfigMap `grafana-datasources-homelab`):
  - **Loki → `derivedFields`**: extrai o `traceID` da linha de log e cria link **log → trace** (abre no Tempo).
  - **Tempo → `serviceMap`/`nodeGraph`**: monta o service graph a partir das métricas no Prometheus.
  - **Tempo → `tracesToLogsV2`**: de um span pula pros **logs no Loki** casando `namespace`/`pod` + `traceID`.
  - **Tempo → `tracesToMetrics`**: liga o span às métricas RED no Prometheus.
- **Pré-requisito:** as apps precisam **emitir traces OTLP** para `tempo.observability.svc:4317` (gRPC) / `:4318` (HTTP).
  Sem tráfego OTLP, os painéis de trace/serviço ficam vazios (o resto da stack funciona normal).

## Dashboards (ConfigMaps `grafana_dashboard=1`, carregados pelo sidecar)
| Dashboard | uid | O que mostra |
|---|---|---|
| **Kubernetes — Apps & Logs** | `k8s-apps-logs` | deployments, CPU/memória por pod, taxa de logs e logs (Loki) |
| **APM — Traces & Serviços** | `apm-traces` | service graph (nodeGraph), RED por serviço (req/s, erros/s, p95) e tabela de traces (TraceQL) → clique no TraceID abre o trace e os logs correlacionados |

## Deploy (GitOps)
Os Applications estão em `argocd/apps/` com **sync-waves** (minio → loki/tempo/prometheus → alloy):
```bash
kubectl apply -f argocd/apps/minio.yml
kubectl apply -f argocd/apps/loki.yml
kubectl apply -f argocd/apps/tempo.yml
kubectl apply -f argocd/apps/kube-prometheus-stack.yml
kubectl apply -f argocd/apps/alloy.yml
# (ou deixe seu app-of-apps descobrir a pasta argocd/apps/)
```
**Cloudflare:** `grafana.mvps.com.br` **CNAME → mvps.com.br**, Proxied; SSL/TLS **Full** (se der erro, `Flexible`). Port-forward 80/443 → `192.168.15.202` (doc `homelab-k3s`).

## Acesso
- **Grafana:** `https://grafana.mvps.com.br` — usuário `admin` / senha `admin` (troque no values).
- Datasources já vêm configurados (Prometheus default, Loki, Tempo) via sidecar (ConfigMap `grafana-datasources-homelab`).

## Verificação
```bash
kubectl -n observability get pods
kubectl -n observability get pvc                       # local-path
# buckets criados no MinIO?
kubectl -n observability logs job/minio-make-bucket-job 2>/dev/null | tail
# Loki recebendo? (via gateway)
kubectl -n observability get svc loki-gateway minio tempo kube-prometheus-stack-prometheus
```

## Troubleshooting
- **Loki/Tempo em CrashLoop com erro de S3**: o MinIO ainda não subiu / bucket não criado. Confira o
  `minio` (wave 1) e o job de buckets; Loki/Tempo têm `retry` no Argo CD e sobem quando o MinIO estiver ok.
- **`annotations too long` nos CRDs do prometheus-operator**: garantido pelo `ServerSideApply=true` no app.
- **Targets `down` no Prometheus (controller-manager/scheduler/etcd/proxy)**: desligados no values (k3s embute).
- **Grafana sem datasources**: o sidecar carrega ConfigMaps `grafana_datasource=1` no ns `observability`.
- **502/erro no `grafana.mvps.com.br`**: rede (port-forward/Cloudflare), não o Grafana — ver doc `homelab-k3s`.

## Atualizar versões dos charts
```bash
# ajuste a versão no Chart.yaml do componente, então re-vendorize:
helm repo update
helm dependency build argocd/observability/<componente>
git add argocd/observability/<componente> && git commit -m "chore(obs): bump <componente>" && git push
```

## ⚠️ Segurança (repo público)
Credenciais versionadas de **homelab** (MinIO `minioadmin/minioadmin123`, Grafana `admin/admin`) — **troque**
e, se o repo for público, prefira **repo privado** ou **Sealed Secrets/SOPS**. Exposição do Grafana:
considere **Cloudflare Access**.
