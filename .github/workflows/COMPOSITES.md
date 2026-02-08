# Composite Workflows

Arquitetura de workflows compostos para pipelines CI/CD modulares e reutilizáveis.

## Overview

Esta arquitetura implementa o padrão de **Composite Workflows** do GitHub Actions, permitindo:

- 🔄 **Reutilização**: Workflows modulares que podem ser chamados de diferentes pipelines
- 🔧 **Flexibilidade**: Suporte a múltiplas tecnologias (.NET, Node.js, custom)
- 📦 **Composição**: Orquestração de workflows em pipelines completos
- 🎯 **Desacoplamento**: Cada etapa do pipeline é independente e testável

### Arquitetura

```
┌──────────────────────────────────────────────────────────────────┐
│                   CONSUMER WORKFLOW (ci.yml / deploy.yml)        │
│              (Orquestração direta pelo projeto consumidor)       │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐      │
│   │ build.  │───▶│ test.   │───▶│ docker. │───▶│ deploy. │      │
│   │   yml   │    │   yml   │    │   yml   │    │   yml   │      │
│   └─────────┘    └─────────┘    └─────────┘    └─────────┘      │
│        │              │              │              │            │
│        └──────────────┴──────────────┴──────────────┘            │
│                          │                                       │
│                    tech-configs/                                 │
│              ┌───────────┼───────────┐                          │
│              │           │           │                          │
│         dotnet.yml   node.yml   custom.yml                      │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Padrão de Uso

Cada projeto consumidor chama diretamente os composites que precisa, sem camadas intermediárias:

- **CI (Pull Request)**: `composite-build.yml` → `composite-test.yml`
- **Deploy (Push)**: `composite-build.yml` → `composite-test.yml` → `composite-docker.yml` → `composite-deploy.yml`

### Fluxo de Execução

1. **Build** → Compila o código e gera artefatos
2. **Test** → Executa testes automatizados
3. **Docker** → Cria imagem e publica no ECR
4. **Deploy** → Deploy no ECS Fargate

---

## Workflows

### 📦 build.yml

Workflow de build agnóstico de tecnologia.

#### Inputs

| Input | Tipo | Obrigatório | Default | Descrição |
|-------|------|-------------|---------|-----------|
| `technology` | string | ❌ | `dotnet` | Tecnologia: `dotnet` ou `node` |
| `technology_version` | string | ❌ | - | Versão do SDK (ex: `8.0` para .NET, `20` para Node) |
| `working_directory` | string | ❌ | `src` | Diretório do código fonte |
| `project_name` | string | ❌ | - | Nome do projeto .csproj (para .NET) |
| `build_args` | string | ❌ | - | Argumentos adicionais de build |

#### Outputs

| Output | Descrição |
|--------|-----------|
| `artifact_path` | Caminho para os artefatos de build |
| `artifact_name` | Nome do artefato para download |
| `build_success` | `true` se build completou com sucesso |

#### Exemplo

```yaml
jobs:
  build:
    uses: ./.github/workflows/composite-build.yml
    with:
      technology: dotnet
      technology_version: '8.0'
      working_directory: src
      project_name: MyApi
```

---

### 🧪 test.yml

Workflow de testes agnóstico de tecnologia.

#### Inputs

| Input | Tipo | Obrigatório | Default | Descrição |
|-------|------|-------------|---------|-----------|
| `technology` | string | ❌ | `dotnet` | Tecnologia: `dotnet` ou `node` |
| `technology_version` | string | ❌ | - | Versão do SDK |
| `working_directory` | string | ❌ | `src` | Diretório do código fonte |
| `skip_tests` | boolean | ❌ | `false` | Pular execução dos testes |
| `test_args` | string | ❌ | - | Argumentos adicionais de teste |

#### Outputs

| Output | Descrição |
|--------|-----------|
| `tests_passed` | `true` se todos os testes passaram |
| `coverage_report_path` | Caminho para relatório de cobertura |

#### Exemplo

```yaml
jobs:
  test:
    needs: [build]
    uses: ./.github/workflows/composite-test.yml
    with:
      technology: dotnet
      working_directory: src
      skip_tests: false
