# CI/CD para AWS ECS com GitHub Actions

Arquitetura escalável e reutilizável para deploy de aplicações .NET (APIs e Workers) no Amazon ECS.

**Características:**
- Template centralizado: um workflow reutilizável (Build → Test → Docker → ECR → ECS)
- **Contexto** (inbound, outbound, platform): define qual ECR recebe o push; o caller passa `ecr_registry` por contexto
- **Dockerfile padrão** no repo de templates (`build/Dockerfile.api`, `build/Dockerfile.worker`) ou Dockerfile do repo da aplicação
- Ambientes 1:1 com branch: dev, qa, sbx, prd (secrets e aprovações por ambiente)
- Rollback manual com workflow dedicado e playbook
- Zero secrets em plain text; uso de GitHub Environments

---

## Estrutura

```
github-ecs/
├── .github/
│   ├── workflows/
│   │   ├── reusable-ecs-pipeline.yml   # Template principal (build → test → docker → ecr → ecs)
│   │   └── rollback.yml                # Rollback manual reutilizável
│   ├── ENVIRONMENTS.md                  # Configuração de environments e secrets
│   ├── VERSIONING.md                   # Estratégia de tags e histórico
│   └── ROLLBACK-PLAYBOOK.md            # Playbook de rollback
├── build/
│   ├── Dockerfile.api                  # Dockerfile genérico para APIs (uso com use_default_dockerfile=true)
│   └── Dockerfile.worker               # Dockerfile genérico para Workers
├── workflows/
│   └── dotnet-ecs-deploy.yml           # Workflow legado (DEV/PRD em 2 jobs)
├── example/                             # Uma pasta de exemplo
│   ├── README.md                       # Como usar os exemplos
│   ├── .github/workflows/
│   │   └── rollback.yml.example        # Exemplo de rollback manual
│   ├── inbound-nfe-api-envioxml/       # Projeto .NET API de exemplo
│   │   ├── src/
│   │   └── .github/workflows/deploy.yml
│   └── inbound-nfe-wrk-processaxml/    # Projeto .NET Worker de exemplo
│       ├── src/
│       └── .github/workflows/deploy.yml
└── README.md
```

---

## Contexto (inbound, outbound, platform)

O pipeline recebe um **contexto** e um **ecr_registry** para esse contexto. Assim, cada aplicação pode fazer push para o ECR correto (ex.: registry do inbound, outbound ou platform).

| Contexto   | Uso típico | Variável no environment (ex.) |
|------------|------------|-------------------------------|
| `inbound`  | Serviços de entrada | `ECR_REGISTRY_INBOUND` |
| `outbound` | Serviços de saída  | `ECR_REGISTRY_OUTBOUND` |
| `platform` | Serviços de plataforma | `ECR_REGISTRY_PLATFORM` |

No workflow da aplicação, passe `context` e `ecr_registry: ${{ vars.ECR_REGISTRY_INBOUND }}` (ou a variável do contexto desejado).

---

## Dockerfile padrão vs do repositório

- **Padrão** (`use_default_dockerfile: true`): o pipeline faz checkout do repo de templates e usa `build/Dockerfile.api` ou `build/Dockerfile.worker`. Obrigatório informar `project_name` (nome do .csproj) e `templates_repo`.
- **Do repositório** (`use_default_dockerfile: false`): usa o Dockerfile do repo da aplicação (`dockerfile_path` ou `Dockerfile.<service_type>` ou `Dockerfile`).

---

## Mapeamento branch → ambiente

| Branch | GitHub Environment | Aprovação |
|--------|--------------------|-----------|
| `dev` | `dev` | Não |
| `qa` | `qa` | Não |
| `sbx` | `sbx` | Sim (Required reviewers) |
| `prd` | `prd` | Sim (Required reviewers) |

Cada branch usa apenas os secrets do environment correspondente.

---

## Fluxo do pipeline

1. **Build**: Restore .NET (com cache NuGet).
2. **Test**: `dotnet test`; falha interrompe o pipeline.
3. **Docker Build**: Imagem com Dockerfile padrão (build/) ou do repo da app; build-arg `PROJECT_NAME` quando uso padrão.
4. **Push ECR**: Tags `sha`, `branch`, `context` e timestamp; push para o registry informado (`ecr_registry`).
5. **Deploy ECS**: `update-service --force-new-deployment` e `wait services-stable`.
6. **Deploy metadata**: Artifact com digest, tag, timestamp (histórico).
7. **Health check** (opcional): Polling HTTP na URL configurada.

---

## Guia de implementação

### 1. Configurar GitHub Environments (org ou repo da app)

Crie os environments `dev`, `qa`, `sbx`, `prd` e, em cada um:

