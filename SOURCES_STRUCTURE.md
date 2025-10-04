# Estrutura de Sources nos ApplicationSets

Este documento explica a estrutura correta dos `sources` nos ApplicationSets para evitar erros de "Chart.yaml not found".

## 🔍 **Problema Identificado**

O erro `"error reading helm chart from <path>/addons/grafana/Chart.yaml: no such file or directory"` ocorria porque:

1. **Múltiplos Sources Incorretos**: Cada source estava sendo tratado como um chart Helm independente
2. **Chart.yaml Ausente**: O diretório `addons/` contém apenas `values.yaml`, não charts completos
3. **Estrutura de Sources Incorreta**: Não estava usando `valueFiles` corretamente

## ✅ **Solução Implementada**

### **Estrutura Correta dos Sources**

```yaml
sources:
  # Source 1: Referência para values (ref: values)
  - repoURL: 'https://github.com/pcnuness/platform-addons-charts.git'
    targetRevision: 'main'
    ref: values
  
  # Source 2: Chart principal com valueFiles
  - repoURL: 'https://github.com/pcnuness/platform-addons-charts.git'
    targetRevision: 'main'
    path: environments/{{metadata.labels.environment}}/{{values.addonChart}}
    helm:
      valueFiles:
        - $values/environments/default/{{values.addonChart}}/values.yaml
        - $values/environments/{{metadata.labels.environment}}/{{values.addonChart}}/values.yaml
        - $values/addons/{{values.addonChart}}/values.yaml
      ignoreMissingValueFiles: true
```

## 🏗️ **Como Funciona**

### **Source 1: Referência Values**
```yaml
- repoURL: 'https://github.com/pcnuness/platform-addons-charts.git'
  targetRevision: 'main'
  ref: values  # ← Permite acesso aos valueFiles
```

### **Source 2: Chart Principal**
```yaml
- repoURL: 'https://github.com/pcnuness/platform-addons-charts.git'
  targetRevision: 'main'
  path: environments/{{metadata.labels.environment}}/{{values.addonChart}}  # ← Chart principal
  helm:
    valueFiles:  # ← Lista de arquivos de valores
      - $values/environments/default/{{values.addonChart}}/values.yaml
      - $values/environments/{{metadata.labels.environment}}/{{values.addonChart}}/values.yaml
      - $values/addons/{{values.addonChart}}/values.yaml
```

## 📁 **Estrutura de Arquivos Necessária**

```
platform-addons-charts/
└── environments/
    ├── default/addons/crossplane/
    │   ├── Charts.yaml          # ← Chart principal
    │   └── values.yaml          # ← Valores padrão
    └── develop/addons/crossplane/
        ├── Charts.yaml          # ← Chart principal
        └── values.yaml          # ← Valores específicos do ambiente

gitops-cluster-management/
└── addons/crossplane/
    └── values.yaml              # ← Valores personalizados do cluster
```

## 🔄 **Ordem de Precedência dos Values**

Os `valueFiles` são aplicados na seguinte ordem:

1. **`$values/environments/default/{{values.addonChart}}/values.yaml`**
   - Valores base/padrão
   - Configurações comuns a todos os ambientes

2. **`$values/environments/{{metadata.labels.environment}}/{{values.addonChart}}/values.yaml`**
   - Valores específicos do ambiente
   - Sobrescreve valores padrão

3. **`$values/addons/{{values.addonChart}}/values.yaml`**
   - Valores personalizados do cluster
   - Sobrescreve valores anteriores (maior precedência)

## 🚫 **Estrutura Incorreta (Evitar)**

```yaml
# ❌ INCORRETO - Cada source como chart independente
sources:
  - repoURL: 'https://github.com/pcnuness/platform-addons-charts.git'
    targetRevision: 'main'
    path: environments/{{metadata.labels.environment}}/{{values.addonChart}}
  - repoURL: 'https://github.com/pcnuness/gitops-cluster-management.git'
    targetRevision: 'main'
    path: addons/{{values.addonChart}}  # ← Erro: não tem Chart.yaml
```

## ✅ **Estrutura Correta (Usar)**

```yaml
# ✅ CORRETO - Source principal com valueFiles
sources:
  - repoURL: 'https://github.com/pcnuness/platform-addons-charts.git'
    targetRevision: 'main'
    ref: values
  - repoURL: 'https://github.com/pcnuness/platform-addons-charts.git'
    targetRevision: 'main'
    path: environments/{{metadata.labels.environment}}/{{values.addonChart}}
    helm:
      valueFiles:
        - $values/environments/default/{{values.addonChart}}/values.yaml
        - $values/environments/{{metadata.labels.environment}}/{{values.addonChart}}/values.yaml
        - $values/addons/{{values.addonChart}}/values.yaml
```

## 🔧 **Troubleshooting**

### **Erro: "Chart.yaml not found"**
- Verifique se o `path` aponta para um diretório com `Charts.yaml`
- Não use múltiplos sources para charts diferentes
- Use `valueFiles` para combinar valores

### **Erro: "Value file not found"**
- Verifique se o arquivo existe no repositório
- Use `ignoreMissingValueFiles: true` para arquivos opcionais
- Confirme se o `ref: values` está configurado

### **Erro: "Template rendering failed"**
- Verifique se as variáveis `{{values.addonChart}}` e `{{metadata.labels.environment}}` estão definidas
- Confirme se o cluster tem as labels necessárias

## 📋 **Verificação Final**

Para confirmar que a estrutura está correta:

1. **Verificar Chart Principal**:
   ```bash
   # Deve existir
   ls platform-addons-charts/environments/develop/addons/crossplane/Charts.yaml
   ```

2. **Verificar Values**:
   ```bash
   # Deve existir
   ls platform-addons-charts/environments/default/addons/crossplane/values.yaml
   ls platform-addons-charts/environments/develop/addons/crossplane/values.yaml
   ls gitops-cluster-management/addons/crossplane/values.yaml
   ```

3. **Verificar ApplicationSet**:
   ```bash
   kubectl get applicationsets -n argocd addons-oss-crossplane -o yaml
   ```

Esta estrutura garante que os ApplicationSets funcionem corretamente sem erros de "Chart.yaml not found".
