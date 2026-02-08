# Workflows Compostos - Tasks

**Design**: `.specs/features/workflows-compostos/design.md`  
**Status**: Implemented ✅

---

## Execution Plan

### Phase 1: Foundation (Sequential)

Configurações base que são dependência para todos os workflows compostos.

```
T1 → T2 → T3
```

### Phase 2: Workflows Compostos (Parallel OK)

Após configurações, os workflows compostos podem ser desenvolvidos em paralelo.

```
     ┌→ T4 (build.yml) ──┐
T3 ──┼→ T5 (test.yml)  ──┼──→ T9
     ├→ T6 (docker.yml) ─┤
     └→ T7 (deploy.yml) ─┘
          T8 (actions) ──┘
```

### Phase 3: Orchestrator (Sequential)

Integração dos workflows compostos.

```
T9 → T10 → T11
```

### Phase 4: Migration & Validation (Sequential)

Testes de integração e migração.

```
T11 → T12 → T13
```

---

## Task Breakdown

### T1: Criar estrutura de diretórios ✅

**What**: Criar estrutura de pastas para workflows compostos e configurações de tecnologia  
**Where**: `.github/workflows/` (prefixo `composite-*.yml`)  
**Depends on**: None  
**Reuses**: None

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [x] Arquivos `composite-*.yml` criados na pasta workflows
- [x] Estrutura segue padrão definido no design

**Note**: Implementação usou prefixo `composite-*.yml` ao invés de subpasta (AD-005)

**Verify**:
```bash
ls -la .github/workflows/composites/
ls -la .github/tech-configs/
```

---

### T2: Criar configuração de tecnologia .NET ✅

**What**: Criar arquivo de configuração para tecnologia .NET com comandos de build/test/cache  
**Where**: Configuração embutida nos workflows compostos  
**Depends on**: T1  
**Reuses**: Padrões de cache e comandos do pipeline atual

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [x] Configuração .NET implementada nos workflows compostos
- [x] Contém `setup_action: actions/setup-dotnet@v4`
- [x] Contém comandos: `restore`, `build`, `test`
- [x] Contém configuração de cache NuGet
- [x] `default_version: "8.0"` configurado

**Verify**:
```bash
cat .github/tech-configs/dotnet.yml
# Deve conter: name, default_version, setup_action, cache, commands
```

---

### T3: Criar configuração de tecnologia Node.js ✅

**What**: Criar arquivo de configuração para tecnologia Node.js com comandos de build/test/cache  
**Where**: Configuração embutida nos workflows compostos  
**Depends on**: T1  
**Reuses**: Padrão de estrutura do `dotnet.yml`

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [x] Configuração Node.js implementada nos workflows compostos
- [x] Contém `setup_action: actions/setup-node@v4`
- [x] Contém comandos: `restore` (npm ci), `build`, `test`
- [x] Contém configuração de cache npm
- [x] `default_version: "20"` configurado

**Verify**:
```bash
cat .github/tech-configs/node.yml
# Estrutura deve ser consistente com dotnet.yml
```

---

### T4: Criar workflow composto de Build [P] ✅

**What**: Criar workflow reutilizável de build agnóstico de tecnologia  
**Where**: `.github/workflows/composite-build.yml`  
**Depends on**: T2, T3  
**Reuses**: Lógica de cache do pipeline atual (`reusable-ecs-pipeline.yml`)

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [x] Workflow com `workflow_call` trigger
- [x] Input `technology` com default `dotnet`
- [x] Input `technology_version` opcional
- [x] Input `working_directory` com default `src`
- [x] Input `project_name` opcional (para .NET)
- [x] Input `build_args` opcional
- [x] Output `artifact_path`, `artifact_name`, `build_success`
- [x] Condicional para executar comandos .NET ou Node baseado em `technology`
- [x] Upload de artefatos via `actions/upload-artifact@v4`
- [x] < 100 linhas

