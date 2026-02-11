# Workflows Compostos

Documentação técnica detalhada dos 4 workflows compostos do pipeline CI/CD.

## Visão Geral

Os workflows compostos são blocos reutilizáveis que encapsulam funcionalidades específicas do pipeline:

- **composite-build.yml**: Compilação multi-tecnologia (dotnet, node)
- **composite-test.yml**: Execução de testes com cobertura
- **composite-docker.yml**: Build e push de imagens Docker para ECR
- **composite-deploy.yml**: Deploy para ECS Fargate com ALB (APIs)

## composite-build

Workflow de compilação agnóstica a tecnologia, suportando .NET e Node.js.

### Inputs

| Nome | Tipo | Obrigatório | Padrão | Descrição |
|------|------|-------------|--------|-----------|
| `technology` | string | ❌ | `dotnet` | Tecnologia: dotnet ou node |
| `technology_version` | string | ❌ | `` | Versão do SDK (8.0 para .NET, 20 para Node) |
| `working_directory` | string | ❌ | `src` | Diretório do código fonte |
| `project_name` | string | ❌ | `` | Nome do projeto (.NET .csproj) |
| `build_args` | string | ❌ | `` | Argumentos adicionais de build |

### Outputs

| Nome | Descrição |
|------|-----------|
| `artifact_path` | Caminho para artefatos de build |
| `artifact_name` | Nome do artefato para download |
| `build_success` | Se o build foi concluído com sucesso |

### Secrets

Nenhum secret obrigatório.

### Exemplo de Uso

```yaml
jobs:
  build:
    uses: ./.github/workflows/composite-build.yml
    with:
      technology: dotnet
      technology_version: "8.0"
      working_directory: src
      project_name: MinhaApi
      build_args: "--verbosity normal"
```

---

## composite-test

Workflow de teste agnóstico a tecnologia, suportando skip de testes e geração de cobertura.

### Inputs

| Nome | Tipo | Obrigatório | Padrão | Descrição |
|------|------|-------------|--------|-----------|
| `technology` | string | ❌ | `dotnet` | Tecnologia: dotnet ou node |
| `technology_version` | string | ❌ | `` | Versão do SDK |
| `working_directory` | string | ❌ | `src` | Diretório do código fonte |
| `skip_tests` | boolean | ❌ | `false` | Pular execução de testes |
| `test_args` | string | ❌ | `` | Argumentos adicionais de teste |

### Outputs

| Nome | Descrição |
|------|-----------|
| `tests_passed` | Se os testes passaram |
| `coverage_report_path` | Caminho para relatório de cobertura |

### Secrets

Nenhum secret obrigatório.

### Exemplo de Uso

```yaml
jobs:
  test:
    uses: ./.github/workflows/composite-test.yml
    with:
      technology: node
      technology_version: "20"
      working_directory: src
      skip_tests: false
      test_args: "--coverage --ci"
    needs: build
```

---

## composite-docker

Workflow de build e push de imagem Docker para Amazon ECR.

### Inputs

| Nome | Tipo | Obrigatório | Padrão | Descrição |
|------|------|-------------|--------|-----------|
| `ecr_repo` | string | ✅ | - | Nome do repositório ECR |
| `service_type` | string | ✅ | - | Tipo do serviço: api ou worker |
| `push` | boolean | ❌ | `true` | Fazer push para ECR |
| `dockerfile_path` | string | ❌ | `` | Caminho para Dockerfile |
| `use_template_dockerfile` | boolean | ❌ | `true` | Usar Dockerfile do repo de templates |
| `templates_repo` | string | ❌ | `` | Repositório de templates |
| `templates_ref` | string | ❌ | `main` | Branch/tag do repo de templates |
| `project_name` | string | ❌ | `` | Nome do projeto (build arg para .NET) |
| `aws_region` | string | ❌ | `us-east-1` | Região AWS |
| `ecr_registry` | string | ❌ | `` | URL do registry ECR (auto-detectado) |
| `image_tags` | string | ❌ | `` | Tags adicionais (separadas por vírgula) |
| `environment` | string | ✅ | - | Ambiente (dev\|qa\|sbx\|prd) |

### Outputs

| Nome | Descrição |
|------|-----------|
| `image_digest` | Digest da imagem |
| `image_tag` | Tag primária da imagem (SHA) |
| `full_image_uri` | URI completo da imagem no ECR |
| `registry` | Registry ECR utilizado |

### Secrets

| Nome | Obrigatório | Descrição |
|------|-------------|-----------|
| `AWS_ACCESS_KEY_ID` | ✅ | Chave de acesso AWS |
| `AWS_SECRET_ACCESS_KEY` | ✅ | Chave secreta AWS |

### Exemplo de Uso

```yaml
jobs:
  docker:
    uses: ./.github/workflows/composite-docker.yml
    with:
      ecr_repo: minha-api
      service_type: api
      environment: dev
      project_name: MinhaApi
      image_tags: "latest,v1.0.0"
    secrets:
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    needs: [build, test]
```

---

## composite-deploy

Workflow de deploy para Amazon ECS Fargate com suporte a Application Load Balancer.

### Inputs

#### Obrigatórios

| Nome | Tipo | Descrição |
|------|------|-----------|
| `image_uri` | string | URI completo da imagem no ECR |
| `ecs_service` | string | Nome do serviço ECS |
| `environment` | string | Ambiente (dev\|qa\|sbx\|prd) |
| `service_type` | string | Tipo do serviço: api ou worker |

