# Configuração de GitHub Environments e Secrets

Cada ambiente (dev, qa, sbx, prd) deve ter seus próprios **Secrets** e **Variables** no GitHub. O pipeline usa o environment definido pela branch e o **contexto** (inbound, outbound, platform) para escolher o ECR via variável.

## Mapeamento branch → environment

| Branch (app) | GitHub Environment | Aprovação |
|--------------|--------------------|-----------|
| `dev` | `dev` | Não |
| `qa` | `qa` | Não |
| `sbx` | `sbx` | Sim (Required reviewers) |
| `prd` | `prd` | Sim (Required reviewers) |

## Contexto e ECR

O pipeline recebe `context` (inbound, outbound ou platform) e `ecr_registry`. O caller passa o registry a partir das variáveis do environment:

- **inbound**: `vars.ECR_REGISTRY_INBOUND`
- **outbound**: `vars.ECR_REGISTRY_OUTBOUND`
- **platform**: `vars.ECR_REGISTRY_PLATFORM`

Assim, cada ambiente pode ter um registry por contexto (ou o mesmo registry para todos, se for o caso).

## Como configurar

### 1. Criar os 4 environments

Em **Settings > Environments** do repositório da aplicação (ou da organização):

1. Crie os environments: `dev`, `qa`, `sbx`, `prd`.
2. Em **sbx** e **prd**: em **Deployment protection rules**, marque **Required reviewers** e adicione os aprovadores.
3. Em cada environment, em **Deployment branches**, restrinja à branch correspondente (ex.: env `prd` apenas branch `prd`).

### 2. Environment variables (por ambiente)

Em cada environment, em **Environment variables**:

| Variable | Obrigatório | Descrição |
|----------|-------------|-----------|
| `ECR_REGISTRY_INBOUND` | Sim* | URL do registry ECR para contexto inbound (ex.: `123456789.dkr.ecr.us-east-1.amazonaws.com`) |
| `ECR_REGISTRY_OUTBOUND` | Sim* | URL do registry ECR para contexto outbound |
| `ECR_REGISTRY_PLATFORM` | Sim* | URL do registry ECR para contexto platform |

\* Pode usar o mesmo valor nos três se todos os contextos usarem o mesmo registry.

### 3. Environment secrets (por ambiente)

Em cada environment, em **Environment secrets**:

| Secret | Obrigatório | Descrição |
|--------|-------------|-----------|
| `AWS_ACCESS_KEY_ID` | Sim | Access Key da conta/role AWS do ambiente |
| `AWS_SECRET_ACCESS_KEY` | Sim | Secret Key correspondente |
| `ECS_CLUSTER` | Não* | Nome do cluster ECS (ex.: `cluster-dev`); pode ser passado como input `ecs_cluster` |
| `ECS_SERVICE` | Não* | Nome do service ECS (ex.: `inbound-nfe-api-envioxml`); pode ser passado como input `ecs_service` |
| `ECS_TASK_EXECUTION_ROLE_ARN` | Não* | ARN da role de execução da task; pode ser passado como input `ecs_task_execution_role_arn` |

\* Para o primeiro deploy (service ainda não existe), informe **cluster** e **service** via inputs ou secrets. A **role de execução** é obrigatória em todo deploy (input ou secret).

### 4. Configuração do serviço ECS (inputs do workflow)

O pipeline suporta criar ou atualizar o service ECS:

- **Service já existe no cluster**: atualiza com nova task definition (imagem do build).
- **Service não existe**: cria a task definition e o service (com rede e, para API, Load Balancer).

**Obrigatório para todo deploy:**

- `ecs_task_execution_role_arn` (ou secret `ECS_TASK_EXECUTION_ROLE_ARN`)
- `ecs_cluster` (ou secret `ECS_CLUSTER`)
- Nome do service: `ecs_service` ou `deployment_name` (ou secret `ECS_SERVICE`)

**Para criar o service (primeiro deploy):**

- `subnet_ids`: IDs das subnets separados por vírgula
- `security_group_ids`: IDs dos security groups separados por vírgula

**Para API (service_type=api) com Load Balancer:**

- `load_balancer_target_group_arn`: ARN do target group do ALB (o path, ex. `/api/nfe/*`, é configurado na listener rule do ALB que aponta para esse target group)
- `container_name`: nome do container na task definition (default: `app`)
- `container_port`: porta exposta (default: `80`)
- Opcional: `load_balancer_name`, `listener_path_pattern` (referência/documentação; ex.: LB invoisys.com.br, path `/api/nfe/*`,`/api/cte/*`)

**Worker (service_type=worker):** não usa Load Balancer; não informe `load_balancer_target_group_arn`.

**Outros inputs opcionais:** `ecs_task_role_arn`, `task_cpu`, `task_memory`, `vpc_id`, `assign_public_ip`, `desired_count`.

**Logs:** o pipeline usa o log group `/ecs/<nome-do-service>`. Crie o log group no CloudWatch (ou garanta que a role de execução tenha permissão `logs:CreateLogGroup`).

### 5. Repositório de templates (este repo)

- Os **secrets** e **variables** são configurados no **repositório da aplicação** (ou na organização) que chama o workflow.
- O job do caller usa `environment: ${{ github.ref_name }}` e `secrets: inherit`; o caller passa `ecr_registry: ${{ vars.ECR_REGISTRY_INBOUND }}` (ou o contexto desejado).

### 6. Resumo de segurança

- Zero secrets em plain text nos YAMLs.
- Cada ambiente tem credenciais isoladas.
- sbx e prd exigem aprovação manual antes do deploy e do rollback em prd.
