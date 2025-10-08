# Arquitetura GitOps - Cluster Management

Este documento apresenta a arquitetura completa do repositório `gitops-cluster-management` e sua integração com o repositório central `platform-addons-charts`.

## 📊 Visão Geral da Arquitetura

```mermaid
graph TB
    subgraph "GitHub Repositories"
        REPO1[platform-addons-charts<br/>Repositório Central]
        REPO2[gitops-cluster-management<br/>Repositório do Cluster]
    end
    
    subgraph "ArgoCD"
        BOOTSTRAP[bootstrap-gitops-cluster-management<br/>Application Root]
        APPSET_CTRL[ApplicationSet Controller]
        APP_CTRL[Application Controller]
    end
    
    subgraph "Kubernetes Cluster"
        direction TB
        NS_ARGOCD[Namespace: argocd]
        NS_CROSSPLANE[Namespace: crossplane-system]
        NS_OBSERVABILITY[Namespace: observability]
        NS_KUBE_SYSTEM[Namespace: kube-system]
        
        subgraph NS_CROSSPLANE
            CROSSPLANE_POD[Crossplane Pods]
        end
        
        subgraph NS_OBSERVABILITY
            GRAFANA_POD[Grafana Pods]
        end
        
        subgraph NS_KUBE_SYSTEM
            METRICS_POD[Metrics Server Pods]
        end
    end
    
    REPO1 -->|1. Bootstrap reads| BOOTSTRAP
    REPO2 -->|1. Bootstrap reads| BOOTSTRAP
    BOOTSTRAP -->|2. Creates| APPSET_CTRL
    APPSET_CTRL -->|3. Generates| APP_CTRL
    APP_CTRL -->|4. Deploys| NS_CROSSPLANE
    APP_CTRL -->|4. Deploys| NS_OBSERVABILITY
    APP_CTRL -->|4. Deploys| NS_KUBE_SYSTEM
    
    style REPO1 fill:#e1f5ff
    style REPO2 fill:#fff4e6
    style BOOTSTRAP fill:#f3e5f5
    style APPSET_CTRL fill:#e8f5e9
    style APP_CTRL fill:#e8f5e9
```

## 🏗️ Estrutura de Diretórios

```mermaid
graph LR
    subgraph "gitops-cluster-management"
        ROOT[/]
        
        subgraph "addons/"
            ADDONS_CROSS[crossplane/values.yaml]
            ADDONS_GRAF[grafana/values.yaml]
            ADDONS_METRICS[metrics-server/values.yaml]
        end
        
        subgraph "bootstraps/"
            BOOT_ROOT[gitops-root.yaml]
            
            subgraph "cluster-init/"
                INIT_PROJECTS[app-argocd-projects.yaml]
                INIT_ADDONS[app-controle-plane-addons.yaml]
            end
            
            subgraph "control-plane/"
                subgraph "argocd-config/"
                    PROJ[app-projects.yaml]
                end
                
                subgraph "addons/oss/"
                    APPSET_CROSS[appset-crossplane.yaml]
                    APPSET_GRAF[appset-grafana.yaml]
                    APPSET_METRICS[appset-metrics-server.yaml]
                end
            end
        end
        
        subgraph "iac/"
            IAC_S3[s3-bucket/]
            IAC_SECRETS[secrets-manager/]
        end
    end
    
    ROOT --> addons/
    ROOT --> bootstraps/
    ROOT --> iac/
    
    style ROOT fill:#f9f9f9
    style addons/ fill:#fff4e6
    style bootstraps/ fill:#e1f5ff
    style iac/ fill:#f3e5f5
```

## 🔄 Fluxo de Deployment

```mermaid
sequenceDiagram
    participant User as Developer
    participant Git as GitHub
    participant ArgoCD as ArgoCD
    participant K8s as Kubernetes Cluster
    
    User->>Git: 1. Push changes to gitops-cluster-management
    Git->>ArgoCD: 2. Webhook/Poll detects changes
    ArgoCD->>Git: 3. Fetch ApplicationSet manifests
    ArgoCD->>ArgoCD: 4. Generate Applications from AppSets
    ArgoCD->>Git: 5. Fetch Helm Charts from platform-addons-charts
    ArgoCD->>Git: 6. Fetch custom values from gitops-cluster-management
    ArgoCD->>ArgoCD: 7. Merge values (default → env → cluster)
    ArgoCD->>K8s: 8. Apply manifests to cluster
    K8s->>ArgoCD: 9. Report health status
    ArgoCD->>User: 10. Sync status notification
```

## 📦 ApplicationSet Generator Pattern

