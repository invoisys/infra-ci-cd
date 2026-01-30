# Estratégia de versionamento e tagging (ECR)

## Tags geradas em cada deploy

| Tag | Exemplo | Uso |
|-----|---------|-----|
| `{sha}` | `abc1234` | Rastreabilidade e rollback (imutável) |
| `{branch}` | `dev`, `qa`, `sbx`, `prd` | Task Definition pode referenciar por ambiente |
| `{timestamp}` | `20250130-143022` | Auditoria (UTC) |

## Histórico das últimas versões

- **Artifact por run**: Cada deploy bem-sucedido gera um artifact `deploy-{env}-{deployment_name}-{run_id}` com `.deploys/deploy.json` contendo:
  - `environment`, `deployment_name`, `image_digest`, `image_tag`, `commit_sha`, `branch`, `timestamp`, `run_id`, `run_url`
- **Consulta**: Em **Actions** do repositório da aplicação, abra um run de deploy e baixe o artifact para ver a versão deployada.
- **Rollback**: Use `image_tag` (SHA) ou a tag de timestamp do ECR no workflow de rollback manual.

## Convenção no ECR

- Repositório: mesmo nome do `deployment_name` (ex.: `minha-api`, `meu-worker`).
- Para rollback manual, informe a tag desejada (ex.: `abc1234` ou `20250130-143022`).
