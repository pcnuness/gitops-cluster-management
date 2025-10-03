# Guia de Gerenciamento de Addons GitOps

Este documento explica como usar a estrutura GitOps para gerenciar addons em clusters EKS utilizando ArgoCD.

## Arquitetura

### Repositórios

1. **gitops-cluster-management**: Contém AppSets e valores personalizados específicos do cluster
2. **platform-addons-charts**: Contém Charts Helm centralizados com valores padrão por ambiente

### Estrutura de Diretórios

```
gitops-cluster-management/
├── addons/                          # Valores personalizados por cluster
│   ├── crossplane/values.yaml
│   ├── grafana/values.yaml
│   ├── metrics-server/values.yaml
│   └── eks-addons/values.yaml
└── bootstraps/control-plane/addons/
    ├── oss/                         # Addons Open Source
    │   ├── appset-crossplane.yaml
    │   ├── appset-grafana.yaml
    │   ├── appset-metrics-server.yaml
    │   └── appset-all-oss-addons.yaml
    └── aws/                         # Addons AWS
        ├── appset-eks-addons.yaml
        ├── appset-ebs-csi-resources.yaml
        └── appset-all-aws-addons.yaml

platform-addons-charts/
└── environments/
    ├── default/                     # Valores padrão
    ├── develop/                     # Ambiente de desenvolvimento
    ├── uat/                         # Ambiente de teste
    └── prod/                        # Ambiente de produção
    └── {environment}/addons/
        ├── crossplane/
        │   ├── Charts.yaml
        │   └── values.yaml
        ├── grafana/
        │   ├── Charts.yaml
        │   └── values.yaml
        ├── metrics-server/
        │   ├── Charts.yaml
        │   └── values.yaml
        └── eks-addons/
            ├── Charts.yaml
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

Para cada ambiente, crie a estrutura:

```bash
mkdir -p platform-addons-charts/environments/{environment}/addons/{addon-name}
```

Crie os arquivos:
- `Charts.yaml`: Dependências do Helm
- `values.yaml`: Valores padrão para o ambiente

### 2. Criar Valores Personalizados no gitops-cluster-management

```bash
mkdir -p gitops-cluster-management/addons/{addon-name}
```

Crie `values.yaml` com configurações específicas do cluster.

### 3. Criar ApplicationSet

Crie um novo AppSet em:
- `gitops-cluster-management/bootstraps/control-plane/addons/{categoria}/appset-{addon-name}.yaml`

Use a estrutura base dos AppSets existentes, ajustando:
- `addonChart`: Nome do addon
- `addonChartNamespace`: Namespace de destino
- `sync-wave`: Ordem de sincronização

### 4. Atualizar AppSet Agregador (Opcional)

Se usar AppSets agregadores (`appset-all-*`), adicione o novo addon à lista de `addonCharts`.

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