```mermaid
graph TB
    subgraph "ApplicationSet Generator"
        APPSET[ApplicationSet<br/>addons-oss-metrics-server]
        
        subgraph "Generator: Clusters"
            SELECTOR[Selector<br/>environment: develop]
            CLUSTER1[Cluster: data-plataform-dev-eks<br/>label: environment=develop]
        end
        
        subgraph "Template"
            TEMPLATE[Template<br/>addon-oss-metrics-server-{{name}}]
        end
    end
    
    subgraph "Generated Applications"
        APP1[Application<br/>addon-oss-metrics-server-data-plataform-dev-eks]
    end
    
    APPSET -->|Reads| SELECTOR
    SELECTOR -->|Matches| CLUSTER1
    CLUSTER1 -->|Generates| TEMPLATE
    TEMPLATE -->|Creates| APP1
    
    style APPSET fill:#e8f5e9
    style SELECTOR fill:#fff3e0
    style CLUSTER1 fill:#e1f5ff
    style TEMPLATE fill:#f3e5f5
    style APP1 fill:#e8f5e9
```

## 🔗 Multi-Source Values Merging

```mermaid
graph LR
    subgraph "Sources"
        SOURCE1[Source 1<br/>platform-addons-charts<br/>ref: values-central]
        SOURCE2[Source 2<br/>gitops-cluster-management<br/>ref: values-cluster]
        SOURCE3[Source 3<br/>Chart Path<br/>environments/develop/addons/metrics-server]
    end
    
    subgraph "Value Files Merging"
        direction TB
        VAL1[default/values.yaml<br/>Priority: 1]
        VAL2[develop/values.yaml<br/>Priority: 2]
        VAL3[cluster/values.yaml<br/>Priority: 3 HIGHEST]
    end
    
    subgraph "Result"
        MERGED[Merged Values<br/>Applied to Cluster]
    end
    
    SOURCE1 -->|Provides| VAL1
    SOURCE1 -->|Provides| VAL2
    SOURCE2 -->|Provides| VAL3
    SOURCE3 -->|Uses| VAL1
    SOURCE3 -->|Uses| VAL2
    SOURCE3 -->|Uses| VAL3
    
    VAL1 --> MERGED
    VAL2 --> MERGED
    VAL3 --> MERGED
    
    style SOURCE1 fill:#e1f5ff
    style SOURCE2 fill:#fff4e6
    style SOURCE3 fill:#f3e5f5
    style VAL3 fill:#c8e6c9
    style MERGED fill:#ffeb3b
```

## 🎯 Bootstrap Flow

```mermaid
graph TD
    START[Start: Apply gitops-root.yaml]
    
    subgraph "Bootstrap Phase"
        ROOT[bootstrap-gitops-cluster-management]
        INIT[cluster-init/]
        PROJECTS[app-argocd-projects.yaml]
        ADDONS_BOOT[app-controle-plane-addons.yaml]
    end
    
    subgraph "Control Plane Phase"
        CTRL[control-plane/]
        ARGOCD_CFG[argocd-config/]
        ADDONS_DIR[addons/oss/]
    end
    
    subgraph "ApplicationSets"
        APPSET1[appset-crossplane.yaml]
        APPSET2[appset-grafana.yaml]
        APPSET3[appset-metrics-server.yaml]
    end
    
    subgraph "Applications Generated"
        APP1[addon-oss-crossplane-*]
        APP2[addon-oss-grafana-*]
        APP3[addon-oss-metrics-server-*]
    end
    
    START --> ROOT
    ROOT --> INIT
    INIT --> PROJECTS
    INIT --> ADDONS_BOOT
    ADDONS_BOOT --> CTRL
    CTRL --> ARGOCD_CFG
    CTRL --> ADDONS_DIR
    ADDONS_DIR --> APPSET1
    ADDONS_DIR --> APPSET2
    ADDONS_DIR --> APPSET3
    APPSET1 --> APP1
    APPSET2 --> APP2
    APPSET3 --> APP3
    
    style START fill:#4caf50
    style ROOT fill:#2196f3
    style APPSET1 fill:#ff9800
    style APPSET2 fill:#ff9800
    style APPSET3 fill:#ff9800
    style APP1 fill:#9c27b0
    style APP2 fill:#9c27b0
    style APP3 fill:#9c27b0
```

## 🔐 Multi-Tenancy & Environment Isolation

