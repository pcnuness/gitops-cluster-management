# Configuração de Clusters no ArgoCD

Este documento explica como configurar clusters no ArgoCD com as labels necessárias para o funcionamento dos ApplicationSets.

## 🏷️ **Labels Necessárias**

Para que os ApplicationSets funcionem corretamente, cada cluster deve ter a label `environment` com um dos seguintes valores:
- `develop`
- `uat` 
- `prod`

## 📋 **Como Configurar Clusters**

### **Método 1: Via ArgoCD CLI**

```bash
# Adicionar cluster com label
argocd cluster add <cluster-context> \
  --label environment=develop \
  --name cluster-develop-01

argocd cluster add <cluster-context> \
  --label environment=uat \
  --name cluster-uat-01

argocd cluster add <cluster-context> \
  --label environment=prod \
  --name cluster-prod-01
```

### **Método 2: Via ArgoCD UI**

1. Acesse **Settings** → **Clusters**
2. Clique em **Connect Cluster**
3. Adicione as labels:
   ```
   Key: environment
   Value: develop (ou uat, prod)
   ```

### **Método 3: Via Secret do Kubernetes**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: cluster-develop-01
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: cluster
    environment: develop  # ← Label necessária
type: Opaque
stringData:
  name: cluster-develop-01
  server: https://kubernetes.default.svc
  config: |
    {
      "bearerToken": "<token>",
      "tlsClientConfig": {
        "insecure": false,
        "caData": "<ca-data>"
      }
    }
```

## 🔍 **Verificar Configuração**

### **Listar Clusters com Labels**
```bash
argocd cluster list --output wide
```

### **Verificar Labels de um Cluster Específico**
```bash
kubectl get secret -n argocd -l argocd.argoproj.io/secret-type=cluster -o yaml
```

## 🚫 **Excluir Cluster in-cluster**

Para garantir que os addons NÃO sejam instalados no cluster `in-cluster` (ArgoCD), você pode:

### **Opção 1: Não adicionar label environment**
O cluster `in-cluster` não deve ter a label `environment`, então será automaticamente excluído.

### **Opção 2: Usar matchLabels para ser mais específico**
```yaml
selector:
  matchLabels:
    environment: develop  # Apenas clusters com esta label específica
```

### **Opção 3: Usar matchExpressions com NotIn**
```yaml
selector:
  matchExpressions:
    - key: argocd.argoproj.io/secret-type
      operator: Exists
    - key: environment
      operator: NotIn
      values: ["in-cluster"]
```

## 📊 **Estrutura de Clusters Recomendada**

```
Clusters ArgoCD:
├── in-cluster (ArgoCD)          # Sem label environment
├── cluster-develop-01           # environment=develop
├── cluster-develop-02           # environment=develop
├── cluster-uat-01               # environment=uat
├── cluster-prod-01              # environment=prod
└── cluster-prod-02              # environment=prod
```

## ✅ **Verificação Final**

Após configurar as labels, verifique se:

1. **Clusters têm labels corretas**:
   ```bash
   kubectl get secrets -n argocd -l argocd.argoproj.io/secret-type=cluster -o jsonpath='{.items[*].metadata.labels.environment}'
   ```

2. **ApplicationSets detectam clusters**:
   ```bash
   kubectl get applicationsets -n argocd addons-oss-crossplane -o yaml
   ```

3. **Applications são criadas apenas nos clusters corretos**:
   ```bash
   kubectl get applications -n argocd -l app.kubernetes.io/instance=addons-oss-crossplane
   ```

## 🔧 **Troubleshooting**

### **Problema: Addons não são instalados**
- Verifique se o cluster tem a label `environment`
- Confirme se o valor da label está na lista: `["develop", "uat", "prod"]`

### **Problema: Addons são instalados no in-cluster**
- Verifique se o cluster `in-cluster` NÃO tem a label `environment`
- Use `kubectl describe secret -n argocd in-cluster` para verificar

### **Problema: Path incorreto nos sources**
- Confirme se o path usa `{{values.environment}}` corretamente
- Verifique se existe o diretório correspondente no repositório

## 📝 **Exemplo Completo**

```bash
# Adicionar cluster de desenvolvimento
argocd cluster add arn:aws:eks:us-east-1:123456789012:cluster/develop-cluster \
  --label environment=develop \
  --label region=us-east-1 \
  --name develop-cluster-01

# Verificar se foi adicionado corretamente
argocd cluster list --output wide | grep develop

# Verificar se ApplicationSet detecta o cluster
kubectl get applications -n argocd | grep develop-cluster-01
```

Esta configuração garante que os addons sejam instalados apenas nos clusters com as labels de ambiente apropriadas, excluindo o cluster `in-cluster` do ArgoCD.