```

---

### 🐳 docker.yml

Build e push de imagem Docker para ECR.

#### Inputs

| Input | Tipo | Obrigatório | Default | Descrição |
|-------|------|-------------|---------|-----------|
| `ecr_repo` | string | ✅ | - | Nome do repositório ECR |
| `service_type` | string | ✅ | - | Tipo: `api` ou `worker` |
| `push` | boolean | ❌ | `true` | Push para ECR |
| `dockerfile_path` | string | ❌ | - | Caminho do Dockerfile |
| `use_template_dockerfile` | boolean | ❌ | `true` | Usar Dockerfile do repo de templates |
| `templates_repo` | string | ❌ | - | Repositório de templates |
| `templates_ref` | string | ❌ | `main` | Branch/tag do repo de templates |
| `project_name` | string | ❌ | - | Nome do projeto (build arg para .NET) |
| `aws_region` | string | ❌ | `us-east-1` | Região AWS |
| `ecr_registry` | string | ❌ | - | URL do registry ECR (auto-detectado) |
| `image_tags` | string | ❌ | - | Tags adicionais (vírgula-separado) |

#### Secrets

| Secret | Obrigatório | Descrição |
|--------|-------------|-----------|
| `AWS_ACCESS_KEY_ID` | ✅ | AWS Access Key |
| `AWS_SECRET_ACCESS_KEY` | ✅ | AWS Secret Access Key |
| `REPO_ACCESS_TOKEN` | ❌ | Token para acessar repo de templates |

#### Outputs

| Output | Descrição |
|--------|-----------|
| `image_digest` | Digest da imagem |
| `image_tag` | Tag primária (SHA) |
| `full_image_uri` | URI completa da imagem no ECR |
| `registry` | Registry ECR utilizado |

#### Exemplo

```yaml
jobs:
  docker:
    needs: [test]
    uses: ./.github/workflows/composite-docker.yml
    with:
      ecr_repo: my-service
      service_type: api
      project_name: MyApi
      use_template_dockerfile: true
      templates_repo: org/infra-ci-cd
    secrets:
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      REPO_ACCESS_TOKEN: ${{ secrets.REPO_ACCESS_TOKEN }}
```

---

### 🚀 deploy.yml

Deploy de imagem Docker para ECS Fargate.

#### Inputs Obrigatórios

| Input | Tipo | Descrição |
|-------|------|-----------|
| `image_uri` | string | URI completa da imagem no ECR |
| `ecs_service` | string | Nome do serviço ECS |
| `environment` | string | Ambiente: `dev`, `qa`, `sbx`, `prd` |
| `service_type` | string | Tipo: `api` ou `worker` |

#### Inputs de Configuração AWS/ECS

| Input | Tipo | Default | Descrição |
|-------|------|---------|-----------|
| `aws_region` | string | `us-east-1` | Região AWS |
| `ecs_cluster` | string | - | Nome do cluster ECS |
| `ecs_task_execution_role_arn` | string | - | ARN da role de execução |
| `ecs_task_role_arn` | string | - | ARN da role da task |
| `task_cpu` | string | `256` | CPU units (256, 512, 1024, 2048, 4096) |
| `task_memory` | string | `512` | Memória em MB |

#### Inputs de Container

| Input | Tipo | Default | Descrição |
|-------|------|---------|-----------|
| `container_name` | string | `app` | Nome do container |
| `container_port` | string | `80` | Porta do container |
| `container_environment` | string | - | JSON array de variáveis de ambiente |
| `container_secrets` | string | - | JSON array de secrets |

#### Inputs de Rede

| Input | Tipo | Default | Descrição |
|-------|------|---------|-----------|
| `subnet_ids` | string | - | IDs das subnets (vírgula-separado) |
| `security_group_ids` | string | - | IDs dos security groups (vírgula-separado) |
| `assign_public_ip` | string | `DISABLED` | `ENABLED` ou `DISABLED` |
| `desired_count` | string | `1` | Número de tasks desejadas |

#### Inputs de Load Balancer (API only)

| Input | Tipo | Default | Descrição |
|-------|------|---------|-----------|
| `create_target_group_and_listener` | boolean | `false` | Criar/usar target group e listener |
| `load_balancer_arn` | string | - | ARN do ALB |
| `load_balancer_name` | string | - | Nome do ALB |
| `listener_port` | string | `80` | Porta do listener |
| `listener_protocol` | string | `HTTP` | Protocolo do listener |
| `target_group_name` | string | - | Nome do target group |
| `target_group_port` | string | `80` | Porta do target group |
| `target_group_health_check_path` | string | `/` | Path do health check |
| `listener_rule_path_pattern` | string | - | Path pattern da regra |
| `listener_rule_host_header` | string | - | Host header da regra |
| `listener_rule_priority` | string | - | Prioridade da regra |

#### Secrets

| Secret | Obrigatório | Descrição |
|--------|-------------|-----------|
| `AWS_ACCESS_KEY_ID` | ✅ | AWS Access Key |
| `AWS_SECRET_ACCESS_KEY` | ✅ | AWS Secret Access Key |
| `ECS_CLUSTER` | ❌ | Cluster ECS (alternativa ao input) |
| `ECS_TASK_EXECUTION_ROLE_ARN` | ❌ | Role ARN (alternativa ao input) |

#### Outputs

| Output | Descrição |
|--------|-----------|
| `service_arn` | ARN do serviço ECS |
| `task_definition_arn` | ARN da task definition |

#### Exemplo

```yaml
jobs:
  deploy:
    needs: [docker]
    uses: ./.github/workflows/composite-deploy.yml
    with:
      image_uri: ${{ needs.docker.outputs.full_image_uri }}
      ecs_service: my-api-dev
      environment: dev
      service_type: api
      task_cpu: '512'
      task_memory: '1024'
      container_port: '8080'
      desired_count: '2'
    secrets:
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

