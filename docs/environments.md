# Configuração de Environments

Guia completo para configurar environments, variáveis e secrets no GitHub para usar com os workflows compostos.

## Pré-requisitos

Antes de configurar os environments, você deve ter:

- **Acesso de Admin**: Permissão para criar environments no repositório GitHub
- **Conta AWS**: Acesso à conta AWS com permissões para:
  - ECR (Elastic Container Registry)
  - ECS (Elastic Container Service)  
  - VPC (subnets e security groups)
  - ALB/ELB (para APIs)
  - IAM (roles de execução)

## Mapa de Environments

O pipeline suporta 4 environments padrão, cada um mapeado para branches específicas:

| Environment | Branch | Descrição | Uso |
|-------------|--------|-----------|-----|
| `dev` | `develop` | Desenvolvimento | Testes contínuos, integração |
| `qa` | `qa`, `staging` | Qualidade/Homologação | Testes de QA, validação |
| `sbx` | `sandbox` | Sandbox | Testes experimentais |
| `prd` | `main`, `master` | Produção | Ambiente final dos usuários |

## Configuração no GitHub

### 1. Criar Environments

Acesse `Settings > Environments` no seu repositório e crie os seguintes environments:

- `dev`
- `qa` 
- `sbx`
- `prd`

### 2. Configurar Proteções (Opcional)

Para environments sensíveis (`prd`, `qa`), configure:

- **Protection rules**: Require reviewers (1-6 pessoas)
- **Wait timer**: Tempo de espera antes do deploy (ex: 5 minutos)
- **Restrict branches**: Apenas branches específicas

## Variáveis por Environment

Configure as seguintes variáveis em cada environment criado:

### Variáveis Obrigatórias

| Variável | Descrição | Exemplo | Notas |
|----------|-----------|---------|-------|
| `ECR_REGISTRY` | URL do registry ECR | `123456789.dkr.ecr.us-east-1.amazonaws.com` | Obtido no console ECR |
| `ECS_CLUSTER` | Nome do cluster ECS | `meu-cluster-dev` | Um por environment |
| `ECS_TASK_EXECUTION_ROLE_ARN` | ARN da role de execução | `arn:aws:iam::123456789:role/ecsTaskExecutionRole` | Permissões para ECR/CloudWatch |

### Variáveis de Rede

| Variável | Descrição | Exemplo | Notas |
|----------|-----------|---------|-------|
| `SUBNET_IDS` | IDs das subnets privadas | `subnet-xxx,subnet-yyy` | Separadas por vírgula |
| `SECURITY_GROUP_IDS` | IDs dos security groups | `sg-xxx,sg-yyy` | Separados por vírgula |
| `ASSIGN_PUBLIC_IP` | Atribuir IP público | `DISABLED` | ENABLED/DISABLED |

### Variáveis de Load Balancer (APIs apenas)

| Variável | Descrição | Exemplo | Notas |
|----------|-----------|---------|-------|
| `LOAD_BALANCER_ARN` | ARN do ALB | `arn:aws:elasticloadbalancing:us-east-1:123456789:loadbalancer/app/dev-alb/abcdef123456` | Para APIs |
| `LOAD_BALANCER_NAME` | Nome do ALB | `dev-alb` | Alternativa ao ARN |
| `TARGET_GROUP_PORT` | Porta do target group | `80` | Padrão: 80 |
| `LISTENER_PORT` | Porta do listener | `80` | Padrão: 80 |

### Variáveis Opcionais

| Variável | Descrição | Exemplo | Padrão |
|----------|-----------|---------|--------|
| `AWS_REGION` | Região AWS | `us-east-1` | `us-east-1` |
| `TASK_CPU` | CPU das tasks | `512` | `256` |
| `TASK_MEMORY` | Memória das tasks | `1024` | `512` |
| `DESIRED_COUNT` | Número de tasks | `2` | `1` |

## Secrets Necessários

Configure os secrets que podem ser no nível do **repositório** (todos environments) ou por **environment** (específico):

### Secrets Obrigatórios

| Secret | Nível | Descrição | Como Obter |
|--------|-------|-----------|------------|
| `AWS_ACCESS_KEY_ID` | Repo/Env | Chave de acesso AWS | Console IAM → Users → Security credentials |
| `AWS_SECRET_ACCESS_KEY` | Repo/Env | Chave secreta AWS | Console IAM → Users → Security credentials |

