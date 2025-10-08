# Troubleshooting Metrics Server ApplicationSet

## 🔍 **Problema Atual**

```
Failed to load target state: failed to generate manifest for source 2 of 2: rpc error: code = Unknown desc = error getting helm repos: error retrieving helm dependency repos: error reading helm chart from <path to cached source>/environments/develop/metrics-server/Chart.yaml: open <path to cached source>/environments/develop/metrics-server/Chart.yaml: no such file or directory
```

## 🔍 **Análise do Problema**

### **1. Path Incorreto no Erro**
O erro mostra que o ArgoCD está procurando em:
```
environments/develop/metrics-server/Chart.yaml
```

Mas o arquivo está localizado em:
```
environments/develop/addons/metrics-server/Chart.yaml
```

### **2. Possíveis Causas**

1. **Cache do ArgoCD**: O ArgoCD pode estar usando uma versão em cache do ApplicationSet
2. **Referência de Template**: `{{metadata.labels.environment}}` pode não estar sendo resolvida corretamente
3. **Estrutura de Sources**: Problema com múltiplos sources

## 🛠️ **Soluções Implementadas**

### **Solução 1: AppSet Simplificado**
Criado `appset-metrics-server-test.yaml` com:
- Path fixo: `environments/develop/addons/metrics-server`
- Source único em vez de múltiplos sources
- Valores fixos em vez de templates

### **Solução 2: Correção do AppSet Original**
Atualizado `appset-metrics-server.yaml` com:
- Path corrigido: `environments/{{metadata.labels.environment}}/addons/{{values.addonChart}}`
- ValueFiles corrigidos com `/addons/` incluído

## 📋 **Estrutura de Arquivos Confirmada**

```
platform-addons-charts/environments/
├── default/addons/metrics-server/
│   ├── Charts.yaml ✅
│   └── values.yaml ✅
└── develop/addons/metrics-server/
    ├── Charts.yaml ✅
    └── values.yaml ✅

gitops-cluster-management/addons/metrics-server/
└── values.yaml ✅
```

## 🔧 **Passos para Resolução**

### **1. Testar AppSet Simplificado**
```bash
# Aplicar o AppSet de teste
kubectl apply -f appset-metrics-server-test.yaml

# Verificar se funciona
kubectl get applicationsets -n argocd addons-oss-metrics-server-test
```

### **2. Limpar Cache do ArgoCD**
Se o problema persistir, pode ser necessário:
```bash
# Reiniciar o ApplicationSet Controller
kubectl rollout restart deployment/argocd-applicationset-controller -n argocd

# Ou deletar e recriar o ApplicationSet
kubectl delete applicationset -n argocd addons-oss-metrics-server
kubectl apply -f appset-metrics-server.yaml
```

### **3. Verificar Labels do Cluster**
Certificar-se de que o cluster tem a label correta:
```bash
# Verificar clusters com label environment=develop
kubectl get secrets -n argocd -l argocd.argoproj.io/secret-type=cluster -o jsonpath='{.items[*].metadata.labels.environment}'
```

## 🎯 **AppSet de Teste vs Original**

### **AppSet de Teste (Simplificado)**
```yaml
source:
  repoURL: 'https://github.com/pcnuness/platform-addons-charts.git'
  targetRevision: 'main'
  path: environments/develop/addons/metrics-server  # ← Path fixo
  helm:
    valueFiles:
      - $values/environments/default/addons/metrics-server/values.yaml
      - $values/environments/develop/addons/metrics-server/values.yaml
      - $values/addons/metrics-server/values.yaml
```

### **AppSet Original (Com Templates)**
```yaml
sources:
  - repoURL: 'https://github.com/pcnuness/platform-addons-charts.git'
    targetRevision: 'main'
    ref: values
  - repoURL: 'https://github.com/pcnuness/platform-addons-charts.git'
    targetRevision: 'main'
    path: environments/{{metadata.labels.environment}}/addons/{{values.addonChart}}  # ← Template
```

## 🚨 **Próximos Passos**

1. **Testar o AppSet simplificado** primeiro
2. **Se funcionar**: O problema é com os templates
3. **Se não funcionar**: O problema é com a estrutura de arquivos ou cache
4. **Aplicar correções** baseadas nos resultados do teste

## 📝 **Notas Importantes**

- O AppSet de teste usa `source` (singular) em vez de `sources` (plural)
- Valores fixos em vez de templates para eliminar variáveis de resolução
- Path absoluto para garantir que o ArgoCD encontre o Chart correto
- Apenas clusters com `environment=develop` serão selecionados no teste
