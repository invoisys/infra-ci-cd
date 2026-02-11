# Configuração de Environments

Guia para configurar a **config de deploy** (centralizada na organização) e o uso de **environments** no repositório (aprovações, wait timers, restrição de branches).

## Modelo recomendado: config na organização

A **configuração de deploy** (ECR, ECS, rede, load balancer) fica na **organização** GitHub, não por environment no repositório:

1. **Uma variável por ambiente (JSON)**  
   Nome: `{ENV}_CONFIG_DEPLOY` (ex.: `SBX_CONFIG_DEPLOY`, `PRD_CONFIG_DEPLOY`).  
   Valor: JSON com as chaves `ecr_registry`, `ecs_cluster`, `ecs_task_execution_role_arn`, `ecs_task_role_arn`, `subnet_ids`, `security_group_ids`, `load_balancer_name`.  
   Schema e exemplos: [organization-variables.md](organization-variables.md).

2. **Secrets por ambiente**  
   Na organização: `{ENV}_AWS_ACCESS_KEY_ID`, `{ENV}_AWS_SECRET_ACCESS_KEY` (ex.: `SBX_AWS_ACCESS_KEY_ID`, `SBX_AWS_SECRET_ACCESS_KEY`).

3. **Resolução no workflow**  
   O job `prepare` do repositório usa o ambiente (branch) para escolher qual variável e quais secrets ler, faz parse do JSON com `jq` e escreve os valores nos outputs. Os jobs `docker` e `deploy` consomem `needs.prepare.outputs.*`.  
   Template do step: [deploy-env-pattern.md](deploy-env-pattern.md).

Variáveis **específicas da aplicação** (filas SQS, URLs de API, etc.) continuam no repositório ou na organização com escopo ao repositório; não entram no JSON de deploy.

## Quando usar environments no repositório

Crie **environments** no repositório (`Settings > Environments`: `dev`, `qa`, `sbx`, `prd`) quando precisar de:

- **Approval gates**: exigir aprovação de um ou mais revisores antes do deploy (ex.: `prd`).
- **Wait timer**: tempo de espera antes do deploy.
- **Restrição de branches**: permitir deploy apenas em branches específicas.

Os **valores de deploy** (ECR, ECS, rede) vêm da organização (`CONFIG_DEPLOY` + secrets). O `environment:` no workflow é usado para acionar as regras de proteção do GitHub (approval, wait, branch), não para definir ECR_REGISTRY, ECS_CLUSTER, etc.

## Mapa de ambientes

O pipeline usa o nome da **branch** para decidir qual config da organização ler (`ref_name` → `vars.{ENV}_CONFIG_DEPLOY`). Convenção típica:

| Ambiente | Prefixo vars/secrets | Branch típica | Descrição |
|----------|---------------------|---------------|-----------|
| dev | `DEV_` | `dev` | Desenvolvimento |
| qa | `QA_` | `qa` | Qualidade / homologação |
| sbx | `SBX_` | `sbx` | Sandbox |
| prd | `PRD_` | `prd`, `main` | Produção |

## Configuração no GitHub

### 1. Organização (obrigatório para deploy)

- **Variables**: criar `DEV_CONFIG_DEPLOY`, `QA_CONFIG_DEPLOY`, `SBX_CONFIG_DEPLOY`, `PRD_CONFIG_DEPLOY` com JSON conforme [organization-variables.md](organization-variables.md). Dar acesso aos repositórios que farão deploy.
- **Secrets**: criar `{ENV}_AWS_ACCESS_KEY_ID` e `{ENV}_AWS_SECRET_ACCESS_KEY` para cada ambiente e dar acesso aos repositórios.

### 2. Repositório – environments (opcional)

Se usar approval/wait timer/restrição de branch:

- Acesse `Settings > Environments` no repositório.
- Crie `dev`, `qa`, `sbx`, `prd` conforme necessidade.
- Configure **Protection rules**, **Wait timer**, **Deployment branch** onde fizer sentido (ex.: `prd` com 1 reviewer e wait 5 min).

## Variáveis e secrets – referência rápida

| Onde | O quê | Exemplos |
|------|-------|----------|
| **Organização** | Config de deploy (JSON) | `SBX_CONFIG_DEPLOY`, `PRD_CONFIG_DEPLOY` |
| **Organização** | Credenciais AWS por ambiente | `SBX_AWS_ACCESS_KEY_ID`, `SBX_AWS_SECRET_ACCESS_KEY` |
| **Repositório** | Config específica da aplicação | `SQS_QUEUE_URL`, `API_PROXY_URL`, `ABBYY_SECRET_ID` |

Detalhes do schema JSON e lista completa: [organization-variables.md](organization-variables.md).

## Permissões IAM necessárias

As chaves AWS configuradas nos secrets da organização devem ter permissão para ECR (push de imagem), ECS (register task definition, update service), VPC (subnets, security groups) e, para APIs, ALB (target group, listener). Exemplos de políticas estão em [organization-variables.md](organization-variables.md) e na documentação AWS.

## Troubleshooting

### Unknown environment

O job `prepare` falha com "Invalid environment" quando a branch não está mapeada no `case` (ex.: branch `staging` sem caso correspondente). Adicione um caso no `case` e crie `STAGING_CONFIG_DEPLOY` e os secrets na organização.

### Variável ou secret vazio

Confirme que a variável `{ENV}_CONFIG_DEPLOY` existe, está acessível ao repositório e o valor é um JSON válido. Confirme que os secrets `{ENV}_AWS_ACCESS_KEY_ID` e `{ENV}_AWS_SECRET_ACCESS_KEY` existem e o repositório tem acesso. Ver [troubleshooting-env-vars.md](troubleshooting-env-vars.md).

### ECR / ECS / rede

Erros como "ECR repository does not exist", "ECS cluster not found", "Invalid subnet" indicam que os **valores dentro do JSON** estão incorretos ou desatualizados. Ajuste o conteúdo da variável `{ENV}_CONFIG_DEPLOY` na organização e valide com `jq` (ver troubleshooting-env-vars.md).

## Próximos passos

- [Organization Variables](organization-variables.md) – Schema do JSON e convenção de nomes
- [Deploy Env Pattern](deploy-env-pattern.md) – Template do job prepare com CONFIG_DEPLOY + jq
- [Troubleshooting env vars](troubleshooting-env-vars.md) – Validação do JSON e problemas comuns
- [Workflows](workflows.md) – Detalhes dos workflows compostos
- [Diagramas](diagramas.md) – Fluxos e mapeamento branch → environment