- **Environment variables**: `ECR_REGISTRY_INBOUND`, `ECR_REGISTRY_OUTBOUND`, `ECR_REGISTRY_PLATFORM` (URL do registry ECR por contexto, ex.: `123456789.dkr.ecr.us-east-1.amazonaws.com`).
- **Environment secrets**: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `ECS_CLUSTER`, `ECS_SERVICE`.

Detalhes em [.github/ENVIRONMENTS.md](.github/ENVIRONMENTS.md).

### 2. Workflow no repositório da aplicação

Use como base os exemplos em [example/inbound-nfe-api-envioxml/.github/workflows/deploy.yml](example/inbound-nfe-api-envioxml/.github/workflows/deploy.yml) ou [example/inbound-nfe-wrk-processaxml/.github/workflows/deploy.yml](example/inbound-nfe-wrk-processaxml/.github/workflows/deploy.yml):

- `deployment_name`: nome do deployment (ECR repo e ECS service).
- `service_type`: `api` ou `worker`.
- `context`: `inbound`, `outbound` ou `platform`.
- `ecr_registry`: variável do environment para o contexto (ex.: `${{ vars.ECR_REGISTRY_INBOUND }}`).
- `use_default_dockerfile`: `true` para usar `build/Dockerfile.api` ou `build/Dockerfile.worker` do repo de templates; `false` para usar Dockerfile do repo da app.
- `templates_repo` e `project_name`: obrigatórios quando `use_default_dockerfile: true`.

Substitua `SEU_ORG/github-ecs` pelo repositório que contém os workflows.

### 3. Pré-requisitos na AWS

- **ECR**: Repositório com nome igual ao `deployment_name` (ex.: `inbound-nfe-api-envioxml`).
- **Task Definition** e **ECS Service**: configurados para a imagem no ECR do contexto.

### 4. Testar

Push nas branches `dev`, `qa`, `sbx` ou `prd`; sbx e prd exigirão aprovação se configurada.

---

## Rollback

- **Manual**: Workflow **Rollback ECS** (Actions > Run workflow). Exemplo: [example/.github/workflows/rollback.yml.example](example/.github/workflows/rollback.yml.example). Em `prd`, o environment exige aprovação.
- **Histórico**: Artifact `deploy.json` em cada deploy; use a tag desejada no rollback.
- **Playbook**: [.github/ROLLBACK-PLAYBOOK.md](.github/ROLLBACK-PLAYBOOK.md).
- **Versionamento**: [.github/VERSIONING.md](.github/VERSIONING.md).

---

## Variáveis do template (reusable-ecs-pipeline)

### Inputs obrigatórios

| Input | Descrição |
|-------|-----------|
| `deployment_name` | Nome do deployment (ECR repo e ECS service) |
| `service_type` | `api` ou `worker` |
| `context` | `inbound`, `outbound` ou `platform` (define qual ECR usar) |
| `ecr_registry` | URL do registry ECR para este contexto (ex.: vars.ECR_REGISTRY_INBOUND) |
| `environment` | `dev`, `qa`, `sbx` ou `prd` (caller passa `github.ref_name`) |

### Inputs opcionais

| Input | Padrão |
|-------|--------|
| `use_default_dockerfile` | `true` (usa build/Dockerfile.* do repo de templates) |
| `templates_repo`, `templates_ref` | Repo e branch dos templates (quando use_default_dockerfile=true) |
| `project_name` | Nome do .csproj (obrigatório quando use_default_dockerfile=true) |
| `dockerfile_path` | Caminho do Dockerfile no repo da app (quando use_default_dockerfile=false) |
| `dotnet_version`, `working_directory`, `aws_region` | 8.0, src, us-east-1 |
| `ecs_cluster`, `ecs_service`, `ecr_repository` | Usam secrets do environment quando vazios |
| `enable_health_check`, `health_check_url` | Health check pós-deploy (opcional) |

### Secrets (por environment)

| Secret | Descrição |
|--------|-----------|
| `AWS_ACCESS_KEY_ID` | Access Key AWS |
| `AWS_SECRET_ACCESS_KEY` | Secret Key AWS |
| `ECS_CLUSTER` | Nome do cluster ECS |
| `ECS_SERVICE` | Nome do service ECS |

---

## Troubleshooting

- **Deployment não atualiza**: `aws ecs describe-services --cluster CLUSTER --services SERVICE`
- **Logs**: `aws logs tail /ecs/APP_NAME --follow`
- **Rollback manual**: Workflow Rollback ECS com tag/SHA da imagem desejada.

---

## Boas práticas

- Segregação de ambientes: um environment por branch; secrets por environment.
- Contexto + `ecr_registry`: um ECR por contexto (inbound/outbound/platform).
- Dockerfile padrão em `build/` para padronizar; opção de Dockerfile customizado por app.
- GitHub Environments com Required reviewers em sbx e prd.
- Tags ECR: sha, branch, context e timestamp.
- Zero secrets em plain text nos YAMLs.