---

## Exemplos de Uso

### CI — Build e Test em Pull Requests

```yaml
name: CI

on:
  pull_request:
    branches: [dev, qa, sbx, prd]

jobs:
  build:
    uses: org/infra-ci-cd/.github/workflows/composite-build.yml@main
    with:
      technology: dotnet
      technology_version: '8.0'
      working_directory: src
      project_name: MyApi

  test:
    needs: [build]
    uses: org/infra-ci-cd/.github/workflows/composite-test.yml@main
    with:
      technology: dotnet
      technology_version: '8.0'
      working_directory: src
```

### Deploy Completo — Build → Test → Docker → Deploy

```yaml
name: Deploy API

on:
  push:
    branches: [dev, qa, sbx, prd]

jobs:
  prepare:
    name: Load environment variables
    runs-on: ubuntu-latest
    environment: ${{ github.ref_name }}
    outputs:
      ecr_registry: ${{ vars.ECR_REGISTRY }}
      ecs_cluster: ${{ vars.ECS_CLUSTER }}
      ecs_task_execution_role_arn: ${{ vars.ECS_TASK_EXECUTION_ROLE_ARN }}
    steps:
      - run: echo "✅ Variáveis carregadas do environment ${{ github.ref_name }}"

  build:
    needs: [prepare]
    uses: org/infra-ci-cd/.github/workflows/composite-build.yml@main
    with:
      technology: dotnet
      technology_version: '8.0'
      working_directory: src
      project_name: MyApi

  test:
    needs: [build]
    uses: org/infra-ci-cd/.github/workflows/composite-test.yml@main
    with:
      technology: dotnet
      technology_version: '8.0'
      working_directory: src

  docker:
    needs: [test, prepare]
    uses: org/infra-ci-cd/.github/workflows/composite-docker.yml@main
    with:
      ecr_repo: my-api
      service_type: api
      project_name: MyApi
      use_template_dockerfile: true
      templates_repo: org/infra-ci-cd
      ecr_registry: ${{ needs.prepare.outputs.ecr_registry }}
    secrets:
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      REPO_ACCESS_TOKEN: ${{ secrets.REPO_ACCESS_TOKEN }}

  deploy:
    needs: [docker, prepare]
    uses: org/infra-ci-cd/.github/workflows/composite-deploy.yml@main
    with:
      image_uri: ${{ needs.docker.outputs.full_image_uri }}
      ecs_service: my-api-dev
      environment: ${{ github.ref_name }}
      service_type: api
      ecs_cluster: ${{ needs.prepare.outputs.ecs_cluster }}
      ecs_task_execution_role_arn: ${{ needs.prepare.outputs.ecs_task_execution_role_arn }}
      task_cpu: '512'
      task_memory: '1024'
      desired_count: '2'
    secrets:
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

### Pipeline Node.js

```yaml
name: Deploy Node App

on:
  push:
    branches: [dev, qa, sbx, prd]

