# Workflows Compostos - Especificação

## Problem Statement

O pipeline atual (`reusable-ecs-pipeline.yml`) é **monolítico com 773 linhas** que acumula build, test, docker e deploy em um único arquivo. Isso gera:
- **Duplicação**: Cada aplicação replica configurações similares
- **Manutenção difícil**: Mudanças requerem analisar o arquivo inteiro
- **Acoplamento a .NET**: Build e test estão "hard-coded" para .NET, impossibilitando Node.js ou outras tecnologias
- **Rigidez de fluxo**: Todas as branches executam build+deploy, sem flexibilidade (ex: `dev` poderia ser só build)

## Goals

- [ ] Separar o pipeline em 4 workflows compostos reutilizáveis (build, test, docker, deploy)
- [ ] Permitir configuração de fluxo por ambiente/branch (ex: dev=build, sbx=build+deploy)
- [ ] Abstrair build para suportar .NET **e** Node.js com a mesma interface
- [ ] Zero breaking changes: aplicações existentes continuam funcionando sem alteração

## Out of Scope

- Outras tecnologias além de .NET e Node.js (Python, Go, etc.)
- Mudanças na infraestrutura AWS (ECS, ECR, ALB)
- Auto-scaling ou métricas
- Kubernetes/EKS

---

## User Stories

### P1: Separar Build em Workflow Composto ⭐ MVP

**User Story**: Como desenvolvedor, quero um workflow de build isolado para que eu possa reutilizá-lo em diferentes contextos e tecnologias.

**Why P1**: Base da modularização - outros workflows dependem dos artefatos de build.

**Acceptance Criteria**:

1. WHEN chamado com `technology: dotnet` THEN workflow SHALL executar `dotnet restore` e `dotnet build`
2. WHEN chamado com `technology: node` THEN workflow SHALL executar `npm ci` e `npm run build`
3. WHEN build falha THEN workflow SHALL retornar exit code != 0 e `build_success: false`
4. WHEN build sucede THEN workflow SHALL exportar output `artifact_path` com caminho dos artefatos
5. WHEN `technology` não é informada THEN workflow SHALL assumir `dotnet` como default

**Independent Test**: Criar workflow que chama apenas o build composto e verificar se artefatos são gerados corretamente.

---

### P1: Separar Test em Workflow Composto ⭐ MVP

**User Story**: Como desenvolvedor, quero um workflow de testes isolado para executar testes de forma independente ou como parte do pipeline.

**Why P1**: Permite rodar testes em PRs sem executar build/deploy completo.

**Acceptance Criteria**:

1. WHEN chamado com `technology: dotnet` THEN workflow SHALL executar `dotnet test`
2. WHEN chamado com `technology: node` THEN workflow SHALL executar `npm test`
3. WHEN testes falham THEN workflow SHALL retornar exit code != 0 e `tests_passed: false`
4. WHEN testes passam THEN workflow SHALL exportar `tests_passed: true` e `coverage_report_path` (se disponível)
5. WHEN input `skip_tests: true` THEN workflow SHALL retornar `tests_passed: true` sem executar testes

**Independent Test**: Criar workflow que chama apenas o test composto e verificar se resultado é reportado corretamente.

---

### P1: Separar Docker em Workflow Composto ⭐ MVP

**User Story**: Como desenvolvedor, quero um workflow de build Docker isolado para construir e enviar imagens independentemente da tecnologia de aplicação.

**Why P1**: Docker é agnóstico de tecnologia e precisa ser reutilizado.

**Acceptance Criteria**:

1. WHEN chamado THEN workflow SHALL fazer docker build usando o Dockerfile especificado
2. WHEN `push: true` THEN workflow SHALL fazer login no ECR e push da imagem
3. WHEN `push: false` THEN workflow SHALL apenas fazer build local (para validação em PRs)
4. WHEN push sucede THEN workflow SHALL exportar `image_digest`, `image_tag` e `full_image_uri`
5. WHEN `use_template_dockerfile: true` THEN workflow SHALL usar Dockerfile do repo de templates

**Independent Test**: Chamar workflow docker isoladamente e verificar se imagem aparece no ECR com as tags corretas.

---

### P1: Separar Deploy em Workflow Composto ⭐ MVP

**User Story**: Como desenvolvedor, quero um workflow de deploy isolado para fazer deploy de imagens já construídas no ECS.

**Why P1**: Permite rollback, redeploy de imagens existentes, e deploy manual.

**Acceptance Criteria**:

