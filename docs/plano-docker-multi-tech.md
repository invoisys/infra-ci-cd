# Plano de Ajuste — composite-docker multi-tecnologia

> Suporte a roteamento de Dockerfiles por tecnologia (`dotnet` / `node`) no workflow `composite-docker.yml`.

---

## 1. Contexto

| Aspecto | Estado atual |
|---|---|
| Estrutura de Dockerfiles | `build/Dockerfile.api` e `build/Dockerfile.worker` (flat, somente .NET) |
| Resolução de path no workflow | `.ci-templates/build/Dockerfile.{service_type}` |
| Input `technology` | **Não existe** no composite-docker (existe em build e test) |
| Build args do Docker | Apenas `PROJECT_NAME` (específico de .NET) |
| Dockerfiles Node.js | **Não existem** |

Os composites de **build** e **test** já suportam `dotnet` e `node` via input `technology`.  
O composite-docker é o único que ainda não faz essa distinção.

---

## 2. Alterações planejadas

### 2.1 Reestruturar diretório `build/`

Mover os Dockerfiles existentes e criar os de Node.js:

```
build/
├── dotnet/
│   ├── Dockerfile.api        ← mover de build/Dockerfile.api
│   └── Dockerfile.worker     ← mover de build/Dockerfile.worker
└── node/
    ├── Dockerfile.api         ← NOVO
    └── Dockerfile.worker      ← NOVO
```

### 2.2 Adicionar input `technology` no `composite-docker.yml`

```yaml
technology:
  description: 'Technology: dotnet or node'
  required: false
  type: string
  default: 'dotnet'
```

> **Nota:** `default: 'dotnet'` garante retrocompatibilidade — pipelines existentes que não passam esse input continuam funcionando.

### 2.3 Adicionar validação de `technology`

No step **Validate inputs**, incluir:

```bash
case "${{ inputs.technology }}" in
  dotnet|node) echo "✅ Technology: ${{ inputs.technology }}" ;;
  *) echo "❌ Invalid technology: ${{ inputs.technology }}. Supported: dotnet, node" && exit 1 ;;
esac
```

### 2.4 Alterar step "Determine Dockerfile path"

**De:**

```bash
DF=".ci-templates/build/Dockerfile.${{ inputs.service_type }}"
```

**Para:**

```bash
DF=".ci-templates/build/${{ inputs.technology }}/Dockerfile.${{ inputs.service_type }}"
```

### 2.5 Condicionar build args por tecnologia

No step **Build Docker image**, ajustar a montagem de `BUILD_ARGS`:

| Tecnologia | Build args |
|---|---|
| `dotnet` | `--build-arg PROJECT_NAME=<valor>` (se informado) |
| `node` | Nenhum por padrão; usar input `docker_build_args` para extras |

Sugestão de implementação:

```bash
BUILD_ARGS=""
if [ "${{ inputs.technology }}" = "dotnet" ] && [ -n "${{ inputs.project_name }}" ]; then
  BUILD_ARGS="--build-arg PROJECT_NAME=${{ inputs.project_name }}"
fi
```

### 2.6 Criar Dockerfiles Node.js

#### `build/node/Dockerfile.api`

```dockerfile
# Dockerfile genérico para APIs Node.js (repo de templates CI/CD)
# Usado quando use_template_dockerfile=true no pipeline.
# Build context: raiz do repositório da aplicação.

FROM node:20-alpine

WORKDIR /app

# Copiar manifests primeiro para cache de dependências
COPY src/package*.json ./
RUN npm ci --omit=dev

# Copiar código-fonte
COPY src/. .

EXPOSE 80

CMD ["node", "index.js"]
```

#### `build/node/Dockerfile.worker`

```dockerfile
# Dockerfile genérico para Workers Node.js (repo de templates CI/CD)
# Usado quando use_template_dockerfile=true no pipeline.
# Build context: raiz do repositório da aplicação.

FROM node:20-alpine

WORKDIR /app

# Copiar manifests primeiro para cache de dependências
COPY src/package*.json ./
RUN npm ci --omit=dev

# Copiar código-fonte
COPY src/. .

CMD ["node", "index.js"]
```

### 2.7 Atualizar chamada no `deploy.yml` (exemplo de uso)

Adicionar `technology: dotnet` na chamada ao job docker para tornar explícito:

```yaml
docker:
  uses: iannkzw/infra-ci-cd/.github/workflows/composite-docker.yml@refactor-v1
  with:
    technology: dotnet           # ← NOVO
    # ... demais inputs
```

---

## 3. Resumo de arquivos afetados

| Arquivo | Ação |
|---|---|
| `.github/workflows/composite-docker.yml` | Adicionar input `technology`; alterar path resolution; condicionar build args; validar technology |
| `build/Dockerfile.api` | **Mover** para `build/dotnet/Dockerfile.api` |
| `build/Dockerfile.worker` | **Mover** para `build/dotnet/Dockerfile.worker` |
| `build/node/Dockerfile.api` | **Criar** |
| `build/node/Dockerfile.worker` | **Criar** |
| `exemplo-uso-pipeline/.github/workflows/deploy.yml` | Adicionar `technology: dotnet` no job docker |

---

## 4. Riscos e pontos de atenção

| # | Risco | Mitigação |
|---|---|---|
| 1 | **Breaking change** — pipelines que apontam para `build/Dockerfile.api` param de funcionar | Usar `default: 'dotnet'` no input; comunicar mudança de path em release notes |
| 2 | **Versão do Node fixa no Dockerfile** | Considerar parametrizar via `--build-arg NODE_VERSION` futuramente |
| 3 | **Build context diferente para Node** | Input `build_context` já existe e resolve |
| 4 | **Dockerfile customizado** | Fluxo `use_template_dockerfile: false` + `dockerfile_path` continua como escape hatch |
| 5 | **Entrypoint Node** | Dockerfile template assume `index.js` na raiz do `src/`; projetos com entrypoint diferente devem usar Dockerfile próprio |
