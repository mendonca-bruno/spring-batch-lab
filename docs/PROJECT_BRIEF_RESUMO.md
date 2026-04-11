# PROJECT BRIEF — spring-batch-lab

## Objetivo e Domínio de Negócio
Aplicação de batch Java que executa jobs periódicos no Kubernetes via Argo Workflows. Serve como laboratório de estudo de Spring Batch + pipeline CI/CD moderno com release imutável e deploy manual controlado.

## Stack Principal
| Camada | Tecnologia |
|--------|-----------|
| Linguagem / Build | Java 17, Maven (mvnw) |
| Framework | Spring Batch |
| Container | Docker (eclipse-temurin:17-jre-alpine), Docker Hub |
| Orquestrador | Kubernetes + Argo Workflows (CronWorkflow) |
| CI/CD | GitHub Actions (self-hosted runner) |
| Versionamento | SemVer + annotated git tags |

## Estrutura de Pastas/Módulos
```
spring-batch-lab/
├── .github/workflows/   # ci.yml e cd.yml
├── k8s/workflows/       # spring-batch-workflow.yaml e spring-batch-cron-workflow.yaml
├── src/                 # Código-fonte Java / Spring Batch
├── docs/                # Documentação (CICD.md, este arquivo)
├── Dockerfile
└── pom.xml
```

## Visão do Pipeline CI/CD
**CI (`ci.yml`) — gatilho: push em `main`**
1. Valida que a versão no `pom.xml` não é SNAPSHOT e que a tag não existe.
2. Cria annotated git tag (`vX.Y.Z`) e GitHub Release com notas automáticas.
3. `mvn clean package -DskipTests` → JAR.
4. `docker build + push` para Docker Hub (tags `:vX.Y.Z` e `:latest`).

**CD (`cd.yml`) — gatilho: manual (`workflow_dispatch`) com input `tag`**
1. `kubectl patch` atualiza o campo `image` do `CronWorkflow` no namespace `argo`.
2. Na próxima janela de 15 minutos, Argo sobe um Pod com a nova imagem.

## Decisões Arquiteturais e Restrições
- **CI e CD desacoplados por design**: CI produz artefato imutável; CD decide quando e qual versão vai para produção.
- **Self-hosted runner**: necessário para acessar o cluster Kubernetes local sem expô-lo publicamente. Requer `mvn`, `docker`, `kubectl` e `kubeconfig` configurados na máquina.
- **SNAPSHOT bloqueado em `main`**: o CI falha com `exit 1` se a versão tiver sufixo `-SNAPSHOT`.
- **Imutabilidade de tags**: re-uso de uma tag existente é bloqueado pelo CI.
- **`concurrencyPolicy: Forbid`**: o CronWorkflow descarta a execução se ainda houver uma em andamento — sem paralelismo acidental no batch.
- **Secrets**: `DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN` e `GITHUB_TOKEN` (escopo mínimo `contents: write`). Sem credenciais em código.