1. WHEN chamado com `image_uri` THEN workflow SHALL registrar task definition com essa imagem
2. WHEN service existe THEN workflow SHALL fazer update-service
3. WHEN service não existe THEN workflow SHALL fazer create-service
4. WHEN `service_type: api` THEN workflow SHALL configurar target group e listener rule
5. WHEN deploy completa THEN workflow SHALL exportar `service_arn` e `task_definition_arn`
6. WHEN deploy falha THEN workflow SHALL retornar exit code != 0 com mensagem de erro clara

**Independent Test**: Fazer deploy de uma imagem existente do ECR e verificar se service atualiza.

---

### P1: Criar Workflow Orquestrador ⭐ MVP

**User Story**: Como desenvolvedor, quero um workflow orquestrador que chame os compostos na ordem correta para manter a experiência atual.

**Why P1**: Garante retrocompatibilidade - aplicações existentes continuam funcionando.

**Acceptance Criteria**:

1. WHEN chamado com mesmos inputs do pipeline atual THEN workflow SHALL produzir mesmo resultado
2. WHEN chamado THEN workflow SHALL respeitar a sequência: build → test → docker → deploy
3. WHEN qualquer fase falha THEN workflow SHALL interromper e não executar fases subsequentes
4. WHEN `deploy_only: true` THEN workflow SHALL pular build/test/docker e ir direto para deploy
5. WHEN todos outputs do pipeline atual existirem THEN workflow SHALL exportá-los igualmente

**Independent Test**: Substituir chamada do pipeline atual pelo orquestrador e verificar deploy funcional.

---

### P2: Configuração de Fluxo por Branch/Ambiente

**User Story**: Como engenheiro de plataforma, quero configurar quais fases executam por branch para otimizar recursos.

**Why P2**: Importante para eficiência, mas não bloqueia uso básico.

**Acceptance Criteria**:

1. WHEN configurado `branches.dev: [build, test]` THEN branch dev SHALL executar apenas build e test
2. WHEN configurado `branches.prd: [build, test, docker, deploy]` THEN branch prd SHALL executar pipeline completo
3. WHEN branch não tem configuração explícita THEN workflow SHALL executar pipeline completo (default)
4. WHEN fase é pulada THEN workflow SHALL logar claramente "Skipping deploy phase for branch X"

**Independent Test**: Configurar dev para só build e verificar que não faz deploy.

---

### P2: Suporte Multi-Tecnologia Extensível

**User Story**: Como engenheiro de plataforma, quero adicionar novas tecnologias sem modificar workflows core.

**Why P2**: Prepara para escalabilidade futura.

**Acceptance Criteria**:

1. WHEN nova tecnologia precisa ser adicionada THEN apenas arquivo de configuração específico SHALL ser criado
2. WHEN `technology: python` (futuro) THEN workflow SHALL buscar configuração em arquivo de tecnologia
3. WHEN tecnologia não existe THEN workflow SHALL falhar com erro claro "Unknown technology: X"

**Independent Test**: Adicionar tecnologia "custom" via configuração e verificar que workflow executa.

---

### P3: Outputs Padronizados entre Workflows

**User Story**: Como desenvolvedor de automação, quero outputs consistentes para integrar com outras ferramentas.

**Why P3**: Nice-to-have para integrações avançadas.

**Acceptance Criteria**:

1. WHEN qualquer workflow completa THEN SHALL exportar `status`, `duration_seconds`, `started_at`, `completed_at`
2. WHEN workflow tem artefatos THEN SHALL exportar metadados em formato JSON padronizado

---

## Edge Cases

- WHEN repository não tem testes configurados THEN test workflow SHALL logar warning e passar com `tests_passed: true`
- WHEN Dockerfile não existe no caminho especificado THEN docker workflow SHALL falhar com erro claro
- WHEN credenciais AWS são inválidas THEN workflows SHALL falhar fast no início, não após build
- WHEN ECR repository não existe THEN docker workflow SHALL indicar erro específico (não genérico)
- WHEN ECS cluster não existe THEN deploy workflow SHALL falhar com mensagem indicando problema
- WHEN branch name contém caracteres especiais THEN workflows SHALL sanitizar para tags e nomes de recursos

---

## Success Criteria

- [ ] Pipeline modularizado funciona identicamente ao atual para aplicações existentes
- [ ] Tempo de execução total não aumenta mais que 10% (overhead de coordenação)
- [ ] Novas aplicações podem ser configuradas com < 50 linhas de YAML
- [ ] Adicionar Node.js requer apenas criar arquivo de configuração de tecnologia
- [ ] Workflow orquestrador tem < 100 linhas (delegando para compostos)
