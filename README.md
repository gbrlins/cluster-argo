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