```mermaid
graph TB
    subgraph "Clusters"
        CLUSTER_DEV[Cluster: develop<br/>label: environment=develop]
        CLUSTER_UAT[Cluster: uat<br/>label: environment=uat]
        CLUSTER_PROD[Cluster: prod<br/>label: environment=prod]
    end
    
    subgraph "ApplicationSets"
        APPSET[ApplicationSet<br/>Selector: environment In [develop,uat,prod]]
    end
    
    subgraph "Platform Addons Charts"
        ENV_DEV[environments/develop/addons/]
        ENV_UAT[environments/uat/addons/]
        ENV_PROD[environments/prod/addons/]
    end
    
    subgraph "Cluster Overrides"
        OVERRIDE[gitops-cluster-management<br/>addons/]
    end
    
    APPSET -->|Matches| CLUSTER_DEV
    APPSET -->|Matches| CLUSTER_UAT
    APPSET -->|Matches| CLUSTER_PROD
    
    CLUSTER_DEV -->|Uses| ENV_DEV
    CLUSTER_UAT -->|Uses| ENV_UAT
    CLUSTER_PROD -->|Uses| ENV_PROD
    
    CLUSTER_DEV -->|Overrides| OVERRIDE
    CLUSTER_UAT -->|Overrides| OVERRIDE
    CLUSTER_PROD -->|Overrides| OVERRIDE
    
    style CLUSTER_DEV fill:#81c784
    style CLUSTER_UAT fill:#ffb74d
    style CLUSTER_PROD fill:#e57373
    style APPSET fill:#64b5f6
    style OVERRIDE fill:#fff176
```

## 📊 Addon Lifecycle

```mermaid
stateDiagram-v2
    [*] --> Defined: Create ApplicationSet
    Defined --> Pending: Cluster matches selector
    Pending --> Syncing: ArgoCD detects Application
    Syncing --> Fetching: Fetch sources
    Fetching --> Merging: Merge value files
    Merging --> Rendering: Render Helm templates
    Rendering --> Applying: Apply to cluster
    Applying --> Healthy: All resources healthy
    Healthy --> Synced: Sync complete
    
    Synced --> Syncing: Configuration change
    Syncing --> OutOfSync: Drift detected
    OutOfSync --> Syncing: Auto-heal enabled
    
    Synced --> Deleting: ApplicationSet deleted
    Deleting --> [*]
    
    Applying --> Degraded: Health check failed
    Degraded --> Syncing: Retry/Fix
```

## 🔄 GitOps Workflow

```mermaid
graph LR
    subgraph "Development"
        DEV[Developer]
        LOCAL[Local Changes]
    end
    
    subgraph "Git Repositories"
        PR[Pull Request]
        MAIN[Main Branch]
    end
    
    subgraph "CI/CD"
        VALIDATE[Validate YAML]
        TEST[Test Manifests]
    end
    
    subgraph "ArgoCD"
        DETECT[Detect Changes]
        SYNC[Sync Application]
    end
    
    subgraph "Kubernetes"
        APPLY[Apply Resources]
        MONITOR[Monitor Health]
    end
    
    DEV --> LOCAL
    LOCAL --> PR
    PR --> VALIDATE
    VALIDATE --> TEST
    TEST --> MAIN
    MAIN --> DETECT
    DETECT --> SYNC
    SYNC --> APPLY
    APPLY --> MONITOR
    MONITOR -->|Healthy| DEV
    MONITOR -->|Degraded| DEV
    
    style DEV fill:#4caf50
    style MAIN fill:#2196f3
    style SYNC fill:#ff9800
    style APPLY fill:#9c27b0
    style MONITOR fill:#f44336
```

## 🎯 Componentes Principais

### 1. **Bootstrap Layer**
- **gitops-root.yaml**: Application raiz que inicializa todo o cluster
- **cluster-init/**: Configurações iniciais (projetos ArgoCD, addons base)

### 2. **Control Plane Layer**
- **argocd-config/**: Configurações do ArgoCD (projetos, RBAC)
- **addons/oss/**: ApplicationSets para addons open source

### 3. **Addons Layer**
- **addons/**: Valores customizados específicos do cluster
- Sobrescreve valores do repositório central

### 4. **IaC Layer**
- **iac/**: Recursos de infraestrutura (S3, Secrets Manager)
- Gerenciados via Crossplane

## 🔑 Conceitos Chave

### Multi-Source Strategy
- **Source 1**: Repositório central (valores default e por ambiente)
- **Source 2**: Repositório do cluster (valores customizados)
- **Source 3**: Chart Helm (templates e manifests)

### Value Precedence
1. Default values (menor precedência)
2. Environment values
3. Cluster overrides (maior precedência)

### Cluster Selection
- ApplicationSets usam `selector.matchExpressions`
- Clusters devem ter label `environment`
- Valores: `develop`, `uat`, `prod`

## 📝 Referências

- [ArgoCD ApplicationSets](https://argo-cd.readthedocs.io/en/stable/user-guide/application-set/)
- [Helm Multi-Source](https://argo-cd.readthedocs.io/en/stable/user-guide/multiple_sources/)
- [GitOps Principles](https://opengitops.dev/)
