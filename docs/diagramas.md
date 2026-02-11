# Diagramas do Pipeline

Visualizações dos fluxos e arquitetura do pipeline CI/CD usando diagramas Mermaid.

## 1. Fluxo Completo do Pipeline

Visão geral de todo o pipeline desde o push até o deploy.

```mermaid
graph TD
    A[Push para Branch] --> B{Branch?}
    
    B -->|develop| C[Environment: dev]
    B -->|qa/staging| D[Environment: qa]
    B -->|sandbox| E[Environment: sbx]
    B -->|main/master| F[Environment: prd]
    
    C --> G[Build]
    D --> G
    E --> G
    F --> G
    
    G --> H{Build OK?}
    H -->|❌| I[❌ Pipeline Failed]
    H -->|✅| J[Test]
    
    J --> K{Tests Pass?}
    K -->|❌| I
    K -->|✅| L[Docker Build]
    
    L --> M{Docker OK?}
    M -->|❌| I
    M -->|✅| N[Push to ECR]
    
    N --> O[Deploy to ECS]
    O --> P{Service Type?}
    
    P -->|API| Q[Configure ALB]
    P -->|Worker| R[Deploy Worker]
    
    Q --> S[✅ API Ready]
    R --> T[✅ Worker Ready]
    
    style A fill:#e1f5fe
    style I fill:#ffebee
    style S fill:#e8f5e8
    style T fill:#e8f5e8
```

## 2. Mapeamento Branch → Environment

Como as branches são mapeadas para environments (usados para aprovações e proteções). A **config de deploy** (ECR, ECS, rede, ALB) vem da **organização** (variável `{ENV}_CONFIG_DEPLOY` + secrets), não dos environments do repositório.

```mermaid
graph LR
    subgraph "Branches"
        A[develop]
        B[qa]
        C[staging]
        D[sandbox]
        E[main]
        F[master]
    end
    
    subgraph "Environments"
        G[dev]
        H[qa]
        I[sbx] 
        J[prd]
    end
    
    subgraph "Approvals"
        K[🔒 None]
        L[🔒 QA Team]
        M[🔒 DevOps]
        N[🔒 Tech Lead + Wait 5min]
    end
    
    A --> G
    B --> H
    C --> H
    D --> I
    E --> J
    F --> J
    
    G -.-> K
    H -.-> L
    I -.-> M
    J -.-> N
    
    style G fill:#4caf50
    style H fill:#ff9800
    style I fill:#2196f3
    style J fill:#f44336
```

## 3. Deploy API com Application Load Balancer

Fluxo detalhado de deploy para APIs com ALB e Target Groups.

```mermaid
graph TD
    A[Docker Image in ECR] --> B[Create Task Definition]
    B --> C{ALB Setup Required?}
    
    C -->|Yes| D[Check ALB Exists]
    C -->|No| E[Create/Update ECS Service]
    
    D --> F{ALB Found?}
    F -->|❌| G[❌ ALB Not Found]
    F -->|✅| H[Create Target Group]
    
    H --> I[Create Listener]
    I --> J[Configure Health Check]
    J --> K[Set Health Check Path: /health]
    
    K --> E[Create/Update ECS Service]
    E --> L[Register Tasks to ALB]
    
    L --> M[Wait for Health Checks]
    M --> N{All Tasks Healthy?}
    
    N -->|❌| O[Check Logs & Health Endpoint]
    N -->|✅| P[Update Traffic Routing]
    
    P --> Q[✅ API Live on ALB]
    
    subgraph "ECS Cluster"
        R[Task 1: Healthy]
        S[Task 2: Healthy]
    end
    
    subgraph "Application Load Balancer"
        T[Target Group]
        U[Listener :80]
        V[Health Check: /health]
    end
    
    E -.-> R
    E -.-> S
    L -.-> T
    I -.-> U
    J -.-> V
    
    style A fill:#e3f2fd
    style Q fill:#e8f5e8
    style G fill:#ffebee
```

## 4. Deploy Worker (Fluxo Simplificado)

Deploy de workers/background services sem Load Balancer.