**Verify**:
```yaml
# Testar chamando isoladamente:
jobs:
  test-build:
    uses: ./.github/workflows/composites/build.yml
    with:
      technology: dotnet
      working_directory: src
```

---

### T5: Criar workflow composto de Test [P] ✅

**What**: Criar workflow reutilizável de testes agnóstico de tecnologia  
**Where**: `.github/workflows/composite-test.yml`  
**Depends on**: T2, T3  
**Reuses**: Lógica de test do pipeline atual

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [x] Workflow com `workflow_call` trigger
- [x] Input `technology` com default `dotnet`
- [x] Input `technology_version` opcional
- [x] Input `working_directory` com default `src`
- [x] Input `skip_tests` com default `false`
- [x] Input `test_args` opcional
- [x] Output `tests_passed`, `coverage_report_path`
- [x] Lógica de skip quando `skip_tests: true`
- [x] Condicional para executar comandos .NET ou Node
- [x] < 80 linhas

**Verify**:
```yaml
# Testar chamando isoladamente:
jobs:
  test-tests:
    uses: ./.github/workflows/composites/test.yml
    with:
      technology: dotnet
      skip_tests: false
```

---

### T6: Criar workflow composto de Docker [P] ✅

**What**: Criar workflow reutilizável para build e push de imagens Docker  
**Where**: `.github/workflows/composite-docker.yml`  
**Depends on**: T1  
**Reuses**: Lógica de Docker build/push e ECR login do pipeline atual (linhas ~L350-L420)

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [x] Workflow com `workflow_call` trigger
- [x] Inputs: `ecr_repo`, `service_type` (required)
- [x] Inputs: `push`, `dockerfile_path`, `use_template_dockerfile`, `templates_repo`, `templates_ref`
- [x] Inputs: `project_name`, `aws_region`, `ecr_registry`, `image_tags`
- [x] Secrets: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `REPO_ACCESS_TOKEN`
- [x] Outputs: `image_digest`, `image_tag`, `full_image_uri`, `registry`
- [x] Login ECR via `aws-actions/amazon-ecr-login@v2`
- [x] Checkout de templates repo quando `use_template_dockerfile: true`
- [x] Docker build com build args para .NET (`PROJECT_NAME`)
- [x] Push condicional baseado em `push` input
- [x] < 120 linhas

**Verify**:
```yaml
# Testar com push: false (dry-run):
jobs:
  test-docker:
    uses: ./.github/workflows/composites/docker.yml
    with:
      ecr_repo: test-repo
      service_type: api
      push: false
```

---

### T7: Criar workflow composto de Deploy [P] ✅

**What**: Criar workflow reutilizável para deploy no ECS Fargate  
**Where**: `.github/workflows/composite-deploy.yml`  
**Depends on**: T1  
**Reuses**: Toda lógica de ECS do pipeline atual (task def L420-L512, target group L515-L556, listener L559-L610, service L613-L698, metadata L712-L725)

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [x] Workflow com `workflow_call` trigger
- [x] Inputs required: `image_uri`, `ecs_service`, `environment`, `service_type`
- [x] Inputs de infraestrutura: todos do pipeline atual (task config, network, load balancer)
- [x] Secrets: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `ECS_CLUSTER`, `ECS_TASK_EXECUTION_ROLE_ARN`
- [x] Outputs: `service_arn`, `task_definition_arn`, `deploy_metadata_artifact`
- [x] Lógica de register task definition
- [x] Lógica de target group (quando `service_type: api`)
- [x] Lógica de listener rule (quando `service_type: api`)
- [x] Lógica de create/update service
- [x] Wait for service stabilization
- [x] Geração de deploy metadata artifact
- [x] < 250 linhas

**Verify**:
```yaml
# Testar com imagem existente:
jobs:
  test-deploy:
    uses: ./.github/workflows/composites/deploy.yml
    with:
      image_uri: 123456789.dkr.ecr.us-east-1.amazonaws.com/app:sha-abc123
      ecs_service: test-service
      environment: dev
      service_type: api
```