jobs:
  build:
    uses: org/infra-ci-cd/.github/workflows/composite-build.yml@main
    with:
      technology: node
      technology_version: '20'
      working_directory: src

  test:
    needs: [build]
    uses: org/infra-ci-cd/.github/workflows/composite-test.yml@main
    with:
      technology: node
      technology_version: '20'
      working_directory: src

  docker:
    needs: [test]
    uses: org/infra-ci-cd/.github/workflows/composite-docker.yml@main
    with:
      ecr_repo: my-node-app
      service_type: api
    secrets:
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}

  deploy:
    needs: [docker]
    uses: org/infra-ci-cd/.github/workflows/composite-deploy.yml@main
    with:
      image_uri: ${{ needs.docker.outputs.full_image_uri }}
      ecs_service: my-node-app-dev
      environment: dev
      service_type: api
    secrets:
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

### Worker Service

```yaml
name: Deploy Worker

on:
  push:
    branches: [dev, qa, sbx, prd]

jobs:
  build:
    uses: org/infra-ci-cd/.github/workflows/composite-build.yml@main
    with:
      technology: dotnet
      working_directory: src
      project_name: MyWorker

  test:
    needs: [build]
    uses: org/infra-ci-cd/.github/workflows/composite-test.yml@main
    with:
      technology: dotnet
      working_directory: src

  docker:
    needs: [test]
    uses: org/infra-ci-cd/.github/workflows/composite-docker.yml@main
    with:
      ecr_repo: my-worker
      service_type: worker
      project_name: MyWorker
      use_template_dockerfile: true
      templates_repo: org/infra-ci-cd
    secrets:
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      REPO_ACCESS_TOKEN: ${{ secrets.REPO_ACCESS_TOKEN }}

  deploy:
    needs: [docker]
    uses: org/infra-ci-cd/.github/workflows/composite-deploy.yml@main
    with:
      image_uri: ${{ needs.docker.outputs.full_image_uri }}
      ecs_service: my-worker-dev
      environment: dev
      service_type: worker
    secrets:
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

---

## Guia de Migração

### De reusable-ecs-pipeline.yml / orchestrator.yml para Composites Diretos

Os arquivos `reusable-ecs-pipeline.yml` e `orchestrator.yml` foram removidos. Agora cada projeto consome os composites diretamente.

#### Antes (chamada única ao wrapper)

```yaml
jobs:
  deploy:
    uses: org/infra-ci-cd/.github/workflows/reusable-ecs-pipeline.yml@main
    with:
      ecs_service: my-api-dev
      service_type: api
      ecr_repo: my-api
      environment: dev
      dotnet_version: '8.0'
      project_name: MyApi
      working_directory: src
      use_default_dockerfile: true
      templates_repo: org/infra-ci-cd
    secrets: inherit
```

#### Depois (chamadas diretas aos composites)

Crie **dois workflows** no seu projeto:

1. **`ci.yml`** — roda em Pull Requests (apenas build + test)
2. **`deploy.yml`** — roda em push (build → test → docker → deploy)

```yaml
# ci.yml
name: CI
on:
  pull_request:
    branches: [dev, qa, sbx, prd]
jobs:
  build:
    uses: org/infra-ci-cd/.github/workflows/composite-build.yml@main
    with:
      technology: dotnet
      technology_version: '8.0'
      working_directory: src
      project_name: MyApi
  test:
    needs: [build]
    uses: org/infra-ci-cd/.github/workflows/composite-test.yml@main
    with:
      technology: dotnet
      technology_version: '8.0'
      working_directory: src
```

```yaml
# deploy.yml — veja seção "Exemplos de Uso > Deploy Completo" acima
```

#### Mapeamento de Inputs

| reusable-ecs-pipeline | Composite equivalente | Input |
|-----------------------|----------------------|-------|
| `dotnet_version` | `composite-build.yml` / `composite-test.yml` | `technology_version` |
| `use_default_dockerfile` | `composite-docker.yml` | `use_template_dockerfile` |
| `ecr_repo` | `composite-docker.yml` | `ecr_repo` |
| `ecs_service` | `composite-deploy.yml` | `ecs_service` |
| `secrets: inherit` | Cada composite | `secrets:` explícitos por composite |

#### Benefícios

- Sem camadas intermediárias — fluxo transparente e auditável
- CI e deploy em workflows separados — mais claro o que roda em cada cenário
- Controle total do encadeamento (`needs`) e passagem de outputs
- Possibilidade de compor apenas os composites necessários

---

## Adicionando Nova Tecnologia

Para adicionar suporte a uma nova tecnologia, crie um arquivo em `.github/tech-configs/`:

```yaml
# .github/tech-configs/custom-example.yml
name: custom-example
default_version: "1.0"

