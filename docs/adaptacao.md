# Guia de Adaptação

Guia step-by-step para adaptar o pipeline CI/CD para seu projeto específico.

## 📋 Checklist de Configuração Mínima

### ✅ **1. Estrutura do Projeto**

- [ ] Código fonte em diretório `src/` (ou configure `working_directory`)
- [ ] Dockerfile adequado para sua aplicação
- [ ] Testes automatizados configurados
- [ ] Health endpoint para APIs (`/health` ou configure path)

### ✅ **2. GitHub Repository**

- [ ] Workflows copiados para `.github/workflows/`
- [ ] Actions auxiliares em `.github/actions/`
- [ ] Tech configs em `.github/tech-configs/`
- [ ] Environments criados: `dev`, `qa`, `sbx`, `prd`

### ✅ **3. AWS Resources**

- [ ] ECR repositories criados
- [ ] ECS cluster configurado
- [ ] VPC/Subnets/Security Groups
- [ ] IAM roles (Task Execution Role)
- [ ] ALB configurado (para APIs)

### ✅ **4. Configuração GitHub**

- [ ] Variáveis por environment configuradas
- [ ] Secrets AWS configurados
- [ ] Branch protection rules (opcional)
- [ ] Environment approval rules (opcional)

## 🔧 Exemplos de Implementação

### Exemplo .NET

#### **Estrutura do Projeto**
```
meu-projeto-dotnet/
├── .github/
│   ├── workflows/
│   │   └── deploy.yml           # ← Seu workflow principal
│   ├── actions/                 # ← Copiado de infra-ci-cd
│   └── tech-configs/            # ← Copiado de infra-ci-cd
├── src/
│   ├── MinhaApi/
│   │   ├── MinhaApi.csproj
│   │   ├── Program.cs
│   │   └── Controllers/
│   └── MinhaApi.Tests/
│       └── MinhaApi.Tests.csproj
├── Dockerfile.api               # ← Custom ou template
└── README.md
```

#### **Workflow Deploy (.github/workflows/deploy.yml)**
```yaml
name: Deploy API

on:
  push:
    branches: [develop, qa, main]
  workflow_dispatch:

env:
  TECHNOLOGY: dotnet
  WORKING_DIR: src
  PROJECT_NAME: MinhaApi
  SERVICE_TYPE: api
  ECR_REPO: minha-api

jobs:
  # 1. Build da aplicação
  build:
    name: Build
    uses: ./.github/workflows/composite-build.yml
    with:
      technology: ${{ env.TECHNOLOGY }}
      technology_version: "8.0"
      working_directory: ${{ env.WORKING_DIR }}
      project_name: ${{ env.PROJECT_NAME }}
      build_args: "--configuration Release --verbosity normal"

  # 2. Execução de testes
  test:
    name: Test
    uses: ./.github/workflows/composite-test.yml
    with:
      technology: ${{ env.TECHNOLOGY }}
      working_directory: ${{ env.WORKING_DIR }}
      test_args: "--configuration Release --collect:'XPlat Code Coverage'"
    needs: build

  # 3. Build e push da imagem Docker
  docker:
    name: Docker
    uses: ./.github/workflows/composite-docker.yml
    with:
      ecr_repo: ${{ env.ECR_REPO }}
      service_type: ${{ env.SERVICE_TYPE }}
      environment: ${{ github.ref_name }}
      project_name: ${{ env.PROJECT_NAME }}
      push: true
    secrets: inherit
    needs: [build, test]
    if: github.ref_name != 'feature/*'

  # 4. Deploy para ECS
  deploy:
    name: Deploy
    uses: ./.github/workflows/composite-deploy.yml
    with:
      image_uri: ${{ needs.docker.outputs.full_image_uri }}
      ecs_service: ${{ env.ECR_REPO }}-${{ github.ref_name }}
      environment: ${{ github.ref_name }}
      service_type: ${{ env.SERVICE_TYPE }}
      
      # Configurações de recursos
      task_cpu: "512"
      task_memory: "1024"
      container_port: "8080"
      desired_count: "2"
      
      # ALB para APIs
      create_target_group_and_listener: true
      target_group_health_check_path: "/health"
      
      # Environment variables
      container_environment: |
        [
          {"name": "ASPNETCORE_ENVIRONMENT", "value": "${{ github.ref_name }}"},
          {"name": "ASPNETCORE_URLS", "value": "http://+:8080"},
          {"name": "Logging__LogLevel__Default", "value": "Information"}
        ]
    secrets: inherit
    needs: docker
    environment: ${{ github.ref_name }}
```

