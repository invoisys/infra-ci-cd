# Troubleshooting - Variáveis de Environment

Guia de troubleshooting para problemas relacionados a variáveis e secrets de environment na organização GitHub.

## Índice

- [Verificação Rápida](#verificação-rápida)
- [Problemas Comuns](#problemas-comuns)
- [Como Debugar](#como-debugar)
- [Comandos Úteis](#comandos-úteis)
- [Checklist de Validação](#checklist-de-validação)

---

## Verificação Rápida

### 1. Verificar se as Variáveis Existem na Organização

**Via Interface GitHub:**

1. Acesse: `https://github.com/organizations/invoisys/settings/variables/actions`
2. Procure pela variável de config de deploy do ambiente (ex.: `SBX_CONFIG_DEPLOY`) e pelos secrets `SBX_AWS_ACCESS_KEY_ID`, `SBX_AWS_SECRET_ACCESS_KEY`
3. Verifique se o repositório tem acesso (campo "Repository access")

**Via GitHub CLI:**

```bash
# Listar todas as variables da organização
gh api /orgs/invoisys/actions/variables

# Verificar se a variável JSON do ambiente existe
gh api /orgs/invoisys/actions/variables/SBX_CONFIG_DEPLOY
```

### 2. Verificar Logs do Workflow

1. Acesse o workflow que falhou no GitHub Actions
2. Abra o job `prepare`
3. Expanda o step `Resolve environment variables`
4. Verifique se as mensagens aparecem:
   - ✅ `Variables resolved for environment: sbx`
   - ✅ `ECR Registry: (set)`
   - ✅ `ECS Cluster: (set)`

---

## Problemas Comuns

### Problema 1: Variável CONFIG_DEPLOY Não Encontrada ou Vazia

**Sintoma:**
- Step "Resolve environment variables" grava outputs vazios (ex.: `ecr_registry=`, `ecs_cluster=`)
- Ou erro ao fazer parse do JSON com `jq`

**Causa:**
- A variável `{ENV}_CONFIG_DEPLOY` não existe na organização
- O repositório não tem acesso à variável
- Nome da variável está incorreto (ex.: `SBX_CONFIG_DEPLOY` para branch `sbx`)
- Valor da variável está vazio ou não é um JSON válido

**Solução:**

1. **Verificar se existe:**
   ```bash
   gh api /orgs/invoisys/actions/variables | jq '.variables[] | select(.name == "SBX_CONFIG_DEPLOY")'
   ```

2. **Criar/editar com JSON válido** (uma linha, sem quebras). Ver schema em [organization-variables.md](organization-variables.md):
   ```bash
   gh api /orgs/invoisys/actions/variables \
     -f name="SBX_CONFIG_DEPLOY" \
     -f value='{"ecr_registry":"123456789012.dkr.ecr.us-east-1.amazonaws.com","ecs_cluster":"cluster-sbx","ecs_task_execution_role_arn":"arn:aws:iam::123456789012:role/ecsTaskExecutionRole","ecs_task_role_arn":"arn:aws:iam::123456789012:role/ecsTaskRole","subnet_ids":"subnet-abc,subnet-def","security_group_ids":"sg-abc,sg-def","load_balancer_name":"alb-sbx-main"}' \
     -f visibility="selected"
   ```

3. **Validar o JSON** (sintaxe e chaves esperadas): veja seção [Validar JSON de CONFIG_DEPLOY](#validar-json-de-config_deploy) abaixo.

4. **Dar acesso ao repositório:**
   - Via Interface: Organization Settings → Actions → Variables → `SBX_CONFIG_DEPLOY` → Edit → Selected repositories

### Problema 2: Secret Vazio

**Sintoma:**
```
Error: Credentials could not be loaded
```

**Causa:**
- Secret não está configurado na organização
- Nome do secret está incorreto no case statement

**Solução:**

1. **Verificar se o secret existe:**
   ```bash
   gh api /orgs/invoisys/actions/secrets | jq '.secrets[] | select(.name == "SBX_AWS_ACCESS_KEY_ID")'
   ```

2. **Criar secret se necessário:**
   ```bash
   # Via GitHub CLI
   gh secret set SBX_AWS_ACCESS_KEY_ID --org invoisys --body "AKIAIOSFODNN7EXAMPLE"
   ```

3. **Verificar acesso do repositório ao secret:**
   - Interface: Organization Settings → Actions → Secrets → `SBX_AWS_ACCESS_KEY_ID` → Edit

### Problema 3: Ambiente Inválido

**Sintoma:**
```
❌ Invalid environment: staging
Error: Process completed with exit code 1.
```

**Causa:**
- Branch/ambiente não está mapeado no case statement

**Solução:**

1. **Adicionar novo ambiente no case statement** (e criar `STAGING_CONFIG_DEPLOY` + secrets na organização):

```yaml
staging) CONFIG='${{ vars.STAGING_CONFIG_DEPLOY }}'
         echo "aws_access_key=${{ secrets.STAGING_AWS_ACCESS_KEY_ID }}" >> $GITHUB_OUTPUT
         echo "aws_secret_key=${{ secrets.STAGING_AWS_SECRET_ACCESS_KEY }}" >> $GITHUB_OUTPUT ;;
```
   O loop `jq` que já existe no step continuará escrevendo as chaves do JSON nos outputs.

2. **Criar a variável `STAGING_CONFIG_DEPLOY`** (JSON) e os secrets `STAGING_AWS_ACCESS_KEY_ID`, `STAGING_AWS_SECRET_ACCESS_KEY` na organização.

### Problema 4: Valor no JSON Incorreto ou Desatualizado

**Sintoma:**
- Erro ao autenticar na AWS (ECR, ECS) ou recurso não encontrado (cluster, subnet, etc.)

**Causa:**
- Alguma chave dentro do JSON `{ENV}_CONFIG_DEPLOY` está com valor errado ou desatualizado

**Solução:**

1. **Inspecionar o JSON atual** (valor da variável; não expor em logs públicos se contiver dados sensíveis):
   ```bash
   gh api /orgs/invoisys/actions/variables/SBX_CONFIG_DEPLOY | jq -r '.value' | jq .
   ```

2. **Extrair uma chave específica com jq:**
   ```bash
   gh api /orgs/invoisys/actions/variables/SBX_CONFIG_DEPLOY | jq -r '.value' | jq -r '.ecr_registry'
   ```

3. **Atualizar a variável** com o JSON corrigido (substitua o valor inteiro; uma linha):
   ```bash
   gh api -X PATCH /orgs/invoisys/actions/variables/SBX_CONFIG_DEPLOY \
     -f value='{"ecr_registry":"...","ecs_cluster":"...", ...}'
   ```

### Problema 5: Secrets Não Chegam aos Jobs Subsequentes

**Sintoma:**
```
Error: Missing required credential AWS_ACCESS_KEY_ID
```

**Causa:**
- Jobs `docker` ou `deploy` estão usando `secrets.*` diretamente ao invés de `needs.prepare.outputs.*`

**Solução:**

Corrigir a passagem de secrets nos jobs:

**❌ Incorreto:**
```yaml
docker:
  secrets:
    AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
```

**✅ Correto:**
```yaml
docker:
  needs: [prepare]
  secrets:
    AWS_ACCESS_KEY_ID: ${{ needs.prepare.outputs.aws_access_key }}
    AWS_SECRET_ACCESS_KEY: ${{ needs.prepare.outputs.aws_secret_key }}
```

---

## Como Debugar

### Passo 1: Verificar o Ambiente

```bash
# Ver qual ambiente (branch) está sendo usado
echo "Ambiente atual: ${{ github.ref_name }}"
```

### Passo 2: Adicionar Debug Logging

Adicione logs temporários no step de resolução:

```yaml
- name: Resolve environment variables
  id: vars
  run: |
    ENV="${{ github.ref_name }}"
    echo "🔍 Debugging: ENV=$ENV"
    
    case "$ENV" in
      sbx)
        echo "🔍 Debugging: Usando SBX_CONFIG_DEPLOY"
        # Não faça echo do JSON inteiro se contiver dados sensíveis; apenas confira se existe
        echo "🔍 CONFIG length: $(echo -n '${{ vars.SBX_CONFIG_DEPLOY }}' | wc -c)"
        CONFIG='${{ vars.SBX_CONFIG_DEPLOY }}'
        echo "ecr_registry=$(echo "$CONFIG" | jq -r '.ecr_registry // ""')" >> $GITHUB_OUTPUT
        echo "ecs_cluster=$(echo "$CONFIG" | jq -r '.ecs_cluster // ""')" >> $GITHUB_OUTPUT
        ;;
    esac
```

**⚠️ Atenção**: Nunca faça log de secrets nem do conteúdo completo do JSON se puder conter dados sensíveis.

### Passo 3: Validar Outputs

Adicione um step para validar os outputs gerados:

```yaml
- name: Validate outputs
  run: |
    if [ -z "${{ steps.vars.outputs.ecr_registry }}" ]; then
      echo "❌ ERROR: ecr_registry is empty"
      exit 1
    fi
    
    if [ -z "${{ steps.vars.outputs.ecs_cluster }}" ]; then
      echo "❌ ERROR: ecs_cluster is empty"
      exit 1
    fi
    
    echo "✅ All required outputs are set"
```

---

## Comandos Úteis

### Listar Todas as Variables da Organização

```bash
gh api /orgs/invoisys/actions/variables --paginate | jq '.variables[] | {name, value}'
```

### Filtrar Variables por Prefixo

```bash
# Listar variáveis do ambiente SBX (CONFIG_DEPLOY e outras)
gh api /orgs/invoisys/actions/variables --paginate | jq '.variables[] | select(.name | startswith("SBX_"))'
```

### Listar Todos os Secrets da Organização

```bash
gh api /orgs/invoisys/actions/secrets --paginate | jq '.secrets[] | {name, created_at, updated_at}'
```

**Nota**: Não é possível ler o valor de secrets via API por motivos de segurança.

### Verificar Repositórios com Acesso a uma Variable

```bash
gh api /orgs/invoisys/actions/variables/SBX_CONFIG_DEPLOY/repositories | jq '.repositories[] | .full_name'
```

### Criar/Atualizar CONFIG_DEPLOY (JSON em uma linha)

```bash
# Criar: use -f value com JSON em uma linha (ver organization-variables.md para schema)
gh api /orgs/invoisys/actions/variables \
  -f name="SBX_CONFIG_DEPLOY" \
  -f value='{"ecr_registry":"...","ecs_cluster":"...", ...}' \
  -f visibility="selected"

# Atualizar
gh api -X PATCH /orgs/invoisys/actions/variables/SBX_CONFIG_DEPLOY -f value='{"ecr_registry":"...", ...}'
```

### Criar Novo Secret na Organização

```bash
# Requer libsodium instalado
gh secret set SECRET_NAME \
  --org invoisys \
  --body "secret-value" \
  --repos "repo1,repo2"
```

### Validar JSON de CONFIG_DEPLOY

Para validar sintaxe e chaves esperadas antes de salvar na organização:

```bash
# Sintaxe (retorna o JSON pretty-printed ou erro)
echo '$CONFIG_RAW' | jq .

# Verificar chaves esperadas (ex.: ecr_registry, ecs_cluster)
echo '$CONFIG_RAW' | jq 'keys'
echo '$CONFIG_RAW' | jq -r '.ecr_registry, .ecs_cluster'
```

Chaves do schema: `ecr_registry`, `ecs_cluster`, `ecs_task_execution_role_arn`, `ecs_task_role_arn`, `subnet_ids`, `security_group_ids`, `load_balancer_name`. Valores devem ser single-line (sem newlines) para `GITHUB_OUTPUT`.

### Deletar Variable da Organização

```bash
gh api -X DELETE /orgs/invoisys/actions/variables/VARIABLE_NAME
```

---

## Checklist de Validação

Use este checklist ao adicionar ou migrar uma aplicação para o padrão de variáveis centralizadas:

### Antes do Deploy

- [ ] Variável `{ENV}_CONFIG_DEPLOY` (JSON) existe na organização para cada ambiente usado
- [ ] JSON está bem formado e contém as chaves necessárias (ecr_registry, ecs_cluster, etc.) — validar com `jq`
- [ ] Secrets `{ENV}_AWS_ACCESS_KEY_ID` e `{ENV}_AWS_SECRET_ACCESS_KEY` existem na organização
- [ ] Repositório tem acesso às variáveis e secrets da organização
- [ ] Job `prepare` tem o case para todos os ambientes e o loop jq para as chaves do schema
- [ ] Outputs do job `prepare` incluem todas as variáveis necessárias (incl. específicas da app se houver)
- [ ] Jobs `docker` e `deploy` usam `needs.prepare.outputs.*` ao invés de `secrets.*` direto

### Durante o Deploy

- [ ] Job `prepare` executa sem erros
- [ ] Mensagem "✅ Variables resolved for environment: {env}" aparece nos logs
- [ ] Variáveis não estão vazias nos logs (valores aparecem como "(set)")
- [ ] Jobs subsequentes recebem os valores corretamente

### Após o Deploy

- [ ] Aplicação inicia corretamente
- [ ] Não há erros de credenciais AWS nos logs
- [ ] Aplicação consegue acessar recursos da AWS (ECR, ECS, etc.)

---

## Cenários de Teste

### Testar Ambiente SBX

```bash
# Fazer push para branch sbx
git checkout sbx
git push origin sbx

# Acompanhar workflow
gh run watch
```

### Testar Ambiente PRD

```bash
# Fazer push para branch prd
git checkout prd
git push origin prd

# Acompanhar workflow
gh run watch
```

### Testar Fallback (Compatibilidade)

Se você ainda tem variáveis no repositório/environment:

1. Remova temporariamente uma variável da organização
2. Verifique se o fallback no `composite-deploy.yml` funciona
3. Restaure a variável da organização

---

## Quando Pedir Ajuda

Se após seguir este guia o problema persistir, colete as seguintes informações:

1. **Nome do repositório e workflow**
2. **Branch/ambiente que está falhando**
3. **Link para o workflow run que falhou**
4. **Logs do job `prepare`** (step "Resolve environment variables")
5. **Mensagem de erro completa**
6. **Screenshot da configuração de variables/secrets na organização**

Abra uma issue no repositório `infra-ci-cd` com essas informações.

---

## Referências

- [Documentação: Organization Variables](organization-variables.md) - Lista completa de variáveis
- [Documentação: Deploy Pattern](deploy-env-pattern.md) - Template de implementação
- [Documentação: Deploy Internals](deploy-internals.md) - Funcionamento técnico
- [GitHub Actions: Variables](https://docs.github.com/en/actions/learn-github-actions/variables)
- [GitHub Actions: Secrets](https://docs.github.com/en/actions/security-guides/encrypted-secrets)
- [GitHub CLI: API Reference](https://cli.github.com/manual/gh_api)