setup_action: actions/setup-custom@v1
setup_with:
  version: ${{ inputs.technology_version || '1.0' }}

cache:
  path: ~/.custom-cache
  key_pattern: "${{ runner.os }}-custom-${{ hashFiles('**/custom.lock') }}"

commands:
  restore: custom install
  build: custom build --production
  test: custom test

file_patterns:
  project: "**/custom.config"
  lock_file: "**/custom.lock"
```

Veja `.github/tech-configs/custom-example.yml` para um exemplo completo com comentários.

---

## Troubleshooting

### Build falha com "Unknown technology"

**Problema**: Workflow falha com mensagem "❌ Unknown technology: X"

**Solução**: Verifique se o valor de `technology` é um dos suportados: `dotnet`, `node`.

```yaml
with:
  technology: dotnet  # ✅ Correto
  technology: DOTNET  # ❌ Case-sensitive
  technology: net8    # ❌ Não suportado
```

### Docker build falha com "Dockerfile not found"

**Problema**: Workflow não encontra o Dockerfile

**Soluções**:

1. Se usando template:
   ```yaml
   with:
     use_template_dockerfile: true
     templates_repo: org/infra-ci-cd  # ← Verificar se está correto
   ```

2. Se usando Dockerfile customizado:
   ```yaml
   with:
     use_template_dockerfile: false
     dockerfile_path: ./Dockerfile.api  # ← Verificar se existe
   ```

### Deploy falha com "Invalid service_type"

**Problema**: Workflow falha com mensagem sobre service_type

**Solução**: Use apenas `api` ou `worker`:

```yaml
with:
  service_type: api     # ✅ Correto
  service_type: API     # ❌ Case-sensitive
  service_type: webapp  # ❌ Não suportado
```

### Tests não executam

**Problema**: Testes são ignorados mesmo com `skip_tests: false`

**Verificações**:

1. O workflow de build precisa completar primeiro
2. Verificar se há projetos de teste no diretório:
   - .NET: `**/*Tests.csproj`
   - Node: `jest.config.*` ou scripts de test em `package.json`

### ECR Login falha

**Problema**: Falha ao fazer login no ECR

**Verificações**:

1. Verificar se os secrets estão configurados:
   - `AWS_ACCESS_KEY_ID`
   - `AWS_SECRET_ACCESS_KEY`

2. Verificar permissões da IAM:
   - `ecr:GetAuthorizationToken`
   - `ecr:BatchGetImage`
   - `ecr:BatchCheckLayerAvailability`
   - `ecr:PutImage`

### Deploy falha com timeout

**Problema**: ECS service não estabiliza

**Verificações**:

1. Verificar health check:
   ```yaml
   with:
     target_group_health_check_path: /health  # ← Garantir que endpoint existe
   ```

2. Verificar recursos:
   ```yaml
   with:
     task_cpu: '512'     # ← Aumentar se necessário
     task_memory: '1024' # ← Aumentar se necessário
   ```

3. Verificar logs no CloudWatch:
   - Log group: `/ecs/{service_name}`
   - Procurar por erros de startup da aplicação

### Cache não funciona

**Problema**: Cache nunca é restaurado

**Verificações**:

1. Verificar se o arquivo de lock existe:
   - .NET: `*.csproj` ou `*.sln`
   - Node: `package-lock.json`

2. Branch pode ter cache isolado de outras branches

### Workflow não encontrado ao usar `uses:`

**Problema**: GitHub não encontra o workflow

**Solução**: Verificar a referência:

```yaml
# Mesmo repositório
uses: ./.github/workflows/composite-build.yml

# Outro repositório
uses: org/repo/.github/workflows/composite-build.yml@main
```

---

## Links Úteis

- [GitHub Reusable Workflows](https://docs.github.com/en/actions/using-workflows/reusing-workflows)
- [AWS ECS with GitHub Actions](https://docs.github.com/en/actions/deployment/deploying-to-amazon-elastic-container-service)