### Secrets Opcionais

| Secret | Nível | Descrição | Quando Usar |
|--------|-------|-----------|-------------|
| `ECS_CLUSTER` | Environment | Override do cluster | Quando diferente da variável |
| `ECS_TASK_EXECUTION_ROLE_ARN` | Environment | Override da role | Quando diferente da variável |

## Permissões IAM Necessárias

A chave AWS configurada deve ter as seguintes permissões:

### Para ECR (Docker workflows)
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload",
        "ecr:PutImage"
      ],
      "Resource": "*"
    }
  ]
}
```

### Para ECS (Deploy workflows)
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecs:RegisterTaskDefinition",
        "ecs:UpdateService",
        "ecs:DescribeServices",
        "ecs:DescribeTaskDefinition",
        "ecs:DescribeTasks",
        "ecs:ListTasks"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "iam:PassRole"
      ],
      "Resource": "arn:aws:iam::*:role/ecsTaskExecutionRole"
    }
  ]
}
```

### Para ALB (APIs com Load Balancer)
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "elasticloadbalancing:CreateTargetGroup",
        "elasticloadbalancing:CreateListener",
        "elasticloadbalancing:DescribeTargetGroups",
        "elasticloadbalancing:DescribeListeners",
        "elasticloadbalancing:ModifyTargetGroup",
        "elasticloadbalancing:ModifyListener"
      ],
      "Resource": "*"
    }
  ]
}
```

## Exemplo de Configuração

### Environment `dev`

**Variáveis:**
```
ECR_REGISTRY=123456789.dkr.ecr.us-east-1.amazonaws.com
ECS_CLUSTER=meu-projeto-dev
ECS_TASK_EXECUTION_ROLE_ARN=arn:aws:iam::123456789:role/ecsTaskExecutionRole-dev
SUBNET_IDS=subnet-12345,subnet-67890
SECURITY_GROUP_IDS=sg-abcdef
LOAD_BALANCER_ARN=arn:aws:elasticloadbalancing:us-east-1:123456789:loadbalancer/app/dev-alb/1234567890
TASK_CPU=512
TASK_MEMORY=1024
```

**Secrets:** (no nível do repositório)
```
AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
```

## Troubleshooting

### Erros Comuns

#### ❌ `Unknown environment: staging`
**Causa**: Environment não configurado no GitHub ou nome incorreto.
**Solução**: Verifique se o environment existe em `Settings > Environments`.

#### ❌ `AWS credentials not provided`
**Causa**: Secrets AWS não configurados.
**Solução**: Configure `AWS_ACCESS_KEY_ID` e `AWS_SECRET_ACCESS_KEY`.

#### ❌ `ECR repository does not exist`
**Causa**: Repositório ECR não existe ou nome incorreto.
**Solução**: Crie o repositório no console ECR ou verifique o nome.

#### ❌ `ECS cluster not found`
**Causa**: Cluster não existe ou região incorreta.
**Solução**: Verifique se o cluster existe na região especificada.

#### ❌ `Invalid subnet ID`
**Causa**: Subnet não existe ou não está na mesma VPC.
**Solução**: Verifique os IDs das subnets no console VPC.

#### ❌ `Target group creation failed`
**Causa**: ALB não existe ou permissões insuficientes.
**Solução**: Verifique se o ALB existe e se as permissões IAM estão corretas.

### Validações Automáticas

Os workflows fazem as seguintes validações:

- **Environment**: Deve ser `dev`, `qa`, `sbx` ou `prd`
- **Service Type**: Deve ser `api` ou `worker`
- **Technology**: Deve ser `dotnet` ou `node`
- **AWS Credentials**: Devem estar presentes para workflows Docker/Deploy
- **Required Variables**: ECR_REGISTRY, ECS_CLUSTER etc. devem estar definidas

### Testando a Configuração

Para testar se sua configuração está correta:

1. **Faça um push** numa branch mapeada (ex: `develop` → `dev`)
2. **Verifique os logs** do workflow na aba Actions
3. **Procure por mensagens** `✅` de validação bem-sucedida
4. **Se houver erros**, verifique as seções acima

## Próximos Passos

- Para detalhes dos workflows: Ver [`workflows.md`](workflows.md)
- Para diagramas do pipeline: Ver [`diagramas.md`](diagramas.md)  
- Para adaptar para seu projeto: Ver [`adaptacao.md`](adaptacao.md)