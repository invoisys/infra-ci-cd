# Organization Variables - Convenção e Nomenclatura

## Visão Geral

Este documento define a convenção de nomenclatura para variáveis e secrets centralizados na organização GitHub (`invoisys`). A centralização permite que múltiplas aplicações compartilhem configurações de infraestrutura comuns, reduzindo duplicação e facilitando a manutenção.

A **configuração de deploy** (ECR, ECS, rede, load balancer) é definida por **uma única variável por ambiente** contendo um JSON com todas as propriedades. Os **secrets** (credenciais AWS) permanecem em variáveis separadas por ambiente.

## Convenção de Nomenclatura

### Config de deploy (variável única JSON)

```
{AMBIENTE}_CONFIG_DEPLOY
```

Onde:
- `{AMBIENTE}`: Prefixo do ambiente em uppercase (`DEV`, `QA`, `SBX`, `PRD`)
- O valor é um **JSON** com todas as propriedades de infraestrutura (ver schema abaixo).

### Secrets (credenciais)

```
{AMBIENTE}_AWS_ACCESS_KEY_ID
{AMBIENTE}_AWS_SECRET_ACCESS_KEY
```

### Exemplos

```
SBX_CONFIG_DEPLOY   → JSON com ecr_registry, ecs_cluster, subnet_ids, etc.
PRD_CONFIG_DEPLOY   → idem para produção
DEV_AWS_ACCESS_KEY_ID
SBX_AWS_SECRET_ACCESS_KEY
```

## Schema do JSON (`{ENV}_CONFIG_DEPLOY`)

Uma única variável por ambiente, por exemplo `SBX_CONFIG_DEPLOY`, com valor JSON no formato:

```json
{
  "ecr_registry": "123456789012.dkr.ecr.us-east-1.amazonaws.com",
  "ecs_cluster": "cluster-sbx",
  "ecs_task_execution_role_arn": "arn:aws:iam::123456789012:role/ecsTaskExecutionRole",
  "ecs_task_role_arn": "arn:aws:iam::123456789012:role/ecsTaskRole",
  "subnet_ids": "subnet-abc,subnet-def",
  "security_group_ids": "sg-abc,sg-def",
  "load_balancer_name": "alb-sbx-main"
}
```

### Chaves do schema

| Chave | Descrição | Exemplo | Obrigatória |
|-------|-----------|---------|-------------|
| `ecr_registry` | URL do registro ECR | `123456789012.dkr.ecr.us-east-1.amazonaws.com` | Sim (para deploy) |
| `ecs_cluster` | Nome do cluster ECS | `cluster-sbx` | Sim |
| `ecs_task_execution_role_arn` | ARN da role de execução da task | `arn:aws:iam::...:role/ecsTaskExecutionRole` | Conforme app |
| `ecs_task_role_arn` | ARN da role da task | `arn:aws:iam::...:role/ecsTaskRole` | Conforme app |
| `subnet_ids` | IDs das subnets (comma-separated) | `subnet-abc,subnet-def` | Conforme app |
| `security_group_ids` | IDs dos security groups (comma-separated) | `sg-abc,sg-def` | Conforme app |
| `load_balancer_name` | Nome do load balancer (APIs com ALB) | `alb-sbx-main` | Não (workers omitem ou usam `""`) |

- **Todas as chaves são opcionais** no JSON: aplicações que não usam `load_balancer_name` (ex.: workers) podem omitir a chave ou usar `""`.
- Valores devem ser **single-line** (sem quebras de linha) para compatibilidade com `GITHUB_OUTPUT`.
- O job `prepare` dos workflows lê `vars.{ENV}_CONFIG_DEPLOY`, faz parse com `jq` e escreve cada chave nos outputs; chaves ausentes resultam em string vazia.

## Mapeamento de Ambientes

| Ambiente | Prefixo | Branch Típico | Variável JSON | Secrets |
|----------|---------|---------------|---------------|---------|
| Development | `DEV_` | `dev` | `DEV_CONFIG_DEPLOY` | `DEV_AWS_ACCESS_KEY_ID`, `DEV_AWS_SECRET_ACCESS_KEY` |
| Quality Assurance | `QA_` | `qa` | `QA_CONFIG_DEPLOY` | `QA_AWS_ACCESS_KEY_ID`, `QA_AWS_SECRET_ACCESS_KEY` |
| Sandbox | `SBX_` | `sbx` | `SBX_CONFIG_DEPLOY` | `SBX_AWS_ACCESS_KEY_ID`, `SBX_AWS_SECRET_ACCESS_KEY` |
| Production | `PRD_` | `prd`, `main` | `PRD_CONFIG_DEPLOY` | `PRD_AWS_ACCESS_KEY_ID`, `PRD_AWS_SECRET_ACCESS_KEY` |

## Variáveis da Organização

### Variables (config de deploy)

| Nome | Descrição | Formato |
|------|-----------|---------|
| `{ENV}_CONFIG_DEPLOY` | Configuração completa de deploy do ambiente (ECR, ECS, rede, load balancer) | JSON (schema acima) |