```mermaid
graph TD
    A[Docker Image in ECR] --> B[Create Task Definition]
    
    B --> C[Configure Worker Settings]
    C --> D[Set CPU/Memory Limits]
    D --> E[Set Environment Variables]
    E --> F[Configure Logging]
    
    F --> G[Create/Update ECS Service]
    G --> H[Start Desired Tasks]
    
    H --> I{All Tasks Running?}
    I -->|❌| J[Check Resource Limits]
    I -->|✅| K[Monitor Logs]
    
    J --> L[Adjust CPU/Memory]
    L --> G
    
    K --> M{Logs OK?}
    M -->|❌| N[Check Application Health]
    M -->|✅| O[✅ Worker Ready]
    
    N --> P[Review Environment Variables]
    P --> Q[Check Dependencies]
    Q --> R[Fix Configuration]
    R --> G
    
    subgraph "ECS Cluster"
        S[Worker Task 1]
        T[Worker Task 2]
        U[Worker Task N]
    end
    
    subgraph "CloudWatch"
        V[Application Logs]
        W[Container Logs]
        X[Metrics]
    end
    
    H -.-> S
    H -.-> T
    H -.-> U
    K -.-> V
    K -.-> W
    K -.-> X
    
    style A fill:#e3f2fd
    style O fill:#e8f5e8
    style C fill:#fff3e0
    style D fill:#fff3e0
    style E fill:#fff3e0
    style F fill:#fff3e0
```

## Fluxos por Tipo de Serviço

### APIs (com ALB)
- ✅ Recebem tráfego HTTP/HTTPS externo
- ✅ Health checks automáticos
- ✅ Load balancing entre múltiplas tasks
- ✅ Traffic routing baseado em path/host
- ⚠️ Requer configuração de Target Group e Listener

### Workers (sem ALB)
- ✅ Processamento em background
- ✅ Escalonamento baseado em CPU/Memory
- ✅ Logs centralizados no CloudWatch
- ⚠️ Sem health checks de rede automáticos
- ⚠️ Monitoramento baseado em logs/métricas

## Recursos AWS Utilizados

```mermaid
graph LR
    subgraph "CI/CD"
        A[GitHub Actions]
        B[ECR Registry]
    end
    
    subgraph "Compute"
        C[ECS Fargate]
        D[CloudWatch Logs]
    end
    
    subgraph "Network (APIs)"
        E[Application Load Balancer]
        F[Target Groups]
        G[VPC Subnets]
        H[Security Groups]
    end
    
    subgraph "IAM"
        I[Task Execution Role]
        J[Task Role]
        K[GitHub OIDC/Keys]
    end
    
    A --> B
    B --> C
    C --> D
    C --> G
    C --> H
    E --> F
    F --> C
    I --> C
    J --> C
    K --> A
    
    style A fill:#24292e,color:#fff
    style B fill:#ff9900
    style C fill:#ff9900
    style D fill:#ff9900
    style E fill:#ff9900
```

## Interpretando os Diagramas

### Símbolos e Cores

- 🔵 **Azul**: Processos de build/docker
- 🟢 **Verde**: Sucessos/endpoints prontos
- 🟠 **Laranja**: Configurações/variáveis
- 🔴 **Vermelho**: Failures/erros
- 🔒 **Cadeado**: Approvals/proteções necessárias
- ✅ **Check**: Validações bem-sucedidas
- ❌ **X**: Falhas que interrompem pipeline

### Pontos de Falha Comuns

1. **Build Stage**: Erros de compilação, dependências faltando
2. **Test Stage**: Testes falhando, cobertura insuficiente
3. **Docker Stage**: Dockerfile incorreto, push ECR sem permissão
4. **Deploy Stage**: Configuração de rede incorreta, health checks falhando

### Monitoramento

- **CloudWatch**: Logs de aplicação e container
- **ECS Console**: Status das tasks e services
- **ALB Console**: Health dos targets (apenas APIs)
- **GitHub Actions**: Logs detalhados de cada stage

## Próximos Passos

- Para config de deploy na organização (variável JSON + secrets): Ver [organization-variables.md](organization-variables.md) e [deploy-env-pattern.md](deploy-env-pattern.md)
- Para configurar environments no repositório (aprovações, wait timer): Ver [environments.md](environments.md)
- Para detalhes dos workflows: Ver [workflows.md](workflows.md)
- Para adaptar para seu projeto: Ver [adaptacao.md](adaptacao.md)