#### **Dockerfile Customizado (Dockerfile.api)**
```dockerfile
# Build stage
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src

# Copy project files
COPY ["src/MinhaApi/MinhaApi.csproj", "MinhaApi/"]
RUN dotnet restore "MinhaApi/MinhaApi.csproj"

# Copy source and build
COPY src/ .
WORKDIR "/src/MinhaApi"
RUN dotnet build "MinhaApi.csproj" -c Release -o /app/build

# Publish stage
FROM build AS publish
RUN dotnet publish "MinhaApi.csproj" -c Release -o /app/publish

# Runtime stage
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS final
WORKDIR /app

# Create non-root user
RUN groupadd -r appuser && useradd -r -g appuser appuser
RUN chown -R appuser:appuser /app
USER appuser

COPY --from=publish --chown=appuser:appuser /app/publish .

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=30s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1

EXPOSE 8080
ENTRYPOINT ["dotnet", "MinhaApi.dll"]
```

---

### Exemplo Node.js

#### **Estrutura do Projeto**
```
meu-projeto-node/
├── .github/
│   ├── workflows/
│   │   └── deploy.yml           # ← Seu workflow principal
│   ├── actions/                 # ← Copiado de infra-ci-cd
│   └── tech-configs/            # ← Copiado de infra-ci-cd
├── src/
│   ├── index.js
│   ├── package.json
│   ├── package-lock.json
│   └── tests/
│       └── app.test.js
├── Dockerfile.api               # ← Custom ou template
└── README.md
```

#### **Workflow Deploy (.github/workflows/deploy.yml)**
```yaml
name: Deploy API Node.js

on:
  push:
    branches: [develop, qa, main]
  workflow_dispatch:

env:
  TECHNOLOGY: node
  WORKING_DIR: src
  SERVICE_TYPE: api
  ECR_REPO: minha-api-node

jobs:
  # 1. Build da aplicação
  build:
    name: Build
    uses: ./.github/workflows/composite-build.yml
    with:
      technology: ${{ env.TECHNOLOGY }}
      technology_version: "20"
      working_directory: ${{ env.WORKING_DIR }}
      build_args: "--production=false"

  # 2. Execução de testes
  test:
    name: Test
    uses: ./.github/workflows/composite-test.yml
    with:
      technology: ${{ env.TECHNOLOGY }}
      working_directory: ${{ env.WORKING_DIR }}
      test_args: "--coverage --ci --watchAll=false"
    needs: build

  # 3. Build e push da imagem Docker
  docker:
    name: Docker
    uses: ./.github/workflows/composite-docker.yml
    with:
      ecr_repo: ${{ env.ECR_REPO }}
      service_type: ${{ env.SERVICE_TYPE }}
      environment: ${{ github.ref_name }}
      push: true
    secrets: inherit
    needs: [build, test]

  # 4. Deploy para ECS
  deploy:
    name: Deploy
    uses: ./.github/workflows/composite-deploy.yml
    with:
      image_uri: ${{ needs.docker.outputs.full_image_uri }}
      ecs_service: ${{ env.ECR_REPO }}-${{ github.ref_name }}
      environment: ${{ github.ref_name }}
      service_type: ${{ env.SERVICE_TYPE }}
      
      # Configurações de recursos
      task_cpu: "256"
      task_memory: "512"
      container_port: "3000"
      desired_count: "1"
      
      # ALB para APIs
      create_target_group_and_listener: true
      target_group_health_check_path: "/api/health"
      
      # Environment variables
      container_environment: |
        [
          {"name": "NODE_ENV", "value": "${{ github.ref_name == 'main' && 'production' || github.ref_name }}"},
          {"name": "PORT", "value": "3000"},
          {"name": "LOG_LEVEL", "value": "info"}
        ]
    secrets: inherit
    needs: docker
    environment: ${{ github.ref_name }}
```

#### **Package.json**
```json
{
  "name": "minha-api-node",
  "version": "1.0.0",
  "scripts": {
    "start": "node index.js",
    "build": "echo 'Build completed'",
    "test": "jest",
    "test:coverage": "jest --coverage"
  },
  "dependencies": {
    "express": "^4.18.0",
    "cors": "^2.8.5",
    "helmet": "^7.0.0"
  },
  "devDependencies": {
    "jest": "^29.0.0",
    "supertest": "^6.3.0"
  }
}
```

#### **Dockerfile Customizado (Dockerfile.api)**
```dockerfile
# Build stage
FROM node:20-alpine AS builder
WORKDIR /app

# Copy package files
COPY package*.json ./
RUN npm ci --only=production

# Production stage
FROM node:20-alpine AS production
WORKDIR /app

# Create non-root user
RUN addgroup -g 1001 -S nodejs && adduser -S -u 1001 nodejs

# Copy dependencies
COPY --from=builder /app/node_modules ./node_modules
COPY --chown=nodejs:nodejs . .

USER nodejs

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=30s --retries=3 \
  CMD curl -f http://localhost:3000/api/health || exit 1

EXPOSE 3000
CMD ["npm", "start"]
```

---

## 🛠️ Dockerfile Customizado

### Quando Usar Template vs Custom