#### AWS & ECS Configuration

| Nome | Tipo | Padrão | Descrição |
|------|------|--------|-----------|
| `aws_region` | string | `us-east-1` | Região AWS |
| `ecs_cluster` | string | `` | Nome do cluster ECS |
| `ecs_task_execution_role_arn` | string | `` | ARN da role de execução |
| `ecs_task_role_arn` | string | `` | ARN da role da task |

#### Task Definition

| Nome | Tipo | Padrão | Descrição |
|------|------|--------|-----------|
| `task_cpu` | string | `256` | Unidades de CPU (256, 512, 1024, 2048, 4096) |
| `task_memory` | string | `512` | Memória em MB |
| `container_name` | string | `app` | Nome do container |
| `container_port` | string | `80` | Porta do container |
| `container_environment` | string | `` | Array JSON de variáveis de ambiente |
| `container_secrets` | string | `` | Array JSON de secrets |

#### Network Configuration

| Nome | Tipo | Padrão | Descrição |
|------|------|--------|-----------|
| `subnet_ids` | string | `` | IDs das subnets (separados por vírgula) |
| `security_group_ids` | string | `` | IDs dos security groups (separados por vírgula) |
| `assign_public_ip` | string | `DISABLED` | Atribuir IP público (ENABLED/DISABLED) |

#### Load Balancer (apenas APIs)

| Nome | Tipo | Padrão | Descrição |
|------|------|--------|-----------|
| `create_target_group_and_listener` | boolean | `false` | Criar/usar target group e listener |
| `load_balancer_arn` | string | `` | ARN do ALB |
| `target_group_name` | string | `` | Nome do target group |
| `target_group_port` | string | `80` | Porta do target group |
| `target_group_protocol` | string | `HTTP` | Protocolo do target group |
| `target_group_health_check_path` | string | `/` | Caminho do health check |
| `listener_port` | string | `80` | Porta do listener |
| `listener_protocol` | string | `HTTP` | Protocolo do listener |

### Outputs

| Nome | Descrição |
|------|-----------|
| `service_arn` | ARN do serviço ECS |
| `task_definition_arn` | ARN da task definition registrada |
| `deploy_metadata_artifact` | Nome do artefato com metadados do deploy |

### Secrets

| Nome | Obrigatório | Descrição |
|------|-------------|-----------|
| `AWS_ACCESS_KEY_ID` | ✅ | Chave de acesso AWS |
| `AWS_SECRET_ACCESS_KEY` | ✅ | Chave secreta AWS |
| `ECS_CLUSTER` | ❌ | Nome do cluster ECS |
| `ECS_TASK_EXECUTION_ROLE_ARN` | ❌ | ARN da role de execução |

**Config de deploy**: Os valores de `ecs_cluster`, `ecr_registry`, `subnet_ids`, `security_group_ids`, `load_balancer_name`, etc. costumam vir de um job `prepare` que lê a variável JSON `vars.{ENV}_CONFIG_DEPLOY` na organização, faz parse com `jq` e expõe via outputs. Secrets AWS vêm de `needs.prepare.outputs.aws_access_key` / `aws_secret_key`. Ver [deploy-env-pattern.md](deploy-env-pattern.md) e [organization-variables.md](organization-variables.md).

### Exemplo de Uso - API

```yaml
jobs:
  deploy:
    uses: ./.github/workflows/composite-deploy.yml
    with:
      image_uri: ${{ needs.docker.outputs.full_image_uri }}
      ecs_service: minha-api-dev
      environment: dev
      service_type: api
      task_cpu: "512"
      task_memory: "1024"
      container_port: "8080"
      create_target_group_and_listener: true
      load_balancer_arn: "arn:aws:elasticloadbalancing:us-east-1:123456789012:loadbalancer/app/dev-alb/50dc6c495c0c9188"
      target_group_name: "minha-api-dev-tg"
      target_group_health_check_path: "/health"
    secrets:
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    needs: docker
    environment: dev
```

### Exemplo de Uso - Worker

```yaml
jobs:
  deploy:
    uses: ./.github/workflows/composite-deploy.yml
    with:
      image_uri: ${{ needs.docker.outputs.full_image_uri }}
      ecs_service: meu-worker-dev
      environment: dev
      service_type: worker
      task_cpu: "1024"
      task_memory: "2048"
      desired_count: "2"
    secrets:
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    needs: docker
    environment: dev
```

## Validação de Inputs

Todos os workflows utilizam validações automáticas:

- **Technology**: Deve ser `dotnet` ou `node`
- **Environment**: Deve ser `dev`, `qa`, `sbx` ou `prd`  
- **Service Type**: Deve ser `api` ou `worker`

Mensagens de erro são padronizadas e incluem os valores aceitos.

## Próximos Passos

- Para config de deploy (variável JSON por ambiente e secrets na organização): Ver [organization-variables.md](organization-variables.md) e [deploy-env-pattern.md](deploy-env-pattern.md)
- Para uso de environments no repositório (aprovações, wait timer): Ver [environments.md](environments.md)
- Para diagramas visuais: Ver [diagramas.md](diagramas.md)
- Para guia de adaptação: Ver [adaptacao.md](adaptacao.md)