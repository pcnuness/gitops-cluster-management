# Estratégias de Rollout e Versionamento de Addons

## 📋 Índice

1. [Visão Geral](#visão-geral)
2. [Controle de Versão](#controle-de-versão)
3. [Estratégias de Rollout](#estratégias-de-rollout)
4. [Workflow de Atualização](#workflow-de-atualização)
5. [Rollback](#rollback)
6. [Monitoramento](#monitoramento)

---

## 🎯 Visão Geral

Este documento descreve as estratégias para gerenciar versões e realizar rollouts controlados de addons na plataforma Kubernetes usando GitOps.

### Arquitetura de Versionamento

```
platform-addons-charts (Central)
├── addons/metrics-server/
│   ├── Chart.yaml (version: 0.1.0, appVersion: 3.12.2)
│   └── values.yaml (valores base)
└── Git Tags: v0.1.0, v0.2.0, v1.0.0

gitops-cluster-management (Por Cluster)
├── addons/metrics-server/
│   └── values.yaml (customizações)
└── ApplicationSets
    └── targetRevision: 'main' ou 'v0.x.x'
```

---

## 🔢 Controle de Versão

### 1. Versionamento Semântico

Seguimos [Semantic Versioning 2.0.0](https://semver.org/):

- **MAJOR** (v1.0.0): Mudanças incompatíveis
- **MINOR** (v0.1.0): Novas funcionalidades compatíveis
- **PATCH** (v0.0.1): Correções de bugs

### 2. Três Níveis de Versão

#### a) Chart Version (Nosso Controle)
```yaml
# platform-addons-charts/addons/metrics-server/Chart.yaml
version: 0.2.0  # Versão do nosso wrapper
```

#### b) App Version (Upstream)
```yaml
appVersion: "3.13.0"  # Versão da aplicação real
```

#### c) Dependency Version (Upstream Chart)
```yaml
dependencies:
  - name: metrics-server
    version: 3.13.0  # Versão do chart upstream
```

### 3. Git Tags

```bash
# Criar tag após atualização
git tag -a v0.2.0 -m "feat(metrics-server): upgrade to 3.13.0"
git push origin v0.2.0

# Listar tags
git tag -l "v*"

# Ver detalhes da tag
git show v0.2.0
```

---

## 🚀 Estratégias de Rollout

### 1. Rolling Update (Padrão)

**Quando usar**: Atualizações normais, baixo risco

**Configuração**:
```yaml
# gitops-cluster-management/addons/metrics-server/values.yaml
metrics-server:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1  # Máximo de pods indisponíveis
      maxSurge: 1        # Máximo de pods extras durante update
```

**Fluxo**:
1. Novo pod é criado
2. Aguarda pod ficar Ready
3. Remove pod antigo
4. Repete até todos os pods serem atualizados

**Vantagens**:
- Zero downtime
- Automático
- Rollback fácil

**Desvantagens**:
- Duas versões rodando simultaneamente
- Pode causar inconsistências temporárias

---

### 2. Blue-Green Deployment

**Quando usar**: Atualizações críticas, necessidade de validação completa

**Configuração**:
```yaml
# gitops-cluster-management/addons/metrics-server/values.yaml

# BLUE (versão atual)
metrics-server:
  fullnameOverride: metrics-server-blue
  replicas: 2
  service:
    name: metrics-server  # Service principal aponta aqui

# GREEN (nova versão) - criar novo ApplicationSet temporário
metrics-server:
  fullnameOverride: metrics-server-green
  replicas: 2
  service:
    name: metrics-server-green  # Service de teste
```

**Fluxo**:
1. Deploy completo da versão GREEN
2. Validar GREEN (testes, smoke tests)
3. Mudar Service principal para GREEN
4. Monitorar por período (1-24h)
5. Remover BLUE se tudo OK

**Vantagens**:
- Rollback instantâneo (mudar Service)
- Validação completa antes de ativar
- Zero downtime

**Desvantagens**:
- Dobro de recursos temporariamente
- Mais complexo de gerenciar

**Exemplo de Switch**:
```bash
# Mudar Service para GREEN
kubectl patch svc metrics-server -n kube-system -p '{"spec":{"selector":{"app.kubernetes.io/name":"metrics-server-green"}}}'

# Rollback para BLUE
kubectl patch svc metrics-server -n kube-system -p '{"spec":{"selector":{"app.kubernetes.io/name":"metrics-server-blue"}}}'
```

---

### 3. Canary Deployment

**Quando usar**: Atualizações de alto risco, necessidade de validação gradual

**Configuração**:
```yaml
# ApplicationSet com múltiplas instâncias

# 90% - Versão estável (v0.1.0)
- name: addon-oss-metrics-server-stable
  sources:
    - targetRevision: 'v0.1.0'
  replicas: 9

# 10% - Versão canary (v0.2.0)
- name: addon-oss-metrics-server-canary
  sources:
    - targetRevision: 'v0.2.0'
  replicas: 1
```

**Fluxo**:
1. Deploy 10% canary
2. Monitorar métricas (error rate, latency)
3. Se OK, aumentar para 25%
4. Se OK, aumentar para 50%
5. Se OK, aumentar para 100%
6. Remover versão antiga

**Vantagens**:
- Risco minimizado
- Detecta problemas cedo
- Impacto limitado

**Desvantagens**:
- Processo lento
- Requer monitoramento ativo
- Complexo de automatizar

**Exemplo com Argo Rollouts**:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: metrics-server
spec:
  strategy:
    canary:
      steps:
        - setWeight: 10
        - pause: {duration: 1h}
        - setWeight: 25
        - pause: {duration: 1h}
        - setWeight: 50
        - pause: {duration: 2h}
        - setWeight: 100
```

---

### 4. Recreate (Downtime Aceitável)

**Quando usar**: Addons não críticos, incompatibilidade de versões

**Configuração**:
```yaml
metrics-server:
  updateStrategy:
    type: Recreate
```

**Fluxo**:
1. Remove todos os pods antigos
2. Aguarda remoção completa
3. Cria novos pods

**Vantagens**:
- Simples
- Garante apenas uma versão rodando

**Desvantagens**:
- Downtime durante atualização

---

## 📊 Workflow de Atualização Completo

### Cenário: Atualizar Metrics Server 3.12.2 → 3.13.0

#### Fase 1: Preparação (platform-addons-charts)

```bash
cd platform-addons-charts

# 1. Criar branch de feature
git checkout -b feat/metrics-server-3.13.0

# 2. Atualizar Chart.yaml
cat > addons/metrics-server/Chart.yaml <<EOF
apiVersion: v2
name: metrics-server
description: A helm chart for metrics-server
type: application
version: 0.2.0
appVersion: "3.13.0"
dependencies:
  - name: metrics-server
    version: 3.13.0
    repository: https://kubernetes-sigs.github.io/metrics-server/
EOF

# 3. Atualizar values.yaml se necessário
vim addons/metrics-server/values.yaml

# 4. Commit e push
git add .
git commit -m "feat(metrics-server): upgrade to 3.13.0

- Update chart version to 0.2.0
- Update metrics-server to 3.13.0
- Breaking changes: none
- New features: improved memory efficiency"

git push origin feat/metrics-server-3.13.0

# 5. Criar PR e revisar
# 6. Merge para main
# 7. Criar tag
git checkout main
git pull
git tag -a v0.2.0 -m "Release v0.2.0: Metrics Server 3.13.0"
git push origin v0.2.0
```

#### Fase 2: Deploy em Develop (Automático)

```yaml
# gitops-cluster-management ApplicationSet já usa 'main'
# ArgoCD detecta mudança automaticamente
sources:
  - repoURL: 'https://github.com/pcnuness/platform-addons-charts.git'
    targetRevision: 'main'  # Develop sempre usa main
```

**Validação**:
```bash
# Verificar versão deployada
kubectl get deploy metrics-server -n kube-system -o jsonpath='{.spec.template.spec.containers[0].image}'

# Verificar pods
kubectl get pods -n kube-system -l app.kubernetes.io/name=metrics-server

# Verificar logs
kubectl logs -n kube-system -l app.kubernetes.io/name=metrics-server --tail=100

# Verificar métricas
kubectl top nodes
kubectl top pods -A
```

**Período de Observação**: 24-48h

#### Fase 3: Promover para UAT

```bash
cd gitops-cluster-management

# 1. Criar branch
git checkout -b promote/metrics-server-v0.2.0-uat

# 2. Atualizar ApplicationSet para UAT
# Criar ApplicationSet específico ou usar cluster labels
cat > bootstraps/control-plane/addons/oss/appset-metrics-server.yaml <<EOF
# ... (mesmo conteúdo, mas com targetRevision diferente por ambiente)
generators:
  - clusters:
      values:
        addonChart: metrics-server
        addonChartNamespace: kube-system
        # Versão por ambiente
        chartVersion: '{{metadata.annotations.metrics-server-version}}'
      selector:
        matchExpressions:
          - key: environment
            operator: In
            values: ["develop", "uat"]
EOF

# 3. Atualizar cluster UAT com annotation
# No cluster UAT, adicionar:
# metadata.annotations.metrics-server-version: "v0.2.0"

# 4. Commit e push
git add .
git commit -m "chore(uat): promote metrics-server to v0.2.0"
git push origin promote/metrics-server-v0.2.0-uat

# 5. Criar PR e merge
```

**Período de Observação**: 1 semana

#### Fase 4: Promover para Prod

```bash
# Mesmo processo, mas para prod
# Usar targetRevision: 'v0.2.0' fixo
```

**Período de Observação**: Monitoramento contínuo

---

## 🔄 Rollback

### 1. Rollback via Git

```bash
# Reverter commit
git revert <commit-hash>
git push

# Ou voltar para tag anterior
cd gitops-cluster-management
# Atualizar targetRevision de 'v0.2.0' para 'v0.1.0'
git add .
git commit -m "rollback(metrics-server): revert to v0.1.0"
git push
```

### 2. Rollback via ArgoCD CLI

```bash
# Ver histórico
argocd app history addon-oss-metrics-server-data-plataform-dev-eks

# Rollback para revisão específica
argocd app rollback addon-oss-metrics-server-data-plataform-dev-eks <revision-id>
```

### 3. Rollback via kubectl

```bash
# Ver histórico de deployment
kubectl rollout history deployment/metrics-server -n kube-system

# Rollback para revisão anterior
kubectl rollout undo deployment/metrics-server -n kube-system

# Rollback para revisão específica
kubectl rollout undo deployment/metrics-server -n kube-system --to-revision=2
```

---

## 📈 Monitoramento

### 1. Métricas Chave

```yaml
# ServiceMonitor para Prometheus
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: metrics-server
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: metrics-server
  endpoints:
    - port: https
      interval: 30s
```

**Métricas para observar**:
- `up{job="metrics-server"}` - Disponibilidade
- `process_resident_memory_bytes` - Uso de memória
- `process_cpu_seconds_total` - Uso de CPU
- `rest_client_requests_total` - Requisições API
- `rest_client_request_duration_seconds` - Latência

### 2. Alertas

```yaml
# PrometheusRule
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: metrics-server-alerts
spec:
  groups:
    - name: metrics-server
      interval: 30s
      rules:
        - alert: MetricsServerDown
          expr: up{job="metrics-server"} == 0
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "Metrics Server is down"
            
        - alert: MetricsServerHighMemory
          expr: process_resident_memory_bytes{job="metrics-server"} > 500000000
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "Metrics Server high memory usage"
```

### 3. Dashboard Grafana

```json
{
  "dashboard": {
    "title": "Metrics Server - Rollout Monitoring",
    "panels": [
      {
        "title": "Pod Restarts",
        "targets": [
          {
            "expr": "rate(kube_pod_container_status_restarts_total{pod=~\"metrics-server.*\"}[5m])"
          }
        ]
      },
      {
        "title": "Memory Usage",
        "targets": [
          {
            "expr": "container_memory_usage_bytes{pod=~\"metrics-server.*\"}"
          }
        ]
      },
      {
        "title": "Request Latency",
        "targets": [
          {
            "expr": "histogram_quantile(0.99, rate(rest_client_request_duration_seconds_bucket[5m]))"
          }
        ]
      }
    ]
  }
}
```

---

## 🎯 Checklist de Rollout

### Antes do Rollout

- [ ] Changelog documentado
- [ ] Breaking changes identificadas
- [ ] Testes locais realizados
- [ ] Backup de configurações atuais
- [ ] Plano de rollback definido
- [ ] Janela de manutenção agendada (se necessário)
- [ ] Stakeholders notificados

### Durante o Rollout

- [ ] Deploy em develop primeiro
- [ ] Monitorar logs em tempo real
- [ ] Verificar métricas de saúde
- [ ] Validar funcionalidades críticas
- [ ] Documentar anomalias

### Após o Rollout

- [ ] Validação completa de funcionalidades
- [ ] Métricas estáveis por 24h
- [ ] Sem alertas críticos
- [ ] Performance dentro do esperado
- [ ] Documentação atualizada
- [ ] Stakeholders notificados de sucesso

---

## 📚 Referências

- [ArgoCD Best Practices](https://argo-cd.readthedocs.io/en/stable/user-guide/best_practices/)
- [Kubernetes Deployment Strategies](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#strategy)
- [Semantic Versioning](https://semver.org/)
- [GitOps Principles](https://opengitops.dev/)
