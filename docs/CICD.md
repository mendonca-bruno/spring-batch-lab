# CI/CD — Documentação de Estudo

> Referência técnica detalhada sobre a arquitetura de integração e entrega contínua do projeto **spring-batch-lab**.

---

## Sumário

1. [Visão Geral da Arquitetura](#1-visão-geral-da-arquitetura)
2. [Fluxo Completo — Do Commit ao Deploy](#2-fluxo-completo--do-commit-ao-deploy)
3. [GitHub Actions — Conceitos Fundamentais](#3-github-actions--conceitos-fundamentais)
4. [Workflow CI — Build and Push](#4-workflow-ci--build-and-push)
5. [Workflow CD — Deploy no Argo](#5-workflow-cd--deploy-no-argo)
6. [Dockerfile e Containerização](#6-dockerfile-e-containerização)
7. [Infraestrutura — Kubernetes e Argo Workflows](#7-infraestrutura--kubernetes-e-argo-workflows)
8. [Secrets e Segurança](#8-secrets-e-segurança)
9. [Estratégia de Versionamento](#9-estratégia-de-versionamento)
10. [Tecnologias Utilizadas — Resumo](#10-tecnologias-utilizadas--resumo)
11. [Diagrama de Relacionamento entre Componentes](#11-diagrama-de-relacionamento-entre-componentes)

---

## 1. Visão Geral da Arquitetura

O projeto adota um modelo de CI/CD em duas etapas independentes, cada uma representada por um workflow do GitHub Actions:

```
┌─────────────────────────────────────────────────────────────────┐
│                        DEVELOPER                                │
│  develop branch  ──► pull request ──► merge em main            │
└─────────────────────────────────────────┬───────────────────────┘
                                          │ push para main
                                          ▼
┌─────────────────────────────────────────────────────────────────┐
│                  CI — ci.yml (automático)                       │
│  1. Valida versão (bloqueia SNAPSHOT)                           │
│  2. Cria git tag (vX.Y.Z)                                       │
│  3. Cria GitHub Release                                         │
│  4. Build Maven → JAR                                           │
│  5. Build Docker → Push para Docker Hub (:vX.Y.Z e :latest)    │
└─────────────────────────────────────────┬───────────────────────┘
                                          │ imagem disponível no Hub
                                          ▼
┌─────────────────────────────────────────────────────────────────┐
│              CD — cd.yml (manual / workflow_dispatch)           │
│  Input: tag (ex: v1.4.0)                                       │
│  kubectl patch → atualiza imagem no CronWorkflow do Argo        │
└─────────────────────────────────────────┬───────────────────────┘
                                          │ nova imagem configurada
                                          ▼
┌─────────────────────────────────────────────────────────────────┐
│               Argo Workflows — Kubernetes (namespace: argo)     │
│  CronWorkflow executa o batch a cada 15 minutos                 │
│  (concurrencyPolicy: Forbid — sem execuções paralelas)          │
└─────────────────────────────────────────────────────────────────┘
```

**Princípio central:** CI e CD são desacoplados. A CI produz artefatos imutáveis (imagem Docker versionada). O CD decide **quando** e **qual versão** vai para produção, via acionamento manual.

---

## 2. Fluxo Completo — Do Commit ao Deploy

| Etapa | Ator | Ação |
|-------|------|------|
| 1 | Dev | Faz commits na branch `develop` |
| 2 | Dev | Abre Pull Request para `main` |
| 3 | Dev | Atualiza versão no `pom.xml` (ex: `1.3.0` → `1.4.0`) |
| 4 | Dev | Merge do PR em `main` |
| 5 | GitHub Actions | **CI dispara automaticamente** |
| 6 | CI | Valida que a versão não é SNAPSHOT |
| 7 | CI | Verifica se a tag `v1.4.0` já existe (proteção contra duplicação) |
| 8 | CI | Cria a tag `v1.4.0` e publica no repositório |
| 9 | CI | Cria GitHub Release com notas automáticas |
| 10 | CI | Builda o JAR com Maven |
| 11 | CI | Constrói a imagem Docker e sobe para Docker Hub |
| 12 | Dev/Ops | **CD acionado manualmente** via GitHub Actions UI, informando a tag |
| 13 | CD | Executa `kubectl patch` no CronWorkflow do Argo com a nova imagem |
| 14 | Argo | Na próxima execução agendada, sobe um Pod com a nova versão |

---

## 3. GitHub Actions — Conceitos Fundamentais

### O que é GitHub Actions?

GitHub Actions é a plataforma de automação nativa do GitHub. Permite criar **workflows** (fluxos de trabalho) que executam tarefas automaticamente em resposta a eventos no repositório.

### Estrutura de um Workflow

```yaml
name: Nome do Workflow        # Nome exibido na UI do GitHub

on:                           # TRIGGER — o que dispara o workflow
  push:
    branches:
      - main

jobs:                         # Conjunto de jobs
  nome-do-job:               # Identificador do job
    runs-on: self-hosted     # Onde o job executa (runner)
    steps:                   # Passos sequenciais do job
      - name: Passo 1
        run: echo "comando"  # Comando shell
      - name: Passo 2
        uses: actions/checkout@v4  # Action reutilizável
```

### Conceitos-Chave

| Conceito | Descrição |
|----------|-----------|
| **Workflow** | Arquivo YAML em `.github/workflows/`. Define o pipeline completo |
| **Job** | Unidade de execução dentro do workflow. Jobs podem rodar em paralelo ou com dependências |
| **Step** | Passo individual dentro de um job. Executa em sequência |
| **Action** | Bloco reutilizável de lógica (`uses: ...`). Pode ser da comunidade ou custom |
| **Runner** | Máquina que executa o job. Pode ser hospedado pelo GitHub ou **self-hosted** |
| **Trigger** (`on`) | Evento que dispara o workflow |
| **Secret** | Variável sensível armazenada de forma criptografada no GitHub |
| **Output** | Valor produzido por um step e consumido por steps posteriores |
| **`GITHUB_OUTPUT`** | Arquivo especial para passar outputs entre steps |
| **`GITHUB_TOKEN`** | Token temporário gerado automaticamente pelo GitHub para o workflow |

### Tipos de Triggers usados no projeto

#### `push` com filtro de branch (ci.yml)
```yaml
on:
  push:
    branches:
      - main
```
Dispara **automaticamente** sempre que há um push direto ou merge de PR para a branch `main`.

#### `workflow_dispatch` com inputs (cd.yml)
```yaml
on:
  workflow_dispatch:
    inputs:
      tag:
        description: 'Tag para deploy (ex: v1.2.0)'
        required: true
```
Permite **acionamento manual** pela UI do GitHub Actions ou via API. O campo `inputs` define parâmetros que o operador preenche na hora de executar. O valor é acessado via `${{ github.event.inputs.tag }}`.

### Expressões e Contextos

GitHub Actions usa a sintaxe `${{ ... }}` para acessar contextos:

| Expressão | Significado |
|-----------|-------------|
| `${{ secrets.DOCKERHUB_TOKEN }}` | Secret configurado no repositório |
| `${{ github.repository }}` | `owner/repo`, ex: `mendonca-bruno/spring-batch-lab` |
| `${{ steps.version.outputs.tag }}` | Output do step com id `version` |
| `${{ github.event.inputs.tag }}` | Input fornecido no `workflow_dispatch` |

### Outputs entre Steps

Para passar dados de um step para outro:

```yaml
- name: Extrair versão do pom.xml
  id: version                        # id que identifica este step
  run: |
    VERSION=$(mvn help:evaluate -Dexpression=project.version -q -DforceStdout)
    TAG="v${VERSION}"
    echo "version=$VERSION" >> $GITHUB_OUTPUT   # escreve o output
    echo "tag=$TAG" >> $GITHUB_OUTPUT

- name: Usar a versão
  run: echo "A tag é ${{ steps.version.outputs.tag }}"   # lê o output
```

### Permissions

```yaml
permissions:
  contents: write
```
Define as permissões do `GITHUB_TOKEN` para este workflow. `contents: write` é necessário para criar tags e releases no repositório.

---

## 4. Workflow CI — Build and Push

**Arquivo:** [.github/workflows/ci.yml](../.github/workflows/ci.yml)

**Trigger:** `push` para `main`

**Runner:** `self-hosted` (máquina local ou servidor próprio com o GitHub Actions Runner instalado)

### Passo a Passo Detalhado

#### Step 1 — Checkout do código
```yaml
- name: Checkout do código
  uses: actions/checkout@v4
  with:
    fetch-depth: 0
```
- `actions/checkout@v4`: Action oficial do GitHub para clonar o repositório
- `fetch-depth: 0`: Clona o histórico **completo** do git (não apenas o último commit). Necessário para que comandos como `git ls-remote --tags` funcionem corretamente e para verificar se tags existem

#### Step 2 — Extrair versão do pom.xml
```yaml
- name: Extrair versão do pom.xml
  id: version
  run: |
    VERSION=$(mvn help:evaluate -Dexpression=project.version -q -DforceStdout)
    TAG="v${VERSION}"
    echo "version=$VERSION" >> $GITHUB_OUTPUT
    echo "tag=$TAG" >> $GITHUB_OUTPUT
```
- `mvn help:evaluate`: Plugin Maven para avaliar expressões do POM sem executar o build
- `-Dexpression=project.version`: Extrai especificamente a versão do projeto
- `-q -DforceStdout`: Suprime output do Maven e força a impressão do resultado no stdout
- Resultado: se `pom.xml` tem `<version>1.4.0</version>`, então `VERSION=1.4.0` e `TAG=v1.4.0`

#### Step 3 — Validar versão de release
```yaml
- name: Validar versão de release
  run: |
    VERSION=${{ steps.version.outputs.version }}
    if [[ "$VERSION" == *"-SNAPSHOT"* ]]; then
      echo "::error::Versão SNAPSHOT ($VERSION) não pode ser mergeada em main."
      exit 1
    fi
```
- Proteção de qualidade: impede que versões de desenvolvimento (`-SNAPSHOT`) cheguem à `main`
- `echo "::error::..."`: Sintaxe especial do GitHub Actions para registrar um erro com formatação na UI
- `exit 1`: Falha o step e consequentemente o workflow inteiro

#### Step 4 — Verificar se a tag já existe
```yaml
- name: Verificar se a tag já existe
  run: |
    TAG=${{ steps.version.outputs.tag }}
    if git ls-remote --tags origin | grep -q "refs/tags/${TAG}$"; then
      echo "::error::Tag ${TAG} já existe."
      exit 1
    fi
```
- `git ls-remote --tags origin`: Lista todas as tags do repositório remoto sem precisar de `git fetch`
- Protege contra re-execução com a mesma versão, garantindo imutabilidade das releases

#### Step 5 — Criar e publicar tag Git
```yaml
- name: Criar e publicar tag Git
  run: |
    git config user.name "github-actions[bot]"
    git config user.email "github-actions[bot]@users.noreply.github.com"
    git remote set-url origin https://x-access-token:${{ secrets.GITHUB_TOKEN }}@github.com/${{ github.repository }}.git
    git tag -a ${{ steps.version.outputs.tag }} -m "${{ steps.version.outputs.tag }}"
    git push origin ${{ steps.version.outputs.tag }}
```
- Configura identidade do git para o bot do GitHub Actions
- `git remote set-url`: Redefine a URL do remote usando o `GITHUB_TOKEN` para autenticação HTTPS
- `git tag -a`: Cria uma **annotated tag** (tem metadados: autor, data, mensagem — diferente de lightweight tag)
- Requer `permissions: contents: write`

#### Step 6 — Criar GitHub Release
```yaml
- name: Criar GitHub Release
  uses: softprops/action-gh-release@v2
  with:
    tag_name: ${{ steps.version.outputs.tag }}
    name: ${{ steps.version.outputs.tag }}
    generate_release_notes: true
```
- `softprops/action-gh-release@v2`: Action da comunidade para criar releases no GitHub
- `generate_release_notes: true`: GitHub gera automaticamente as notas com base nos commits e PRs desde a última release
- Cria uma entrada visível em `github.com/owner/repo/releases`

#### Step 7 — Build com Maven
```yaml
- name: Build com Maven
  run: mvn clean package -DskipTests
```
- `clean`: Remove o diretório `target/` antes de compilar
- `package`: Compila, roda testes e empacota como JAR
- `-DskipTests`: Pula a execução dos testes (assumindo que já foram validados no PR)
- Produz o artefato `target/spring-batch-lab-1.4.0.jar`

#### Step 8 — Login no Docker Hub
```yaml
- name: Login no Docker Hub
  uses: docker/login-action@v3
  with:
    username: ${{ secrets.DOCKERHUB_USERNAME }}
    password: ${{ secrets.DOCKERHUB_TOKEN }}
```
- `docker/login-action@v3`: Action oficial do Docker para autenticar no registry
- Usa Access Token (não senha) — prática recomendada de segurança
- A sessão de login persiste para os steps seguintes do mesmo job

#### Step 9 — Set up Docker Buildx
```yaml
- name: Set up Docker Buildx
  uses: docker/setup-buildx-action@v3
```
- **Docker Buildx**: Extensão do Docker CLI que habilita builds avançados
- Suporta multi-plataforma (ARM, AMD64), cache avançado e build em paralelo
- É pré-requisito para usar a `docker/build-push-action` com todas as suas funcionalidades

#### Step 10 — Build e Push da imagem Docker
```yaml
- name: Build e Push da imagem Docker
  uses: docker/build-push-action@v5
  with:
    context: .
    push: true
    tags: |
      ${{ secrets.DOCKERHUB_USERNAME }}/spring-batch-lab:${{ steps.version.outputs.tag }}
      ${{ secrets.DOCKERHUB_USERNAME }}/spring-batch-lab:latest
```
- `context: .`: Usa o diretório raiz como contexto de build (onde está o `Dockerfile`)
- `push: true`: Envia a imagem para o registry após o build
- Publica **duas tags** para a mesma imagem:
  - `:v1.4.0` — tag imutável e versionada (ideal para rollback)
  - `:latest` — alias dinâmico que sempre aponta para a versão mais recente

---

## 5. Workflow CD — Deploy no Argo

**Arquivo:** [.github/workflows/cd.yml](../.github/workflows/cd.yml)

**Trigger:** `workflow_dispatch` (manual)

**Runner:** `self-hosted`

### Por que CD manual?

O deploy em produção é separado e manual por design. Isso garante:
- **Controle humano** sobre quando uma versão vai para produção
- **Flexibilidade** para escolher qualquer versão tagueada (não necessariamente a última)
- **Separação de responsabilidades**: a CI prova que o código funciona; o CD decide quando expor

### Como executar

1. Ir em **Actions** no GitHub
2. Selecionar **CD - Deploy no Argo**
3. Clicar em **Run workflow**
4. Informar a tag desejada (ex: `v1.4.0`)
5. Clicar em **Run workflow**

### Passo a Passo Detalhado

#### Step 1 — Checkout do código
```yaml
- name: Checkout do código
  uses: actions/checkout@v4
```
O checkout é necessário porque o runner self-hosted precisa do contexto do repositório (sem `fetch-depth: 0` aqui pois não precisa do histórico completo).

#### Step 2 — Atualizar imagem no CronWorkflow do Argo
```yaml
- name: Atualizar imagem no CronWorkflow do Argo
  run: |
    kubectl patch cronworkflow spring-batch-scheduled -n argo \
      --type='json' \
      -p='[{"op": "replace", "path": "/spec/workflowSpec/templates/0/container/image", "value": "${{ secrets.DOCKERHUB_USERNAME }}/spring-batch-lab:${{ github.event.inputs.tag }}"}]'
```

**Anatomia do `kubectl patch`:**

| Parâmetro | Significado |
|-----------|-------------|
| `cronworkflow` | Tipo de recurso Kubernetes (CRD do Argo) |
| `spring-batch-scheduled` | Nome do recurso |
| `-n argo` | Namespace onde o recurso existe |
| `--type='json'` | Usa JSON Patch (RFC 6902) para patches cirúrgicos |
| `op: replace` | Operação de substituição de valor |
| `path` | Caminho no JSON do recurso a ser alterado |
| `value` | Novo valor — a imagem Docker com a tag informada |

**O runner self-hosted** precisa ter `kubectl` instalado e configurado com acesso ao cluster Kubernetes (via `~/.kube/config` ou variável de ambiente `KUBECONFIG`).

---

## 6. Dockerfile e Containerização

**Arquivo:** [Dockerfile](../Dockerfile)

```dockerfile
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY target/*.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### Análise de Cada Instrução

| Instrução | Descrição |
|-----------|-----------|
| `FROM eclipse-temurin:17-jre-alpine` | Imagem base com JRE 17 (apenas runtime, não SDK) em Alpine Linux (~70MB) |
| `WORKDIR /app` | Define o diretório de trabalho dentro do container |
| `COPY target/*.jar app.jar` | Copia o JAR buildado pelo Maven para dentro da imagem |
| `ENTRYPOINT ["java", "-jar", "app.jar"]` | Comando de inicialização do container |

### Decisões de Design

**JRE vs JDK:** A imagem usa `jre` (Java Runtime Environment), não `jdk` (Java Development Kit). Em produção, apenas o runtime é necessário — o JDK inclui ferramentas de desenvolvimento desnecessárias que aumentam o tamanho da imagem e a superfície de ataque.

**Alpine Linux:** Distribuição Linux minimalista (~5MB). Resulta em imagens Docker significativamente menores que Ubuntu ou Debian.

**Eclipse Temurin:** Distribuição OpenJDK mantida pela Adoptium (Eclipse Foundation). É a substituição do AdoptOpenJDK, com suporte de longo prazo e builds verificados.

### Tags publicadas no Docker Hub

Para cada release, duas tags são criadas:

```
mendoncabruno/spring-batch-lab:v1.4.0   ← imutável, versionada
mendoncabruno/spring-batch-lab:latest   ← sempre aponta para a última versão
```

---

## 7. Infraestrutura — Kubernetes e Argo Workflows

### Self-Hosted Runner

O GitHub Actions oferece runners hospedados pelo próprio GitHub (ubuntu-latest, windows-latest, etc.). Este projeto usa `runs-on: self-hosted`, ou seja, um **runner registrado no próprio ambiente**.

**Vantagens do self-hosted:**
- Acesso direto ao cluster Kubernetes local (sem exposição pública)
- Sem custo de minutos de CI
- Configuração customizada (JDK, Maven, kubectl, Docker já instalados)

**Pré-requisitos do runner:**
- `mvn` (Maven) instalado e no PATH
- `docker` instalado e daemon rodando
- `kubectl` configurado com acesso ao cluster Kubernetes
- Registrado no repositório GitHub via token

### Kubernetes

Plataforma de orquestração de containers. No projeto, é usado para executar os workflows do Argo.

**Namespace `argo`:** Separação lógica de recursos no cluster. Todos os recursos do Argo Workflows ficam no namespace `argo`.

### Argo Workflows

Argo Workflows é um motor de orquestração de workflows nativamente Kubernetes. Cada workflow é executado como um Pod dentro do cluster.

**Custom Resource Definitions (CRDs) usados:**

#### `Workflow` — Execução Pontual

**Arquivo:** [k8s/workflows/spring-batch-workflow.yaml](../k8s/workflows/spring-batch-workflow.yaml)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: spring-batch-job-   # prefixo; sufixo aleatório é adicionado
  namespace: argo
spec:
  entrypoint: executar-job           # template de entrada
  templates:
    - name: executar-job
      container:
        image: mendoncabruno/spring-batch-lab:IMAGE_TAG
        imagePullPolicy: Always
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
```

- `generateName`: O Argo gera um nome único para cada execução (ex: `spring-batch-job-k9xz2`)
- `imagePullPolicy: Always`: Força o Kubernetes a sempre baixar a imagem do registry, nunca usar cache local — importante para que a tag `latest` sempre traga a versão mais recente
- `resources.requests`: Recursos **garantidos** para o container (o Kubernetes reserva isso)
- `resources.limits`: Recursos **máximos** que o container pode usar

#### `CronWorkflow` — Execução Agendada

**Arquivo:** [k8s/workflows/spring-batch-cron-workflow.yaml](../k8s/workflows/spring-batch-cron-workflow.yaml)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: CronWorkflow
metadata:
  name: spring-batch-scheduled      # nome fixo (não usa generateName)
  namespace: argo
spec:
  schedule: "*/15 * * * *"          # a cada 15 minutos
  timezone: "America/Sao_Paulo"     # fuso horário do schedule
  concurrencyPolicy: Forbid         # não permite execuções simultâneas
  startingDeadlineSeconds: 0        # sem tolerância para execuções atrasadas
  workflowSpec:                     # especificação do workflow a executar
    entrypoint: executar-job
    templates:
      - name: executar-job
        container:
          image: mendoncabruno/spring-batch-lab:IMAGE_TAG
          ...
```

**Anatomia do Schedule (cron):**

```
*/15  *  *  *  *
 │    │  │  │  └─ dia da semana (0-7, 0=Dom)
 │    │  │  └──── mês (1-12)
 │    │  └─────── dia do mês (1-31)
 │    └────────── hora (0-23)
 └─────────────── minuto (0-59), */15 = a cada 15 minutos
```

**`concurrencyPolicy: Forbid`:** Se um workflow ainda está rodando quando o próximo horário chegar, o novo é **descartado** (não enfileirado). Adequado para batch jobs que não devem rodar em paralelo.

**`startingDeadlineSeconds: 0`:** Sem janela de tolerância. Se o scheduler falhar e perder um horário, a execução não é recuperada retroativamente.

### Como o CD atualiza o CronWorkflow

O `kubectl patch` com JSON Patch modifica cirurgicamente o campo `image` do recurso no cluster:

```
Antes do patch:
/spec/workflowSpec/templates/0/container/image = "mendoncabruno/spring-batch-lab:v1.3.0"

Depois do patch (com tag v1.4.0):
/spec/workflowSpec/templates/0/container/image = "mendoncabruno/spring-batch-lab:v1.4.0"
```

O CronWorkflow existente é **modificado in-place** — sem downtime, sem recriar o recurso.

---

## 8. Secrets e Segurança

### Secrets configurados no repositório GitHub

| Secret | Uso |
|--------|-----|
| `DOCKERHUB_USERNAME` | Login no Docker Hub (usuário) |
| `DOCKERHUB_TOKEN` | Login no Docker Hub (Access Token, não senha) |
| `GITHUB_TOKEN` | Gerado automaticamente pelo GitHub; usado para criar tags e releases |

### Boas Práticas Adotadas

1. **Docker Access Token em vez de senha:** Tokens têm escopo limitado (apenas push/pull) e podem ser revogados independentemente da conta
2. **`GITHUB_TOKEN` com escopo mínimo:** Declarado explicitamente `permissions: contents: write` — apenas o necessário
3. **Secrets nunca em código:** Nenhuma credencial hard-coded; tudo via `${{ secrets.* }}`
4. **Autenticação via token HTTPS:** `https://x-access-token:${GITHUB_TOKEN}@github.com/...` em vez de SSH keys no runner

---

## 9. Estratégia de Versionamento

O projeto adota **Semantic Versioning (SemVer)**: `MAJOR.MINOR.PATCH`

| Componente | Quando incrementar |
|------------|-------------------|
| `MAJOR` | Mudanças incompatíveis na API |
| `MINOR` | Novas funcionalidades backwards-compatible |
| `PATCH` | Bug fixes |

### Fluxo de Versão

1. Durante desenvolvimento: versão pode ter sufixo `-SNAPSHOT` (ex: `1.5.0-SNAPSHOT`)
2. Antes do merge em `main`: remover `-SNAPSHOT` e incrementar conforme necessário
3. O CI **bloqueia** merges com versão SNAPSHOT em `main`
4. O CI **bloqueia** re-uso de uma tag já existente
5. Uma vez tagueada, uma versão é **imutável**

### Annotated Tags vs Lightweight Tags

```bash
# Lightweight tag — apenas um ponteiro para um commit
git tag v1.4.0

# Annotated tag — objeto próprio com metadados
git tag -a v1.4.0 -m "v1.4.0"
```

O projeto usa **annotated tags** (`git tag -a`) pois:
- Têm autor, data e mensagem armazenados
- São tratadas como objetos de primeira classe pelo Git
- São as recomendadas para releases públicas

---

## 10. Tecnologias Utilizadas — Resumo

| Tecnologia | Versão/Variante | Papel no Pipeline |
|------------|----------------|-------------------|
| **GitHub Actions** | — | Plataforma de CI/CD; orquestração dos workflows |
| **Self-Hosted Runner** | — | Executor dos jobs; tem acesso ao cluster local |
| **Maven** | Wrapper (`./mvnw`) | Build do projeto Java, extração de versão |
| **Docker** | — | Containerização da aplicação |
| **Docker Buildx** | v3 | Build avançado de imagens (multi-plataforma, cache) |
| **Docker Hub** | — | Registry público para armazenamento das imagens |
| **eclipse-temurin:17-jre-alpine** | JRE 17 | Base da imagem Docker (runtime Java mínimo) |
| **Kubernetes** | — | Plataforma de orquestração dos containers em produção |
| **kubectl** | — | CLI para interagir com o cluster Kubernetes |
| **Argo Workflows** | — | Motor de execução de workflows no Kubernetes |
| **actions/checkout@v4** | v4 | Action para clonar o repositório no runner |
| **docker/login-action@v3** | v3 | Action para autenticar no Docker Hub |
| **docker/setup-buildx-action@v3** | v3 | Action para configurar o Docker Buildx |
| **docker/build-push-action@v5** | v5 | Action para build e push da imagem Docker |
| **softprops/action-gh-release@v2** | v2 | Action para criar GitHub Releases |

---

## 11. Diagrama de Relacionamento entre Componentes

```
┌──────────────────────────────────────────────────────────────────────────┐
│                           GITHUB                                         │
│                                                                          │
│  ┌──────────────┐     push/merge     ┌─────────────────────────────┐    │
│  │   Branch     │ ─────────────────► │   CI Workflow (ci.yml)      │    │
│  │   main       │                    │                             │    │
│  └──────────────┘                    │  1. Extrai versão pom.xml   │    │
│                                      │  2. Valida (não-SNAPSHOT)   │    │
│  ┌──────────────┐                    │  3. Verifica tag duplicada  │    │
│  │   Releases   │◄───────────────────│  4. Cria tag git            │    │
│  │   Tags       │   cria release     │  5. Cria GitHub Release     │    │
│  └──────────────┘                    │  6. mvn clean package       │    │
│                                      │  7. docker build + push     │    │
│  ┌──────────────┐    manual trigger  └──────────────┬──────────────┘    │
│  │   CD Workflow│◄────────────────────────── ops    │                   │
│  │  (cd.yml)    │    input: tag                     │                   │
│  └──────┬───────┘                                   │                   │
└─────────┼─────────────────────────────────────────  │  ─────────────────┘
          │                                           │
          │ kubectl patch                             │ docker push
          ▼                                           ▼
┌─────────────────────┐                   ┌──────────────────────┐
│   Kubernetes        │                   │     Docker Hub       │
│   Cluster           │                   │                      │
│                     │                   │  spring-batch-lab:   │
│  namespace: argo    │                   │    :v1.4.0           │
│                     │                   │    :latest           │
│  ┌───────────────┐  │◄──────────────────└──────────────────────┘
│  │ CronWorkflow  │  │  imagePullPolicy: Always
│  │ (a cada 15min)│  │
│  └───────┬───────┘  │
│          │ spawn    │
│  ┌───────▼───────┐  │
│  │  Pod          │  │
│  │  (batch job)  │  │
│  └───────────────┘  │
└─────────────────────┘
```

---

*Documento gerado em 2026-02-22 com base na versão 1.4.0 do projeto.*
