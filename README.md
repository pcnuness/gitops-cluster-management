# GitOps Cluster Management

Repositório para gerenciamento de clusters Kubernetes usando GitOps com ArgoCD.

## 🎯 Visão Geral

Este repositório gerencia dois tipos de componentes:

1. **Addons** (Helm Charts): Componentes de infraestrutura como metrics-server, grafana, crossplane
2. **Applications** (Plain YAML): Aplicações usando manifestos Kubernetes puros

## 📁 Estrutura

```
gitops-cluster-management/
├── addons/                    # Valores customizados para Helm Charts
├── applications/              # Aplicações com manifestos YAML puros
├── bootstraps/               # ApplicationSets do ArgoCD
├── iac/                      # Infrastructure as Code (Terraform, etc)
└── .docs/                    # 📚 Documentação completa
```

## 📚 Documentação

Toda a documentação está organizada no diretório [`.docs/`](./.docs/):

### Guias Principais

- **[README.md](./.docs/README.md)** - Documentação principal e visão geral
- **[ADDONS_GUIDE.md](./.docs/ADDONS_GUIDE.md)** - Como gerenciar addons (Helm)
- **[APPLICATIONS_GUIDE.md](./.docs/APPLICATIONS_GUIDE.md)** - Como gerenciar applications (YAML)

### Referências Técnicas

- **[SOURCES_STRUCTURE.md](./.docs/SOURCES_STRUCTURE.md)** - Estrutura de sources multi-repositório
- **[ROLLOUT_STRATEGIES.md](./.docs/ROLLOUT_STRATEGIES.md)** - Estratégias de rollout e versionamento
- **[ARCHITECTURE.md](./.docs/ARCHITECTURE.md)** - Diagrama de arquitetura
- **[CLUSTER_CONFIGURATION.md](./.docs/CLUSTER_CONFIGURATION.md)** - Configuração de clusters
- **[TROUBLESHOOTING_METRICS_SERVER.md](./.docs/TROUBLESHOOTING_METRICS_SERVER.md)** - Troubleshooting

## 🚀 Quick Start

### Adicionar um Addon (Helm)

```bash
# 1. Adicionar Chart no platform-addons-charts
# 2. Criar ApplicationSet aqui
# 3. Opcional: adicionar custom values

# Ver guia completo: .docs/ADDONS_GUIDE.md
```

### Adicionar uma Application (YAML)

```bash
# 1. Criar diretório em applications/
mkdir -p applications/my-app

# 2. Adicionar manifestos YAML
cat > applications/my-app/01-namespace.yaml <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: my-app
EOF

# 3. Commit e push - ArgoCD detecta automaticamente!
git add applications/my-app/
git commit -m "feat(app): add my-app"
git push

# Ver guia completo: .docs/APPLICATIONS_GUIDE.md
```

## 🔗 Repositórios Relacionados

- **[platform-addons-charts](https://github.com/pcnuness/platform-addons-charts)** - Charts Helm centralizados

## 📊 Componentes Gerenciados

### Addons (Helm)
- metrics-server 3.12.2
- grafana 10.0.0
- crossplane 1.15.3

### Applications (YAML)
- game-2048
- inflate

## 🛠️ Ferramentas

- **ArgoCD**: GitOps continuous delivery
- **Helm**: Package manager para Kubernetes
- **Crossplane**: Infrastructure as Code

## 🤝 Contribuindo

1. Crie uma branch de feature
2. Faça suas alterações
3. Teste em ambiente de develop
4. Crie um Pull Request
5. Aguarde revisão e aprovação

Ver detalhes em [.docs/README.md](./.docs/README.md)

---

**📖 Para documentação completa, acesse o diretório [`.docs/`](./.docs/)**