### Secrets (sensíveis)

Credenciais e valores sensíveis, acessíveis via `${{ secrets.NOME }}`.

#### Credenciais AWS

| Nome | Descrição | Tipo |
|------|-----------|------|
| `{ENV}_AWS_ACCESS_KEY_ID` | Access Key ID da AWS | Secret |
| `{ENV}_AWS_SECRET_ACCESS_KEY` | Secret Access Key da AWS | Secret |

## Lista por Ambiente

### DEV (Development)

**Variables:**
- `DEV_CONFIG_DEPLOY` (JSON)

**Secrets:**
- `DEV_AWS_ACCESS_KEY_ID`
- `DEV_AWS_SECRET_ACCESS_KEY`

### QA (Quality Assurance)

**Variables:**
- `QA_CONFIG_DEPLOY` (JSON)

**Secrets:**
- `QA_AWS_ACCESS_KEY_ID`
- `QA_AWS_SECRET_ACCESS_KEY`

### SBX (Sandbox)

**Variables:**
- `SBX_CONFIG_DEPLOY` (JSON)

**Secrets:**
- `SBX_AWS_ACCESS_KEY_ID`
- `SBX_AWS_SECRET_ACCESS_KEY`

### PRD (Production)

**Variables:**
- `PRD_CONFIG_DEPLOY` (JSON)

**Secrets:**
- `PRD_AWS_ACCESS_KEY_ID`
- `PRD_AWS_SECRET_ACCESS_KEY`

## Como Usar no Workflow (job prepare)

O job `prepare` resolve o ambiente (branch), lê a variável `vars.{ENV}_CONFIG_DEPLOY`, grava os secrets no `GITHUB_OUTPUT` e usa `jq` para extrair as chaves do JSON para os outputs:

```yaml
- name: Resolve environment variables
  id: vars
  run: |
    ENV="${{ github.ref_name }}"
    case "$ENV" in
      sbx)  CONFIG='${{ vars.SBX_CONFIG_DEPLOY }}'
            echo "aws_access_key=${{ secrets.SBX_AWS_ACCESS_KEY_ID }}" >> $GITHUB_OUTPUT
            echo "aws_secret_key=${{ secrets.SBX_AWS_SECRET_ACCESS_KEY }}" >> $GITHUB_OUTPUT ;;
      prd)  CONFIG='${{ vars.PRD_CONFIG_DEPLOY }}'
            echo "aws_access_key=${{ secrets.PRD_AWS_ACCESS_KEY_ID }}" >> $GITHUB_OUTPUT
            echo "aws_secret_key=${{ secrets.PRD_AWS_SECRET_ACCESS_KEY }}" >> $GITHUB_OUTPUT ;;
      dev)  CONFIG='${{ vars.DEV_CONFIG_DEPLOY }}'
            echo "aws_access_key=${{ secrets.DEV_AWS_ACCESS_KEY_ID }}" >> $GITHUB_OUTPUT
            echo "aws_secret_key=${{ secrets.DEV_AWS_SECRET_ACCESS_KEY }}" >> $GITHUB_OUTPUT ;;
      qa)   CONFIG='${{ vars.QA_CONFIG_DEPLOY }}'
            echo "aws_access_key=${{ secrets.QA_AWS_ACCESS_KEY_ID }}" >> $GITHUB_OUTPUT
            echo "aws_secret_key=${{ secrets.QA_AWS_SECRET_ACCESS_KEY }}" >> $GITHUB_OUTPUT ;;
      *)    echo "❌ Invalid environment: $ENV"; exit 1 ;;
    esac
    for key in ecr_registry ecs_cluster ecs_task_execution_role_arn ecs_task_role_arn subnet_ids security_group_ids load_balancer_name; do
      val=$(echo "$CONFIG" | jq -r --arg k "$key" '.[$k] // ""')
      echo "${key}=${val}" >> $GITHUB_OUTPUT
    done
    echo "✅ Variables resolved for environment: $ENV"
```

### Acessando nos Jobs Subsequentes

Os jobs `docker` e `deploy` continuam usando os outputs do `prepare` da mesma forma:

```yaml
docker:
  needs: prepare
  uses: invoisys/infra-ci-cd/.github/workflows/composite-docker.yml@main
  with:
    ecr_registry: ${{ needs.prepare.outputs.ecr_registry }}
```

## Variáveis Específicas da Aplicação

Configurações específicas de uma aplicação (URLs de filas SQS, APIs, IDs de secrets de aplicação) **não** entram no JSON `CONFIG_DEPLOY`. Elas permanecem em variáveis separadas no repositório ou na organização, e o job `prepare` pode escrevê-las no `GITHUB_OUTPUT` após o loop do `jq` (ex.: `sqs_queue_url`, `url_api_proxy`, `abbyy_secret_id` para SendToABBYY).

## Como Adicionar / Configurar

### 1. Variável JSON de config (organização)

