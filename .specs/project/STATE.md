# State

**Last Updated:** 2026-02-07T00:00:00Z  
**Current Work:** Modularização do Pipeline - Feature "workflows-compostos" **IMPLEMENTED**

---

## Recent Decisions (Last 60 days)

### AD-005: Estrutura de arquivos com prefixo ao invés de subpasta (2026-02-07)

**Decision:** Usar prefixo `composite-*.yml` na pasta workflows/ ao invés de subpasta `composites/`  
**Reason:** Simplifica referências e evita conflitos de path em `workflow_call`  
**Trade-off:** Menos organização visual vs compatibilidade garantida  
**Impact:** Arquivos: `composite-build.yml`, `composite-test.yml`, `composite-docker.yml`, `composite-deploy.yml`

### AD-003: Usar reusable workflows ao invés de composite actions (2026-02-07)

**Decision:** Implementar workflows compostos usando `workflow_call` (reusable workflows)  
**Reason:** Permite jobs separados com runners independentes, environments e secrets próprios  
**Trade-off:** Composite actions rodariam no mesmo job (menos overhead), mas sem isolamento de environment  
**Impact:** Cada workflow composto é um job separado no pipeline

### AD-004: Configuração de tecnologia em arquivos YAML externos (2026-02-07)

**Decision:** Criar arquivos `.github/tech-configs/{tech}.yml` para configuração por linguagem  
**Reason:** Extensibilidade sem modificar código core - adicionar Python = criar `python.yml`  
**Trade-off:** Mais arquivos vs lógica embutida nos workflows  
**Impact:** Suporte multi-tecnologia via configuração, não código

### AD-001: Usar workflows compostos do GitHub Actions (2026-02-07)

**Decision:** Adotar composite actions e reusable workflows para modularização  
**Reason:** Permite reutilização nativa no GitHub Actions sem ferramentas externas  
**Trade-off:** Limitações de compartilhamento de contexto entre jobs  
**Impact:** Arquitetura baseada em workflows chamando outros workflows

### AD-002: Separar em 4 workflows compostos (2026-02-07)

**Decision:** Modularizar em build, test, docker, deploy  
**Reason:** Granularidade equilibrada - nem muito fine-grained nem monolítico  
**Trade-off:** 4 arquivos vs 1, mas cada um < 150 linhas  
**Impact:** Permite executar fases individualmente (ex: apenas build em PRs)

---

## Active Blockers

*Nenhum blocker ativo*

---

## Lessons Learned

*Nenhuma lição registrada ainda*

---

## Preferences

**Model Guidance Shown:** never
