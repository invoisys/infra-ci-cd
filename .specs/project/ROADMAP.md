# Roadmap

**Current Milestone:** Modularização do Pipeline  
**Status:** In Progress

---

## Milestone 1: Modularização do Pipeline

**Goal:** Transformar o pipeline monolítico em arquitetura modular e extensível  
**Target:** Estrutura base pronta para consumo por múltiplas aplicações

### Features

**Workflows Compostos** - ✅ IMPLEMENTED
- Separar build, test, docker, deploy em workflows individuais
- Criar workflow orquestrador que chama os compostos
- Manter compatibilidade retroativa com uso atual
- Suportar .NET e Node.js com mesma interface

**Configuração por Ambiente** - PLANNED
- Extrair configurações de ambiente para arquivos declarativos
- Suportar override de configurações por aplicação
- Templates de configuração para novos ambientes

**Suporte Multi-Tecnologia** - PLANNED
- Abstrair build steps por linguagem/framework
- Plugin system para adicionar novas tecnologias
- Manter .NET como referência, preparar para Node.js, Python

---

## Milestone 2: Extensibilidade Avançada

**Goal:** Permitir customizações sem modificar código core

### Features

**Hooks e Extensões** - PLANNED
- Pre/post hooks em cada fase do pipeline
- Callbacks customizáveis (notificações, validações)
- Integração com ferramentas externas (SonarQube, etc.)

**Self-Service Onboarding** - PLANNED
- Wizard/template para criar novos repositórios
- Geração automática de workflows
- Documentação inline

---

## Milestone 3: Observabilidade e Governança

**Goal:** Visibilidade e controle sobre todos os deployments

### Features

**Dashboard de Deployments** - PLANNED
- Status consolidado de todas as aplicações
- Histórico de deployments por ambiente
- Métricas de lead time e frequência

**Políticas de Deployment** - PLANNED
- Regras configuráveis por ambiente
- Validações obrigatórias (cobertura, security scan)
- Audit trail

---

## Future Considerations

- Suporte a outros cloud providers (Azure, GCP)
- Kubernetes/EKS como alternativa ao ECS
- GitOps com ArgoCD/Flux
- Feature flags integration
- Canary deployments
