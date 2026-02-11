# Pattern: Resolução de Variáveis de Environment

## Visão Geral

Este documento fornece templates reutilizáveis para resolver variáveis de environment nos workflows de deploy. A **config de deploy** (ECR, ECS, rede, load balancer) vem de **uma única variável por ambiente** na organização contendo um JSON (`{ENV}_CONFIG_DEPLOY`). O job `prepare` faz um `case` por ambiente para escolher qual variável e quais secrets usar, depois usa `jq` para extrair as chaves do JSON e escrever nos outputs. Secrets (AWS) permanecem em variáveis separadas.

## Problema

O GitHub Actions não suporta interpolação dinâmica de nomes de variáveis:

```yaml
# ❌ NÃO FUNCIONA
${{ vars[format('{0}_CONFIG_DEPLOY', github.ref_name)] }}
```

## Solução

Usar um `case` statement no job `prepare` para definir `CONFIG` a partir de `vars.{ENV}_CONFIG_DEPLOY` e escrever os secrets no `GITHUB_OUTPUT`; em seguida, usar `jq` para extrair as chaves do JSON e escrevê-las nos outputs. O runner `ubuntu-latest` já inclui `jq`.

### Exemplo de JSON por ambiente

A variável `SBX_CONFIG_DEPLOY` (e equivalentes para DEV, QA, PRD) contém um JSON no formato:

```json
{"ecr_registry":"123456789012.dkr.ecr.us-east-1.amazonaws.com","ecs_cluster":"cluster-sbx","ecs_task_execution_role_arn":"arn:aws:iam::123456789012:role/ecsTaskExecutionRole","ecs_task_role_arn":"arn:aws:iam::123456789012:role/ecsTaskRole","subnet_ids":"subnet-abc,subnet-def","security_group_ids":"sg-abc,sg-def","load_balancer_name":"alb-sbx-main"}
```

Chaves não usadas pela aplicação podem ser omitidas ou ter valor `""`; o `jq` com `// ""` produz string vazia. Ver schema em [organization-variables.md](organization-variables.md).

## Template Básico

Use para aplicações que precisam apenas de ECR e ECS (outputs `ecr_registry`, `ecs_cluster` e secrets). O mesmo step escreve todas as chaves do schema nos outputs; o job expõe apenas as que você usa.

```yaml
prepare:
  name: Load environment variables
  runs-on: ubuntu-latest
  outputs:
    ecr_registry: ${{ steps.vars.outputs.ecr_registry }}
    ecs_cluster: ${{ steps.vars.outputs.ecs_cluster }}
    aws_access_key: ${{ steps.vars.outputs.aws_access_key }}
    aws_secret_key: ${{ steps.vars.outputs.aws_secret_key }}
    technology: dotnet
    technology_version: '8.0'
    working_directory: src
    aws_region: us-east-1
    service_type: api
    deployment_name: ${{ github.ref_name }}-xxxx-xxx-xxx-service-name
  steps:
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
        echo "ECR Registry: (set)"
        echo "ECS Cluster: (set)"
```

## Template Completo

Use para aplicações que precisam de todas as variáveis de infraestrutura (incluindo rede e load balancer) e opcionalmente variáveis específicas da aplicação:

```yaml
prepare:
  name: Load environment variables
  runs-on: ubuntu-latest
  outputs:
    ecr_registry: ${{ steps.vars.outputs.ecr_registry }}
    ecs_cluster: ${{ steps.vars.outputs.ecs_cluster }}
    ecs_task_execution_role_arn: ${{ steps.vars.outputs.ecs_task_execution_role_arn }}
    ecs_task_role_arn: ${{ steps.vars.outputs.ecs_task_role_arn }}
    subnet_ids: ${{ steps.vars.outputs.subnet_ids }}
    security_group_ids: ${{ steps.vars.outputs.security_group_ids }}
    load_balancer_name: ${{ steps.vars.outputs.load_balancer_name }}
    aws_access_key: ${{ steps.vars.outputs.aws_access_key }}
    aws_secret_key: ${{ steps.vars.outputs.aws_secret_key }}
    app_specific_var: ${{ steps.vars.outputs.app_specific_var }}
    technology: node
    technology_version: '18.0'
    working_directory: src
    aws_region: us-east-1
    service_type: worker
    deployment_name: ${{ github.ref_name }}-xxxx-xxx-xxx-service-name
  steps:
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
        # Variáveis específicas da aplicação (fora do JSON; repositório ou organização)
        echo "app_specific_var=${{ vars.APP_SPECIFIC_VAR }}" >> $GITHUB_OUTPUT
        echo "✅ Variables resolved for environment: $ENV"
        echo "ECR Registry: (set)"
        echo "ECS Cluster: (set)"
        echo "Subnet IDs: (set)"
        echo "Security Groups: (set)"
```

## Como Usar nos Jobs Subsequentes

### Job Docker

```yaml
docker:
  needs: prepare
  uses: invoisys/infra-ci-cd/.github/workflows/composite-docker.yml@main
  with:
    ecr_registry: ${{ needs.prepare.outputs.ecr_registry }}
    technology: ${{ needs.prepare.outputs.technology }}
    technology_version: ${{ needs.prepare.outputs.technology_version }}
    working_directory: ${{ needs.prepare.outputs.working_directory }}
    aws_region: ${{ needs.prepare.outputs.aws_region }}
  secrets:
    AWS_ACCESS_KEY_ID: ${{ needs.prepare.outputs.aws_access_key }}
    AWS_SECRET_ACCESS_KEY: ${{ needs.prepare.outputs.aws_secret_key }}
```

### Job Deploy

