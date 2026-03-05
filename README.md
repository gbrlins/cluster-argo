# Repositório GitOps: Argo CD e Kustomize

Este repositório contém a estrutura de manifestos Kubernetes gerenciados via Argo CD utilizando Kustomize. O objetivo é manter a infraestrutura como código (IaC) e garantir a entrega contínua (CD) das aplicações em diferentes ambientes.

## Estrutura de Diretórios

A organização do repositório segue as boas práticas do Kustomize, dividindo as aplicações em `base` e `overlays` (camadas por ambiente).

```text
.
├── apps/
│   └── meu-app/
│       ├── base/
│       │   ├── kustomization.yaml
│       │   ├── deployment.yaml
│       │   ├── service.yaml
│       │   └── configmap.yaml
│       └── overlays/
│           └── producao/
│               ├── kustomization.yaml
│               ├── patch-replicas.yaml
│               └── configmap-patch.yaml
└── infra/
    └── argo-setup/
        └── values-argocd.yaml
```

Papel de cada pasta (Exemplos)
1. Pasta apps/
Diretório raiz para todas as aplicações hospedadas no cluster. Cada aplicação terá a sua própria subpasta (ex: meu-app).

2. Pasta base/ (O Molde Padrão)
Contém os manifestos genéricos e fundamentais da aplicação. Estes arquivos definem como a aplicação funciona de forma isolada, sem se preocupar com o ambiente final (Desenvolvimento, Homologação ou Produção).

Exemplo: O arquivo base/deployment.yaml pode definir que a aplicação roda com 1 réplica e consome 128Mi de memória.

3. Pasta overlays/<ambiente>/ (As Camadas)
Contém as modificações específicas para um ambiente alvo. O Kustomize pega a base e aplica os "patches" (remendos) definidos aqui por cima dela.

Exemplo: Na pasta overlays/producao/, o arquivo patch-replicas.yaml altera as réplicas de 1 para 3. O arquivo kustomization.yaml desta pasta herda a base, aplica o patch e injeta labels automaticamente (ex: env: producao) em todos os recursos.

Guia de Instalação e Setup do Argo CD
Este guia descreve como iniciar a infraestrutura do zero em um cluster Kubernetes utilizando o Rancher.

Passo 1: Preparação do Namespace e Credenciais
Antes de instalar o Argo CD, crie o namespace e a credencial para puxar imagens de um registry privado (se aplicável).

```
# Criar o namespace
kubectl create namespace argo

# Criar a credencial do registry (substitua os valores de usuario e senha)
kubectl create secret docker-registry application-collection \
  --docker-server=dp.apps.rancher.io \
  --docker-username=seu_usuario@apps.rancher.io \
  --docker-password=sua_senha_em_base64 \
  -n argo
```

Passo 2: Instalação via Rancher (Helm)
Acesse o painel do Rancher e selecione o seu cluster.

Vá em Apps > Charts e busque por argo-cd.

Selecione o namespace argo previamente criado.

Edite o YAML (Values) e garanta as seguintes configurações mínimas:
```
global:
  domain: argocd.seudominio.com
  imagePullSecrets:
    - name: application-collection

server:
  extraArgs:
    - --insecure # Necessário se o Ingress for fazer a terminação SSL
  ingress:
    enabled: true
    ingressClassName: nginx
    hostname: argocd.seudominio.com
    # Adicione anotações do cert-manager aqui, se usar TLS
```


Conclua a instalação e aguarde os pods iniciarem no namespace argo.

Passo 3: Acesso Inicial à Interface do Argo CD
Após a instalação, recupere a senha inicial gerada automaticamente para o usuário admin:

```
kubectl -n argo get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

Acesse a URL do Ingress configurado (ex: https://argocd.seudominio.com), faça o login com o usuário admin e a senha recuperada. Recomenda-se alterar a senha e deletar a secret em seguida.

Passo 4: Fazer o Deploy de uma Aplicação (meu-app)
Para que o Argo CD comece a monitorar a pasta producao do Kustomize e aplique no cluster, crie o seguinte manifesto e aplique no Kubernetes (ajuste a URL do repositório):

Crie um arquivo meu-app-argo.yaml:
```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: meu-app-producao
  namespace: argo
spec:
  project: default
  source:
    repoURL: '[https://github.com/seu-utilizador/seu-repo.git](https://github.com/seu-utilizador/seu-repo.git)'
    targetRevision: HEAD
    path: apps/meu-app/overlays/producao
  destination:
    server: '[https://kubernetes.default.svc](https://kubernetes.default.svc)'
    namespace: meu-app-prod
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

Aplique no cluster:
```
kubectl apply -f meu-app-argo.yaml
```

A partir deste momento, qualquer mudança feita na branch principal do diretório apps/meu-app/overlays/producao/ será detectada e aplicada automaticamente pelo Argo CD.