---

### T8: Criar action de validação de inputs [P] ✅

**What**: Criar composite action para validação comum de inputs  
**Where**: `.github/actions/validate-inputs/action.yml`  
**Depends on**: T1  
**Reuses**: Lógica de validação do pipeline atual (L297-L304)

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [x] Action com inputs para validar: `technology`, `environment`, `service_type`
- [x] Validação de `technology` em lista permitida (dotnet, node)
- [x] Validação de `environment` em lista permitida (dev, qa, sbx, prd)
- [x] Validação de `service_type` em lista permitida (api, worker)
- [x] Outputs: `valid` (boolean), `error_message`
- [x] Mensagens de erro claras e específicas
- [x] < 50 linhas

**Verify**:
```yaml
# Testar com input inválido:
- uses: ./.github/actions/validate-inputs
  with:
    technology: python  # Deve falhar
```

---

### T9: Criar workflow Orchestrator ✅

**What**: Criar workflow orquestrador que coordena os workflows compostos  
**Where**: `.github/workflows/orchestrator.yml`  
**Depends on**: T4, T5, T6, T7, T8  
**Reuses**: Usa todos os workflows compostos como dependências

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [x] Workflow com `workflow_call` trigger
- [x] Inputs: todos os inputs do pipeline atual (`reusable-ecs-pipeline.yml`)
- [x] Secrets: todos os secrets do pipeline atual
- [x] Job `validate` que chama action de validação
- [x] Job `build` que chama `composites/build.yml` com `needs: [validate]`
- [x] Job `test` que chama `composites/test.yml` com `needs: [build]`
- [x] Job `docker` que chama `composites/docker.yml` com `needs: [test]`
- [x] Job `deploy` que chama `composites/deploy.yml` com `needs: [docker]`
- [x] Input `deploy_only` que pula build/test/docker
- [x] Outputs: todos os outputs do pipeline atual (compatibility)
- [x] < 100 linhas

**Verify**:
```yaml
# Testar end-to-end:
jobs:
  test-orchestrator:
    uses: ./.github/workflows/orchestrator.yml
    with:
      technology: dotnet
      # ... demais inputs
```

---

### T10: Atualizar pipeline original para usar Orchestrator ✅

**What**: Modificar `reusable-ecs-pipeline.yml` para internamente chamar o orchestrator  
**Where**: `.github/workflows/reusable-ecs-pipeline.yml`  
**Depends on**: T9  
**Reuses**: Mantém interface externa idêntica

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [x] Todos os inputs mantidos (interface externa inalterada)
- [x] Todos os secrets mantidos
- [x] Todos os outputs mantidos
- [x] Internamente delega para `orchestrator.yml`
- [x] Nenhuma breaking change para consumidores
- [x] Comentário indicando deprecation futuro

**Verify**:
```bash
# Aplicação existente deve continuar funcionando sem alterações
# Comparar outputs antes/depois da mudança
```

---

### T11: Criar testes de integração ✅

**What**: Criar workflow de teste que valida os workflows compostos isoladamente  
**Where**: `.github/workflows/test-composites.yml`  
**Depends on**: T9  
**Reuses**: None

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [x] Workflow disparado via `workflow_dispatch`
- [x] Job que testa `build.yml` isoladamente com .NET
- [x] Job que testa `build.yml` isoladamente com Node
- [x] Job que testa `test.yml` com `skip_tests: true`
- [x] Job que testa `docker.yml` com `push: false`
- [x] Todos os jobs independentes com `continue-on-error: false`
- [x] Summary com resultados de cada teste

**Verify**:
```bash
# Executar via GitHub Actions UI ou gh cli:
gh workflow run test-composites.yml
```

---

### T12: Documentar workflows compostos

**What**: Criar documentação de uso dos workflows compostos no README  
**Where**: `.github/workflows/composites/README.md`  
**Depends on**: T9  
**Reuses**: None

