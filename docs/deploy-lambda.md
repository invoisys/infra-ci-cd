# Deploy Lambda

Workflow de deploy idempotente para **AWS Lambda** via código em ZIP no S3. Segue o mesmo padrão de idempotência do [composite-deploy](workflows.md#composite-deploy) (ECS): check → update ou create, com merge de variáveis de ambiente.

**Importante:** Triggers (API Gateway, SQS, S3, EventBridge, etc.) **não** são configurados por este workflow. Devem ser definidos manualmente via Console AWS, AWS CLI, Terraform ou CloudFormation.

## Visão geral

- **Arquivo:** `.github/workflows/composite-deploy-lambda.yml`
- **Gatilhos:** Nenhum (apenas `workflow_call`). O repositório que usa o composite define quando rodar (ex.: push em branch).
- **Autenticação:** OIDC (AWS role ARN). Não utiliza secrets de acesso AWS.
- **Idempotência:** Se a função já existe, atualiza configuração e código; caso contrário, cria a função. Variáveis de ambiente são mescladas (existente × input; input prevalece por chave).

## Inputs

### Obrigatórios

| Nome | Tipo | Descrição |
|------|------|-----------|
| `lambda_function_name` | string | Nome da função Lambda |
| `s3_bucket` | string | Bucket S3 onde está o ZIP do código |
| `s3_key` | string | Chave S3 do arquivo ZIP |
| `runtime` | string | Runtime (ex.: `python3.11`, `nodejs20.x`, `dotnet8`) |
| `handler` | string | Handler (ex.: `index.handler`, `MyAssembly::Namespace.Class::Handler`) |
| `environment` | string | Ambiente (`dev` \| `qa` \| `sbx` \| `prd`) |
| `aws_role_arn` | string | ARN da role IAM para OIDC (deploy) |
| `lambda_role_arn` | string | ARN da execution role da Lambda |

### Opcionais – AWS e Lambda

| Nome | Tipo | Padrão | Descrição |
|------|------|--------|-----------|
| `aws_region` | string | `us-east-1` | Região AWS |
| `memory_size` | string | `512` | Memória em MB (128–10240, múltiplo de 64) |
| `timeout` | string | `30` | Timeout em segundos (1–900) |
| `environment_variables` | string | `{}` | JSON object de variáveis de ambiente |

### Opcionais – VPC

| Nome | Tipo | Padrão | Descrição |
|------|------|--------|-----------|
| `vpc_subnet_ids` | string | `` | IDs das subnets (separados por vírgula) |
| `vpc_security_group_ids` | string | `` | IDs dos security groups (separados por vírgula) |

Se `vpc_subnet_ids` for informado, `vpc_security_group_ids` é obrigatório.

### Opcionais – Versão e alias

| Nome | Tipo | Padrão | Descrição |
|------|------|--------|-----------|
| `publish_version` | boolean | `true` | Publicar nova versão após o deploy |
| `create_alias` | boolean | `false` | Criar ou atualizar alias |
| `alias_name` | string | `latest` | Nome do alias (ex.: `dev`, `prd`, `latest`) |

Se `create_alias` for `true`, `alias_name` deve ser informado.

### Opcionais – Avançado

| Nome | Tipo | Padrão | Descrição |
|------|------|--------|-----------|
| `layers` | string | `` | ARNs de Lambda Layers (separados por vírgula) |
| `reserved_concurrent_executions` | string | `-1` | Concorrência reservada (-1 = sem reserva) |

## Outputs

| Nome | Descrição |
|------|-----------|
| `lambda_arn` | ARN da função Lambda |
| `lambda_version` | Versão publicada (se `publish_version: true`) |
| `lambda_qualified_arn` | ARN com versão ou alias (quando aplicável) |
| `deploy_metadata_artifact` | Nome do artifact com metadata do deploy |

## Validações

- **environment:** deve ser `dev`, `qa`, `sbx` ou `prd`.
- **memory_size:** numérico, 128–10240, múltiplo de 64.
- **timeout:** numérico, 1–900.
- **environment_variables:** JSON válido.
- **runtime:** validado contra lista comum (nodejs18.x, nodejs20.x, python3.10/3.11/3.12, dotnet6/8, java17/21). Outros valores geram aviso.
- **VPC:** se `vpc_subnet_ids` for informado, `vpc_security_group_ids` é obrigatório.
- **Alias:** se `create_alias` for `true`, `alias_name` é obrigatório.

## Exemplo de uso

```yaml
jobs:
  prepare:
    runs-on: ubuntu-latest
    outputs:
      environment: ${{ github.ref_name }}
      aws_role_arn: ${{ steps.config.outputs.aws_role_arn }}
      lambda_role_arn: ${{ steps.config.outputs.lambda_role_arn }}
      lambda_bucket: ${{ steps.config.outputs.lambda_bucket }}
    steps:
      - name: Parse config
        id: config
        run: |
          # Definir outputs a partir de vars / secrets (ex.: vars.DEV_CONFIG_DEPLOY)
          echo "aws_role_arn=arn:aws:iam::123456789012:role/github-oidc" >> $GITHUB_OUTPUT
          echo "lambda_role_arn=arn:aws:iam::123456789012:role/my-lambda-role" >> $GITHUB_OUTPUT
          echo "lambda_bucket=my-artifacts-bucket" >> $GITHUB_OUTPUT

  package:
    needs: [prepare, test]
    runs-on: ubuntu-latest
    outputs:
      s3_key: ${{ steps.upload.outputs.s3_key }}
    steps:
      - uses: actions/checkout@v4
      - name: Build and zip
        run: |
          cd src/MyLambdaFunction
          dotnet publish -c Release -o publish
          cd publish && zip -r ../lambda.zip .
      - name: Configure AWS
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ needs.prepare.outputs.aws_role_arn }}
          aws-region: us-east-1
      - name: Upload to S3
        id: upload
        run: |
          S3_KEY="lambda-packages/${{ github.sha }}/lambda.zip"
          aws s3 cp src/MyLambdaFunction/lambda.zip "s3://${{ needs.prepare.outputs.lambda_bucket }}/${S3_KEY}"
          echo "s3_key=${S3_KEY}" >> $GITHUB_OUTPUT

  deploy:
    needs: [package, prepare]
    uses: ./.github/workflows/composite-deploy-lambda.yml
    with:
      lambda_function_name: my-lambda-${{ needs.prepare.outputs.environment }}
      s3_bucket: ${{ needs.prepare.outputs.lambda_bucket }}
      s3_key: ${{ needs.package.outputs.s3_key }}
      runtime: dotnet8
      handler: MyLambdaFunction::MyNamespace.Handler::FunctionHandler
      environment: ${{ needs.prepare.outputs.environment }}
      lambda_role_arn: ${{ needs.prepare.outputs.lambda_role_arn }}
      aws_role_arn: ${{ needs.prepare.outputs.aws_role_arn }}
      aws_region: us-east-1
      memory_size: "1024"
      timeout: "30"
      publish_version: true
      create_alias: true
      alias_name: ${{ needs.prepare.outputs.environment }}
      environment_variables: '{"LOG_LEVEL":"INFO","ENV":"${{ needs.prepare.outputs.environment }}"}'
```

## Configuração de triggers (fora do workflow)

Triggers não são criados pelo composite. Exemplos de onde configurá-los:

1. **Console AWS** – Lambda → Function → Add trigger.
2. **AWS CLI** – `aws lambda add-permission`, event source mappings, etc.
3. **Terraform / CloudFormation** – Recursos `aws_lambda_permission`, `aws_lambda_event_source_mapping`, `aws_cloudwatch_event_target`, etc.

Ver exemplos em Terraform no plano do repositório (seção “Configuração de Triggers”).

## Deploy metadata

O workflow gera um artifact com JSON de metadata (ambiente, nome da função, S3, ARN, versão, alias, commit, timestamp, run_id). O nome do artifact é exposto em `deploy_metadata_artifact` para uso em jobs downstream.

## Exemplo completo

Um exemplo de pipeline completo (prepare → build → test → package → deploy) está em [examples/deploy-lambda-example.yml](examples/deploy-lambda-example.yml). Copie e adapte para seu repositório.
