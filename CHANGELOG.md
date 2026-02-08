# Changelog

Todas as mudanças notáveis neste projeto serão documentadas neste arquivo.

O formato é baseado em [Keep a Changelog](https://keepachangelog.com/pt-BR/1.0.0/),
e este projeto adere ao [Versionamento Semântico](https://semver.org/lang/pt-BR/).

## [1.0.0] - 2026-02-07

### Adicionado

- **Workflows Compostos**: Arquitetura modular com 4 workflows independentes
  - `composite-build.yml` - Build agnóstico de tecnologia (.NET, Node.js)
  - `composite-test.yml` - Execução de testes automatizados
  - `composite-docker.yml` - Build e push de imagens para ECR
  - `composite-deploy.yml` - Deploy no ECS Fargate
- **Orchestrator**: Workflow orquestrador (`orchestrator.yml`) que coordena os compostos
- **Testes de Integração**: Workflow `test-composites.yml` para validar os compostos
- **Documentação de Consumo**: `COMPOSITES.md` com guia completo de inputs/outputs
- **Suporte Multi-Tecnologia**: Abstração para .NET e Node.js com mesma interface

### Alterado

- `reusable-ecs-pipeline.yml` agora delega para o orchestrator internamente
- Estrutura de documentação reorganizada (`.specs/` para interno, `.github/` para externo)

### Decisões Técnicas

- **AD-005**: Usar prefixo `composite-*.yml` ao invés de subpasta `composites/`
- **AD-003**: Usar reusable workflows ao invés de composite actions
- **AD-004**: Configuração de tecnologia embutida nos workflows
- **AD-002**: Separação em 4 workflows compostos (build, test, docker, deploy)
- **AD-001**: Adotar workflows compostos do GitHub Actions

### Compatibilidade

- ✅ Zero breaking changes para consumidores existentes
- ✅ Todos os inputs do pipeline original mantidos
- ✅ Todos os outputs do pipeline original mantidos

---

## [Unreleased]

### Planejado

- Configuração de fluxo por branch/ambiente (P2)
- Suporte multi-tecnologia extensível via arquivos de configuração (P2)
- Hooks e extensões customizáveis (Milestone 2)