```yaml
deploy:
  needs: [prepare, docker]
  uses: invoisys/infra-ci-cd/.github/workflows/composite-deploy.yml@main
  with:
    ecr_registry: ${{ needs.prepare.outputs.ecr_registry }}
    ecs_cluster: ${{ needs.prepare.outputs.ecs_cluster }}
    service_type: ${{ needs.prepare.outputs.service_type }}
    deployment_name: ${{ needs.prepare.outputs.deployment_name }}
    technology: ${{ needs.prepare.outputs.technology }}
    aws_region: ${{ needs.prepare.outputs.aws_region }}
    # Networking (se necessário)
    ecs_task_execution_role_arn: ${{ needs.prepare.outputs.ecs_task_execution_role_arn }}
    ecs_task_role_arn: ${{ needs.prepare.outputs.ecs_task_role_arn }}
    subnet_ids: ${{ needs.prepare.outputs.subnet_ids }}
    security_group_ids: ${{ needs.prepare.outputs.security_group_ids }}
    load_balancer_name: ${{ needs.prepare.outputs.load_balancer_name }}
  secrets:
    AWS_ACCESS_KEY_ID: ${{ needs.prepare.outputs.aws_access_key }}
    AWS_SECRET_ACCESS_KEY: ${{ needs.prepare.outputs.aws_secret_key }}
```

## Variáveis Específicas vs. Comuns

### Na Organização (Config de deploy + Secrets)

- **Uma variável JSON por ambiente**: `{ENV}_CONFIG_DEPLOY` — contém `ecr_registry`, `ecs_cluster`, `ecs_task_execution_role_arn`, `ecs_task_role_arn`, `subnet_ids`, `security_group_ids`, `load_balancer_name` (ver [organization-variables.md](organization-variables.md)).
- **Secrets por ambiente**: `{ENV}_AWS_ACCESS_KEY_ID`, `{ENV}_AWS_SECRET_ACCESS_KEY`.

### Manter no Repositório ou Organização (Específicas)

Variáveis específicas da aplicação **fora** do JSON:

- URLs de APIs, filas SQS, buckets S3
- IDs de secrets de aplicação (ex.: ABBYY_SECRET_ID)
- Configurações de aplicação

## Checklist de Implementação

Ao implementar este padrão em uma aplicação:

- [ ] Configurar na organização `{ENV}_CONFIG_DEPLOY` (JSON) e secrets `{ENV}_AWS_*` para cada ambiente
- [ ] Identificar variáveis específicas da aplicação (manter fora do JSON, no repo ou org)
- [ ] Copiar o template apropriado (Básico ou Completo)
- [ ] Ajustar os outputs do job `prepare` (incluir variáveis específicas se houver)
- [ ] Manter o `case` para sbx/prd/dev/qa e o loop `jq`; adicionar `echo` de variáveis específicas após o loop
- [ ] Atualizar jobs `docker` e `deploy` para usar `needs.prepare.outputs.*`
- [ ] Validar JSON (sintaxe e chaves) — ver [troubleshooting-env-vars.md](troubleshooting-env-vars.md)
- [ ] Testar em ambiente de sandbox primeiro e confirmar deploy bem-sucedido

## Vantagens desta Abordagem

1. **Simples e Direto**: Não depende de features não suportadas do GitHub Actions
2. **Explícito**: Fácil de entender qual variável vem de onde
3. **Centralizado**: Variáveis comuns gerenciadas em um único lugar
4. **Testável**: Fácil adicionar logging e validação
5. **Mantenível**: Mudanças são localizadas e previsíveis

## Debugging

Para verificar se as variáveis estão sendo resolvidas corretamente, adicione logs no final do `case` statement:

```yaml
echo "✅ Variables resolved for environment: $ENV"
echo "ECR Registry: ${ecr_registry:-NOT SET}"
echo "ECS Cluster: ${ecs_cluster:-NOT SET}"
```

Ou acesse os logs do workflow no GitHub Actions para ver os valores das variáveis resolvidas.

## Troubleshooting

### Erro: "Invalid environment"

O ambiente (branch) não está mapeado no `case` statement. Adicione um novo caso ou verifique o nome da branch.

### Variável Vazia ou JSON inválido

A variável `{ENV}_CONFIG_DEPLOY` não está configurada, está vazia ou o JSON é inválido. Verifique:

1. Se a variável existe: GitHub → Organization Settings → Actions → Variables → `SBX_CONFIG_DEPLOY` (e equivalentes)
2. Se o repositório tem acesso à variável
3. Se o valor é um JSON válido (sem quebras de linha); use `jq` para validar — ver [troubleshooting-env-vars.md](troubleshooting-env-vars.md)

### Secrets Não Passados

Certifique-se de usar `needs.prepare.outputs.*` para passar secrets entre jobs, não diretamente `secrets.*`.

## Referências

- [Documentação: Organization Variables](organization-variables.md)
- [Documentação: Troubleshooting](troubleshooting-env-vars.md)
- [GitHub Actions: Outputs](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#jobsjob_idoutputs)

## Exemplos Reais

Veja implementações reais nas aplicações:

- [`exemplos-migracao-github/api.customlookupdata/.github/workflows/deploy.yml`](../../exemplos-migracao-github/api.customlookupdata/.github/workflows/deploy.yml)
- [`exemplos-migracao-github/nfeepechandler/.github/workflows/deploy.yml`](../../exemplos-migracao-github/nfeepechandler/.github/workflows/deploy.yml)
- [`exemplos-migracao-github/SendToABBYY/.github/workflows/deploy.yml`](../../exemplos-migracao-github/SendToABBYY/.github/workflows/deploy.yml)
