# Estrutura de Sources nos ApplicationSets

O documento explica a estrutura correta dos `sources` nos ApplicationSets para combinar valores de múltiplos repositórios.

## 📋 Índice

1. [Arquitetura Atual](#arquitetura-atual)
2. [Estrutura de Sources](#estrutura-de-sources)
3. [Como Funciona](#como-funciona)
4. [Precedência de Valores](#precedência-de-valores)
5. [Troubleshooting](#troubleshooting)

---

## 🏗️ Arquitetura Atual

### Repositórios

#### `platform-addons-charts` (Central)
```
platform-addons-charts/
└── addons/
    ├── metrics-server/
    │   ├── Chart.yaml          # Chart wrapper com dependencies
    │   └── values.yaml         # Valores padrão base
    ├── grafana/
    │   ├── Chart.yaml
    │   └── values.yaml
    └── crossplane/
        ├── Chart.yaml
        └── values.yaml
```

#### `gitops-cluster-management` (Por Cluster)
```
gitops-cluster-management/
└── addons/
    ├── metrics-server/
    │   └── values.yaml         # Customizações específicas do cluster
    ├── grafana/
    │   └── values.yaml
    └── crossplane/
        └── values.yaml
```

---

## **Estrutura Correta dos Sources (3 Sources)**

```yaml
sources:
  # Source 1: Repositório central - valores base
  - repoURL: 'https://github.com/pcnuness/platform-addons-charts.git'
    targetRevision: 'main'  # ou 'v0.x.x' para prod
    ref: values-central  # ← Permite usar $values-central/...
  
  # Source 2: Repositório do cluster - customizações
  - repoURL: 'https://github.com/pcnuness/gitops-cluster-management.git'
    targetRevision: 'main'
    ref: values-cluster  # ← Permite usar $values-cluster/...
  
  # Source 3: Chart principal (centralizado)
  - repoURL: 'https://github.com/pcnuness/platform-addons-charts.git'
    targetRevision: 'main'  # Controle de versão aqui
    path: addons/{{values.addonChart}}
    helm:
      valueFiles:
        # Ordem de precedência: último sobrescreve os anteriores
        - $values-central/addons/{{values.addonChart}}/values.yaml
        - $values-cluster/addons/{{values.addonChart}}/values.yaml
      ignoreMissingValueFiles: true
```

## 🏗️ **Como Funciona**

### **Source 1: Referência Central (platform-addons-charts)**
```yaml
- repoURL: 'https://github.com/pcnuness/platform-addons-charts.git'
  targetRevision: 'main'
  ref: values-central  # ← Cria namespace $values-central/
```
- Permite acessar arquivos do repositório central usando `$values-central/...`
- Contém valores padrão e específicos por ambiente

### **Source 2: Referência do Cluster (gitops-cluster-management)**
```yaml
- repoURL: 'https://github.com/pcnuness/gitops-cluster-management.git'
  targetRevision: 'main'
  ref: values-cluster  # ← Cria namespace $values-cluster/
```
- Permite acessar arquivos do repositório do cluster usando `$values-cluster/...`
- Contém valores customizados específicos do cluster

### **Source 3: Chart Principal**
```yaml
- repoURL: 'https://github.com/pcnuness/platform-addons-charts.git'
  targetRevision: 'main'
  path: environments/{{metadata.labels.environment}}/addons/{{values.addonChart}}
  helm:
    valueFiles:
      - $values-central/environments/{{metadata.labels.environment}}/addons/{{values.addonChart}}/values.yaml
      - $values-cluster/addons/{{values.addonChart}}/values.yaml
```
- Contém o Chart Helm (Chart.yaml + templates)
- Combina valores de ambos os repositórios

## 📁 **Estrutura de Arquivos Necessária**

```
platform-addons-charts/ (Repositório Central)
└── environments/
    ├── develop/addons/metrics-server/
    │   ├── Chart.yaml           # ← Chart principal (obrigatório)
    │   └── values.yaml          # ← Valores específicos do ambiente
    └── production/addons/metrics-server/
        ├── Chart.yaml           # ← Chart principal (obrigatório)
        └── values.yaml          # ← Valores específicos do ambiente

gitops-cluster-management/ (Repositório do Cluster)
└── addons/metrics-server/
    └── values.yaml              # ← Valores customizados do cluster
```

## 🔄 **Ordem de Precedência dos Values**

Os `valueFiles` são aplicados na seguinte ordem (último vence):

### **1. Valores Padrão (Base)**
```yaml
- $values-central/environments/default/addons/{{values.addonChart}}/values.yaml
```
- Valores base/padrão do repositório central
- Configurações comuns a todos os ambientes
- Exemplo: `platform-addons-charts/environments/default/addons/metrics-server/values.yaml`

### **2. Valores do Ambiente**
```yaml
- $values-central/environments/{{metadata.labels.environment}}/addons/{{values.addonChart}}/values.yaml
```
- Valores específicos do ambiente (develop, uat, prod)
- Sobrescreve valores padrão
- Exemplo: `platform-addons-charts/environments/develop/addons/metrics-server/values.yaml`

### **3. Valores Customizados do Cluster**
```yaml
- $values-cluster/addons/{{values.addonChart}}/values.yaml
```
- Valores personalizados do cluster
- **Maior precedência** - sobrescreve todos os anteriores
- Exemplo: `gitops-cluster-management/addons/metrics-server/values.yaml`

## 🚫 **Estrutura Incorreta (Evitar)**

### **❌ ERRO 1: Tentar usar valueFiles sem ref separado**
```yaml
sources:
  - repoURL: 'https://github.com/pcnuness/platform-addons-charts.git'
    targetRevision: 'main'
    path: environments/develop/addons/metrics-server
    ref: base-chart  # ← ERRO: ref dentro do chart não funciona para valueFiles
    helm:
      valueFiles:
        - $base-chart/values.yaml  # ← Não vai funcionar
```

### **❌ ERRO 2: Source sem Chart.yaml**
```yaml
sources:
  - repoURL: 'https://github.com/pcnuness/gitops-cluster-management.git'
    path: addons/metrics-server  # ← ERRO: não tem Chart.yaml
```

### **❌ ERRO 3: Um único ref para dois repositórios**
```yaml
sources:
  - repoURL: 'https://github.com/pcnuness/platform-addons-charts.git'
    ref: values  # ← Só acessa platform-addons-charts
  - repoURL: 'https://github.com/pcnuness/platform-addons-charts.git'
    path: environments/develop/addons/metrics-server
    helm:
      valueFiles:
        - $values/addons/metrics-server/values.yaml  # ← ERRO: está no outro repo
```

## ✅ **Estrutura Correta (Usar)**

```yaml
# ✅ CORRETO - Múltiplos refs para múltiplos repositórios
sources:
  # Ref para repo central
  - repoURL: 'https://github.com/pcnuness/platform-addons-charts.git'
    targetRevision: 'main'
    ref: values-central
  
  # Ref para repo do cluster
  - repoURL: 'https://github.com/pcnuness/gitops-cluster-management.git'
    targetRevision: 'main'
    ref: values-cluster
  
  # Chart principal com valueFiles de ambos os repos
  - repoURL: 'https://github.com/pcnuness/platform-addons-charts.git'
    targetRevision: 'main'
    path: environments/{{metadata.labels.environment}}/addons/{{values.addonChart}}
    helm:
      valueFiles:
        - $values-central/environments/default/addons/{{values.addonChart}}/values.yaml
        - $values-central/environments/{{metadata.labels.environment}}/addons/{{values.addonChart}}/values.yaml
        - $values-cluster/addons/{{values.addonChart}}/values.yaml
      ignoreMissingValueFiles: true
```

## 🔧 **Troubleshooting**

### **Erro: "Chart.yaml not found"**
- ✅ Verifique se o `path` aponta para um diretório com `Chart.yaml`
- ✅ O arquivo deve ser `Chart.yaml` (não `Charts.yaml`)
- ✅ Use `valueFiles` para combinar valores, não múltiplos charts

### **Erro: "Value file not found"**
- ✅ Verifique se o arquivo existe no repositório
- ✅ Use `ignoreMissingValueFiles: true` para arquivos opcionais
- ✅ Confirme se os `ref` estão configurados corretamente

### **Erro: "Values não são mesclados"**
- ✅ Verifique se você tem sources separados com `ref` únicos
- ✅ O `ref` deve estar em uma source SEM `path` ou COM `path` mas SEM `helm`
- ✅ Use `$ref-name/caminho/arquivo.yaml` nos valueFiles

### **Erro: "Template rendering failed"**
- ✅ Verifique se as variáveis `{{values.addonChart}}` e `{{metadata.labels.environment}}` estão definidas
- ✅ Confirme se o cluster tem as labels necessárias
- ✅ Verifique se o nome do `ref` não tem caracteres especiais (evite hífens)

## 📋 **Exemplo Completo de ApplicationSet**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: addons-oss-metrics-server
  namespace: argocd
spec:
  generators:
    - clusters:
        selector:
          matchExpressions:
            - key: environment
              operator: In
              values: ["develop", "uat", "prod"]
        values:
          addonChart: metrics-server
          addonChartNamespace: kube-system
  template:
    metadata:
      name: addon-oss-{{values.addonChart}}-{{name}}
    spec:
      project: cluster-management
      sources:
        # Repo central (platform-addons-charts)
        - repoURL: 'https://github.com/pcnuness/platform-addons-charts.git'
          targetRevision: 'main'
          ref: values-central
        
        # Repo do cluster (gitops-cluster-management)
        - repoURL: 'https://github.com/pcnuness/gitops-cluster-management.git'
          targetRevision: 'main'
          ref: values-cluster
        
        # Chart principal
        - repoURL: 'https://github.com/pcnuness/platform-addons-charts.git'
          targetRevision: 'main'
          path: environments/{{metadata.labels.environment}}/addons/{{values.addonChart}}
          helm:
            valueFiles:
              - $values-central/environments/default/addons/{{values.addonChart}}/values.yaml
              - $values-central/environments/{{metadata.labels.environment}}/addons/{{values.addonChart}}/values.yaml
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
          - ServerSideApply=true
```

## 📝 **Checklist de Verificação**

Antes de aplicar o ApplicationSet, verifique:

- [ ] O Chart principal está em `platform-addons-charts/environments/<env>/addons/<addon>/Chart.yaml`
- [ ] Valores padrão estão em `platform-addons-charts/environments/default/addons/<addon>/values.yaml`
- [ ] Valores do ambiente estão em `platform-addons-charts/environments/<env>/addons/<addon>/values.yaml`
- [ ] Valores customizados estão em `gitops-cluster-management/addons/<addon>/values.yaml`
- [ ] Source 1 tem `ref: values-central` (sem path)
- [ ] Source 2 tem `ref: values-cluster` (sem path)
- [ ] Source 3 tem o `path` do chart e `helm.valueFiles` usando os refs corretos
- [ ] O cluster tem a label `environment` com valor correto

## 🎯 **Resultado Esperado**

Com esta estrutura, os valores serão mesclados na seguinte ordem:

1. **Base** → `platform-addons-charts/environments/default/addons/metrics-server/values.yaml`
2. **Ambiente** → `platform-addons-charts/environments/develop/addons/metrics-server/values.yaml`
3. **Cluster** → `gitops-cluster-management/addons/metrics-server/values.yaml` ✅ (maior precedência)

O resultado final no cluster refletirá todos os valores mesclados, com os valores do cluster sobrescrevendo os demais.