1. Acesse **GitHub → Organization Settings → Secrets and variables → Actions**.
2. Aba **Variables** → **New organization variable**.
3. Nome: `{ENV}_CONFIG_DEPLOY` (ex.: `SBX_CONFIG_DEPLOY`).
4. Valor: JSON no formato do schema (uma linha, sem quebras).
5. Repository access: selecione os repositórios que farão deploy.
6. Repita para cada ambiente (DEV, QA, SBX, PRD) conforme necessário.

### 2. Secrets (organização)

1. Na aba **Secrets** → **New organization secret**.
2. Nome: `{ENV}_AWS_ACCESS_KEY_ID` ou `{ENV}_AWS_SECRET_ACCESS_KEY`.
3. Valor: credencial AWS.
4. Repository access: conforme política.

### 3. Variáveis específicas da aplicação

- Mantenha no **repositório** (Repository Settings → Secrets and variables → Actions) ou na organização com escopo ao repositório.
- No workflow, acesse via `${{ vars.NOME }}` e escreva no `GITHUB_OUTPUT` no step `Resolve environment variables` (após o loop `jq`).

## Variáveis da Organização vs. Repositório

| Tipo | Onde | Exemplos |
|------|------|----------|
| **Organização** | Config de deploy (uma var JSON por ambiente) | `SBX_CONFIG_DEPLOY`, `PRD_CONFIG_DEPLOY` |
| **Organização** | Credenciais AWS por ambiente | `SBX_AWS_ACCESS_KEY_ID`, `SBX_AWS_SECRET_ACCESS_KEY` |
| **Repositório** | Configurações específicas da aplicação | `SQS_INBOUND_NFSE_ABBYY_QUEUE_URL`, `API_PROXY_URL`, `ABBYY_SECRET_ID` |

## Gerenciamento via GitHub CLI

### Listar Variables da Organização

```bash
gh api /orgs/invoisys/actions/variables
```

### Criar/atualizar CONFIG_DEPLOY (exemplo SBX)

O valor deve ser um JSON válido em uma linha. Exemplo:

```bash
# Montar o JSON e criar a variável (substitua os valores reais)
gh api /orgs/invoisys/actions/variables \
  -f name="SBX_CONFIG_DEPLOY" \
  -f value='{"ecr_registry":"123456789012.dkr.ecr.us-east-1.amazonaws.com","ecs_cluster":"cluster-sbx","ecs_task_execution_role_arn":"arn:aws:iam::123456789012:role/ecsTaskExecutionRole","ecs_task_role_arn":"arn:aws:iam::123456789012:role/ecsTaskRole","subnet_ids":"subnet-abc,subnet-def","security_group_ids":"sg-abc,sg-def","load_balancer_name":"alb-sbx-main"}' \
  -f visibility="selected"
```

### Secrets

```bash
gh api /orgs/invoisys/actions/secrets
# Criar: use encrypted_value conforme documentação do GitHub CLI
```

## Validação

1. **JSON bem formado**: use `jq` para validar o conteúdo de `{ENV}_CONFIG_DEPLOY` (veja [troubleshooting-env-vars.md](troubleshooting-env-vars.md)).
2. **Logs do job prepare**: devem mostrar `✅ Variables resolved for environment: {ENV}`.
3. **Outputs**: confirme nos steps seguintes que os valores necessários (ecr_registry, ecs_cluster, etc.) não estão vazios.

## Migração a partir das variáveis antigas

Se você ainda tem as variáveis individuais (`SBX_ECR_REGISTRY`, `SBX_ECS_CLUSTER`, etc.):

1. Monte o JSON juntando os valores atuais (uma chave por variável antiga).
2. Crie a variável `{ENV}_CONFIG_DEPLOY` na organização com esse JSON.
3. Atualize os workflows para o novo padrão (CONFIG_DEPLOY + jq).
4. Após validar o deploy, remova as variáveis antigas.

## Troubleshooting

Veja o documento [troubleshooting-env-vars.md](troubleshooting-env-vars.md) para:

- Validar sintaxe e chaves do JSON
- Debugar variáveis não encontradas
- Comandos `jq` para inspecionar valores

## Referências

- [deploy-env-pattern.md](deploy-env-pattern.md) — Template do step prepare com CONFIG_DEPLOY + jq
- [troubleshooting-env-vars.md](troubleshooting-env-vars.md) — Validação do JSON e problemas comuns
- [GitHub Actions: Variables](https://docs.github.com/en/actions/learn-github-actions/variables)
- [GitHub Actions: Secrets](https://docs.github.com/en/actions/security-guides/encrypted-secrets)

## Histórico de Mudanças

| Data | Descrição |
|------|-----------|
| 2026-02-11 | Criação inicial do documento com convenção de nomenclatura e lista completa de variáveis |
| 2026-02-11 | Substituição de variáveis individuais por uma variável JSON por ambiente (`{ENV}_CONFIG_DEPLOY`), schema documentado; secrets mantidos separados |