**Tools**:
- MCP: NONE
- Skill: docs-writer

**Done when**:
- [x] README com overview da arquitetura
- [x] Documentação de cada workflow composto com inputs/outputs
- [x] Exemplos de uso para cada workflow
- [x] Guia de migração do pipeline atual para orchestrator
- [x] Seção de troubleshooting comum

**Verify**:
```bash
cat .github/workflows/composites/README.md
# Deve conter seções: Overview, Workflows, Examples, Migration, Troubleshooting
```

---

### T13: Adicionar workflow para nova tecnologia (validação de extensibilidade)

**What**: Criar arquivo de configuração para tecnologia fictícia para validar extensibilidade  
**Where**: `.github/tech-configs/custom-example.yml`  
**Depends on**: T4, T5  
**Reuses**: Estrutura de `dotnet.yml` e `node.yml`

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [x] Arquivo `custom-example.yml` criado seguindo schema
- [x] Comentários explicando como criar nova tecnologia
- [x] build.yml e test.yml funcionam com tecnologia customizada
- [x] Documentado no README

**Verify**:
```yaml
# build.yml deve funcionar com:
with:
  technology: custom-example
```

---

## Parallel Execution Map

```
Phase 1 (Sequential):
  T1 ──→ T2 ──→ T3

Phase 2 (Parallel):
  T3 complete, then:
    ├── T4 [P] (build.yml)
    ├── T5 [P] (test.yml)     } Can run simultaneously
    ├── T6 [P] (docker.yml)   }
    ├── T7 [P] (deploy.yml)   }
    └── T8 [P] (validate action)

Phase 3 (Sequential):
  T4-T8 complete, then:
    T9 ──→ T10 ──→ T11

Phase 4 (Parallel after T11):
  T11 complete, then:
    ├── T12 [P] (docs)
    └── T13 [P] (extensibility test)
```

---

## Task Granularity Check

| Task | Scope | Status |
|------|-------|--------|
| T1: Criar estrutura de diretórios | 2 diretórios | ✅ Granular |
| T2: Config dotnet.yml | 1 arquivo | ✅ Granular |
| T3: Config node.yml | 1 arquivo | ✅ Granular |
| T4: build.yml | 1 workflow | ✅ Granular |
| T5: test.yml | 1 workflow | ✅ Granular |
| T6: docker.yml | 1 workflow | ✅ Granular |
| T7: deploy.yml | 1 workflow (maior, mas coeso) | ✅ OK |
| T8: validate-inputs action | 1 action | ✅ Granular |
| T9: orchestrator.yml | 1 workflow | ✅ Granular |
| T10: Atualizar pipeline original | 1 arquivo (modificação) | ✅ Granular |
| T11: Testes de integração | 1 workflow | ✅ Granular |
| T12: Documentação | 1 arquivo | ✅ Granular |
| T13: Extensibilidade | 1 arquivo + validação | ✅ Granular |

---

## Dependencies Summary

```
T1 (estrutura)
├── T2 (dotnet.yml)
│   ├── T4 (build.yml)
│   └── T5 (test.yml)
├── T3 (node.yml)
│   ├── T4 (build.yml)
│   └── T5 (test.yml)
├── T6 (docker.yml)
├── T7 (deploy.yml)
└── T8 (validate action)
    └── T9 (orchestrator)
        └── T10 (atualizar pipeline)
            └── T11 (testes)
                ├── T12 (docs)
                └── T13 (extensibilidade)
```

---

## Estimated Effort

| Phase | Tasks | Estimated Time |
|-------|-------|---------------|
| Phase 1 | T1-T3 | ~30 min |
| Phase 2 | T4-T8 | ~2-3 hours |
| Phase 3 | T9-T11 | ~1-2 hours |
| Phase 4 | T12-T13 | ~1 hour |
| **Total** | **13 tasks** | **~5-7 hours** |