**Use o Template** quando:
- ✅ Aplicação simples sem dependências especiais
- ✅ Estrutura de diretórios padrão
- ✅ Sem necessidade de multi-stage builds complexos

**Use Custom Dockerfile** quando:
- ⚠️ Dependências específicas do sistema
- ⚠️ Build process complexo
- ⚠️ Otimizações específicas de imagem
- ⚠️ Configurações de segurança avançadas

### Como Usar Dockerfile Customizado

#### **1. Criar o Dockerfile**
Crie `Dockerfile.api` ou `Dockerfile.worker` na raiz do projeto.

#### **2. Configurar o Workflow**
```yaml
docker:
  uses: ./.github/workflows/composite-docker.yml
  with:
    use_template_dockerfile: false    # ← Desabilitar template
    dockerfile_path: "Dockerfile.api" # ← Seu dockerfile
    # ... outras configurações
```

### Boas Práticas para Dockerfiles

#### **Multi-stage Builds**
```dockerfile
# Build dependencies in one stage
FROM node:20-alpine AS dependencies
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

# Runtime in another stage (smaller image)
FROM node:20-alpine AS runtime
WORKDIR /app
COPY --from=dependencies /app/node_modules ./node_modules
COPY . .
CMD ["npm", "start"]
```

#### **Security**
```dockerfile
# Use non-root user
RUN addgroup -g 1001 -S appuser && adduser -S -u 1001 appuser
USER appuser

# Use specific versions
FROM node:20.10.0-alpine

# Minimize attack surface
RUN apk del apk-tools
```

#### **Health Checks**
```dockerfile
# .NET API
HEALTHCHECK CMD curl -f http://localhost:8080/health || exit 1

# Node.js API  
HEALTHCHECK CMD curl -f http://localhost:3000/api/health || exit 1

# Worker (check process)
HEALTHCHECK CMD pgrep -f "my-worker-process" || exit 1
```

## 🔧 Referência para custom-example.yml

Para adicionar suporte a uma nova tecnologia, use [`custom-example.yml`](../tech-configs/custom-example.yml) como template:

### **1. Copiar e Adaptar**
```bash
cp .github/tech-configs/custom-example.yml .github/tech-configs/python.yml
```

### **2. Configurar para Python**
```yaml
name: python
default_version: "3.11"

setup_action: actions/setup-python@v5
setup_with:
  python-version: ${{ inputs.technology_version || '3.11' }}
  cache: 'pip'

commands:
  restore: pip install -r requirements.txt
  build: python -m build
  test: pytest --cov=. --cov-report=xml
  
file_patterns:
  project: "**/requirements.txt"
  lock_file: "**/requirements-lock.txt"
```

### **3. Atualizar Workflows**
Adicione suporte nos workflows `composite-build.yml` e `composite-test.yml`:

```yaml
# Adicionar na validação
case "$TECH" in
  dotnet|node|python) echo "✅ Technology: $TECH" ;;
  *) echo "❌ Unknown technology" && exit 1 ;;
esac

# Adicionar setup
- name: Setup Python
  if: inputs.technology == 'python'
  uses: actions/setup-python@v5
  with:
    python-version: ${{ inputs.technology_version || '3.11' }}
    cache: 'pip'
```

## 🚀 Checklist Final

Antes de fazer o primeiro deploy:

- [ ] **Testado localmente**: Aplicação roda sem erros
- [ ] **Health endpoint**: Responde corretamente
- [ ] **Dockerfile**: Build local bem-sucedido
- [ ] **Environments**: Configurados no GitHub
- [ ] **AWS Resources**: ECR, ECS, ALB prontos
- [ ] **Secrets**: AWS credentials configurados
- [ ] **Primeiro deploy**: Em environment de dev
- [ ] **Logs**: Verificados no CloudWatch
- [ ] **Health checks**: Passando no ALB

## 🆘 Troubleshooting

### Deploy Falha

1. **Verifique logs** no GitHub Actions
2. **Verifique logs** no CloudWatch
3. **Teste health endpoint** localmente
4. **Valide configurações** de environment
5. **Confirme recursos AWS** existem

### Performance Issues  

1. **Ajuste CPU/Memory** nas configurações
2. **Otimize Dockerfile** (multi-stage, cache)
3. **Configure auto-scaling** no ECS
4. **Monitore métricas** no CloudWatch

### Problemas de Rede

1. **Verifique security groups** (portas abertas)
2. **Confirme subnets** estão corretas
3. **Teste conectividade** entre recursos
4. **Valide target group health** no ALB

## 🔗 Próximos Passos

- **Configuração**: Ver [environments.md](environments.md)  
- **Troubleshooting**: Ver [workflows.md](workflows.md)
- **Monitoramento**: Configurar CloudWatch dashboards
- **Alertas**: Configurar SNS notifications
- **Backup**: Implementar strategy de rollback

---

💡 **Dica**: Comece sempre pelo environment `dev` e teste completamente antes de promover para `prd`.