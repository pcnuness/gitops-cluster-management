# Guia de Gerenciamento de Addons GitOps

Este documento explica como usar a estrutura GitOps para gerenciar addons em clusters EKS utilizando ArgoCD.

## Arquitetura

### Repositórios

1. **gitops-cluster-management**: Contém AppSets e valores personalizados específicos do cluster
2. **platform-addons-charts**: Contém Charts Helm centralizados (estrutura simplificada)

### Estrutura de Diretórios

```
gitops-cluster-management/
├── addons/                          # Valores personalizados por cluster
│   ├── crossplane/values.yaml
│   ├── grafana/values.yaml
│   └── metrics-server/values.yaml
└── bootstraps/control-plane/addons/
    └── oss/                         # Addons Open Source
        ├── appset-crossplane.yaml
        ├── appset-grafana.yaml
        └── appset-metrics-server.yaml

platform-addons-charts/
└── addons/                          # Charts centralizados (sem separação por ambiente)
    ├── crossplane/
    │   ├── Chart.yaml              # Não mais Charts.yaml
    │   └── values.yaml             # Valores padrão base
    ├── grafana/
    │   ├── Chart.yaml
    │   └── values.yaml
    └── metrics-server/
        ├── Chart.yaml
        └── values.yaml
```

## Addons Disponíveis

### Open Source Addons (OSS)

#### Crossplane
- **Namespace**: `crossplane-system`
- **Função**: Infrastructure as Code para AWS
- **Configurações**: AWS Provider, Composition Functions
- **Sync Wave**: -1 (primeiro a ser instalado)

#### Grafana
- **Namespace**: `observability`
- **Função**: Dashboards e visualização de métricas
- **Configurações**: Plugins, Data Sources, Ingress
- **Sync Wave**: 0

#### Metrics Server
- **Namespace**: `kube-system`
- **Função**: Coleta de métricas de recursos
- **Configurações**: TLS, tolerations para control-plane
- **Sync Wave**: 1

### AWS Addons

#### EKS Addons
- **Namespace**: `kube-system`
- **Função**: Addons nativos do EKS (CoreDNS, VPC CNI, EBS CSI, etc.)
- **Configurações**: Versões específicas, Service Account Roles
- **Sync Wave**: 0

## Como Adicionar um Novo Addon

### 1. Criar Chart no platform-addons-charts

Crie a estrutura centralizada:

```bash
mkdir -p platform-addons-charts/addons/{addon-name}
```

Crie os arquivos:
- `Chart.yaml`: Metadados do chart e dependências upstream
- `values.yaml`: Valores padrão base aplicáveis a todos os clusters

Exemplo de `Chart.yaml`:
```yaml
apiVersion: v2
name: {addon-name}
description: A helm chart for {addon-name}
type: application
version: 0.1.0
appVersion: "1.0.0"
dependencies:
  - name: {addon-name}
    version: 1.0.0
    repository: https://charts.example.com/
```

### 2. Criar Valores Personalizados no gitops-cluster-management

```bash
mkdir -p gitops-cluster-management/addons/{addon-name}
```

Crie `values.yaml` com configurações específicas do cluster.

### 3. Criar ApplicationSet

Crie um novo AppSet em:
- `gitops-cluster-management/bootstraps/control-plane/addons/oss/appset-{addon-name}.yaml`

Estrutura recomendada com multi-source:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: addons-oss-{addon-name}
  namespace: argocd
spec:
  generators:
    - clusters:
        values:
          addonChart: {addon-name}
          addonChartNamespace: {namespace}
  template:
    metadata:
      name: addon-oss-{{values.addonChart}}-{{name}}
    spec:
      project: cluster-management
      sources:
        # Source 1: Valores base centralizados
        - repoURL: 'https://github.com/pcnuness/platform-addons-charts.git'
          targetRevision: 'main'  # ou 'v0.x.x' para prod
          ref: values-central
        # Source 2: Valores customizados do cluster
        - repoURL: 'https://github.com/pcnuness/gitops-cluster-management.git'
          targetRevision: 'main'
          ref: values-cluster
        # Source 3: Chart principal
        - repoURL: 'https://github.com/pcnuness/platform-addons-charts.git'
          targetRevision: 'main'
          path: addons/{{values.addonChart}}
          helm:
            valueFiles:
              - $values-central/addons/{{values.addonChart}}/values.yaml
              - $values-cluster/addons/{{values.addonChart}}/values.yaml
            ignoreMissingValueFiles: true
      destination:
        namespace: '{{values.addonChartNamespace}}'
        name: '{{name}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```

Ajuste:
- `addonChart`: Nome do addon
- `addonChartNamespace`: Namespace de destino
- `sync-wave` (annotation): Ordem de sincronização

## Configuração por Ambiente

### Desenvolvimento (develop)
- Recursos mínimos
- Configurações de debug habilitadas
- Senhas padrão para desenvolvimento

### UAT
- Recursos moderados
- Configurações de produção simuladas
- Service Monitors habilitados

### Produção (prod)
- Recursos otimizados
- Alta disponibilidade (múltiplas réplicas)
- Configurações de segurança rigorosas

## Fluxo de Deployment

1. **Push** para branch principal do `platform-addons-charts`
2. **ArgoCD** detecta mudanças automaticamente
3. **Sync** automático com validação
4. **Health checks** e rollback automático em caso de falha

## Troubleshooting

### Verificar Status dos Addons

```bash
# Listar Applications do ArgoCD
argocd app list

# Verificar status específico
argocd app get addon-oss-crossplane-{cluster-name}

# Sincronizar manualmente
argocd app sync addon-oss-crossplane-{cluster-name}
```

### Logs e Debugging

```bash
# Logs do ArgoCD ApplicationSet Controller
kubectl logs -n argocd deployment/argocd-applicationset-controller

# Logs específicos do addon
kubectl logs -n {namespace} deployment/{addon-name}
```

## Segurança

### RBAC
- Cada AppSet usa o projeto `cluster-management`
- Políticas de acesso definidas no ArgoCD

### Secrets
- Credenciais AWS gerenciadas via Crossplane
- Secrets do Kubernetes para senhas de admin

### Network Policies
- Isolamento de rede por namespace
- Políticas específicas para cada addon

## Monitoramento

### Métricas
- ServiceMonitors para Prometheus
- Dashboards pré-configurados no Grafana

### Alertas
- Health checks automáticos
- Notificações via Slack/Email

## Manutenção

### Atualizações
- Atualizações de charts via values.yaml
- Testes em staging antes da produção
- Rollback automático em caso de falha

### Backup
- Configurações versionadas no Git
- Backup de dados críticos (Grafana dashboards)

## Exemplos de Uso

### Adicionar um novo addon de monitoramento

1. Criar Chart em `platform-addons-charts/environments/*/addons/prometheus/`
2. Adicionar valores personalizados em `gitops-cluster-management/addons/prometheus/`
3. Criar AppSet em `gitops-cluster-management/bootstraps/control-plane/addons/oss/appset-prometheus.yaml`

### Configurar addon específico da AWS

1. Criar Chart em `platform-addons-charts/environments/*/addons/aws-load-balancer-controller/`
2. Configurar IAM roles e policies
3. Adicionar ao AppSet de AWS addons

Este guia garante que a estrutura GitOps seja consistente, escalável e fácil de manter.
