# GitOps Cluster Management

Este repositório gerencia os addons essenciais para clusters Kubernetes usando GitOps com ArgoCD. A estrutura foi projetada para suportar crescimento de forma escalável e organizacional.

## Estrutura do Repositório

```
gitops-cluster-management/
├── addons/            # Values de addons base por controler           
│   ├── grafana/values.yaml       # Addon Grafana
│   ├── crossplane/values.yaml    # Addon IaC
│   └── mimir/values.yaml         # Addon Colletor Metrics
├── bootstraps/                   # ApplicationSets e configurações do ArgoCD
│   ├── cluster-init/             # Bootstrap inicial do cluster
│   └── control-plane/            # Gerenciamento dos addons
│       └── addons/
|           └── oss/                       # ApplicationSets organizados por categoria
│               ├── appset-crossplane.yaml # Controller Crossplane
│               ├── appset-grafana.yaml    # Controller Grafana
│               └── appset-mimir.yaml      # Controler Mimir
└── README.md # Esta documentação
```

## Princípios de Design

### 1. **Configuração**
Os values.yaml são carregados para cada controller:
1. `addons/{addons}/values.yaml`

## Addons Suportados

### Monitoring Stack
- **Mimir**: Métricas e alertas
- **Grafana**: Dashboards e visualização

### Infrastructure as Code
- **Crossplane**: Provisionamento de recursos AWS

## Como Usar

### 1. Adicionar um Novo Addon

1. Crie a estrutura de diretórios:
   ```bash
   mkdir -p addons/{addon}/values.yaml
   ```

2. Crie os values.yaml para cada addon

3. Crie o ApplicationSet em:
   ```
   bootstraps/control-plane/addons/{categoria}/appset-{addon}.yaml
   ```

## Segurança

### IAM Roles
- Cada cluster deve ter IAM roles configuradas para Crossplane
- Políticas mínimas necessárias

## Monitoramento

### Métricas
- ServiceMonitors para Prometheus
- Dashboards pré-configurados no Grafana

### Logs
- Logs estruturados em JSON
- Integração com CloudWatch (opcional)
- Rotação de logs configurada

## CI/CD Integration

### Fluxo de Deploy
1. **Push** para branch principal
2. **ArgoCD** detecta mudanças automaticamente
3. **Sync** automático com validação
4. **Health checks** e rollback automático

## Manutenção

### Atualizações
- Atualizações de charts via values.yaml
- Testes em staging antes da produção
- Rollback automático em caso de falha

### Backup
- Configurações versionadas no Git
- Backup de dados críticos (Grafana dashboards, Mimir data)
---

**Nota**: Esta estrutura foi projetada para ser escalável e de forma consistente e segura.
