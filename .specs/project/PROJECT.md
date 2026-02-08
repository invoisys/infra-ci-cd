# Infra CI/CD

**Vision:** Plataforma de CI/CD reutilizável e extensível para deploy de aplicações no AWS ECS Fargate via GitHub Actions.  
**For:** Times de desenvolvimento que precisam de pipelines automatizados e padronizados.  
**Solves:** Complexidade de criar e manter pipelines de CI/CD para múltiplas aplicações e ambientes.

## Goals

- **Reutilização:** Um único workflow reutilizável para todas as aplicações .NET (APIs e Workers)
- **Extensibilidade:** Arquitetura modular que permite adicionar novas tecnologias e ambientes facilmente
- **Simplicidade:** Configuração mínima para consumidores, com defaults sensatos
- **Segurança:** Zero secrets em plain text, usando exclusivamente GitHub Environments
- **Confiabilidade:** Suporte a rollback com aprovação em produção

## Tech Stack

**Core:**
- Platform: GitHub Actions (Reusable Workflows)
- Cloud: AWS (ECS Fargate, ECR, ALB)
- Language: .NET 8.0

**Key dependencies:**
- aws-actions/configure-aws-credentials
- aws-actions/amazon-ecs-render-task-definition
- aws-actions/amazon-ecs-deploy-task-definition
- docker/build-push-action

## Scope

**v1 includes:**
- Pipeline reutilizável para .NET (API e Worker)
- Multi-ambiente: dev, qa, sbx, prd
- Suporte Greenfield e Brownfield
- Rollback manual com aprovação
- Health check pós-deploy

**Explicitly out of scope:**
- Outras linguagens além de .NET
- Kubernetes/EKS
- Multi-cloud (Azure, GCP)
- Auto-scaling configurável
- Métricas e alertas

## Constraints

- Timeline: Iterativo, sem deadline fixo
- Technical: Dependência da infraestrutura AWS existente
- Resources: Manutenção por equipe de plataforma
