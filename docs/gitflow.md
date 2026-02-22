# GitFlow — Guia de Uso Diário

Este documento descreve o fluxo de branches adotado no projeto `spring-batch-lab` seguindo o modelo GitFlow, com os comandos Git exatos para cada cenário.

---

## Estrutura de Branches

| Branch         | Propósito                                          | Origem    | Destino final        |
|----------------|----------------------------------------------------|-----------|----------------------|
| `main`         | Código em produção, sempre estável                 | —         | —                    |
| `develop`      | Branch de integração contínua                      | `main`    | —                    |
| `feature/*`    | Desenvolvimento de novas funcionalidades           | `develop` | `develop`            |
| `release/*`    | Preparação e ajustes finos para uma release        | `develop` | `main` + `develop`   |
| `hotfix/*`     | Correções urgentes direto em produção              | `main`    | `main` + `develop`   |

---

## 1. Configuração Inicial (uma única vez)

```bash
# Clone o repositório
git clone git@github.com:mendonca-bruno/spring-batch-lab.git
cd spring-batch-lab

# Crie a branch develop a partir de main (se ainda não existir remotamente)
git checkout -b develop
git push -u origin develop
```

---

## 2. Feature — Nova funcionalidade

Toda feature é desenvolvida em isolamento e integrada ao `develop` via PR.

```bash
# 1. Garanta que develop está atualizado
git checkout develop
git pull origin develop

# 2. Crie a branch de feature
git checkout -b feature/nome-da-feature

# 3. Desenvolva e faça commits normalmente
git add .
git commit -m "feat: descrição da mudança"

# 4. Mantenha a branch sincronizada com develop (periodicamente)
git fetch origin
git rebase origin/develop

# 5. Envie para o repositório remoto
git push -u origin feature/nome-da-feature

# 6. Abra um Pull Request: feature/nome-da-feature → develop
# Após aprovação e merge, delete a branch:
git branch -d feature/nome-da-feature
git push origin --delete feature/nome-da-feature
```

---

## 3. Release — Preparando uma versão para produção

A branch `release/*` congela o escopo, **atualiza a versão no `pom.xml`** e é mergeada em `main` via PR.
O pipeline detecta o merge, lê a versão do `pom.xml`, cria a tag Git automaticamente e executa o build/push/deploy.

> **Contrato:** a versão no `pom.xml` deve estar atualizada antes do merge em `main`.
> Se a tag `vX.Y.Z` já existir, o pipeline falha com uma mensagem de erro clara.

```bash
# 1. Garanta que develop está atualizado
git checkout develop
git pull origin develop

# 2. Crie a branch de release
git checkout -b release/1.0.0

# 3. Atualize a versão no pom.xml — este é o único passo obrigatório
mvn versions:set -DnewVersion=1.0.0 -DgenerateBackupPoms=false
git add pom.xml
git commit -m "chore: bump version to 1.0.0"

# 4. Faça ajustes finais se necessário (bugfixes menores, changelog, etc.)
# git add . && git commit -m "chore: prepare release 1.0.0"

# 5. Envie para o remoto e abra PR: release/1.0.0 → main
git push -u origin release/1.0.0

# --- Após aprovação e merge do PR em main ---
# O pipeline CI/CD dispara automaticamente, lê "1.0.0" do pom.xml,
# cria a tag v1.0.0, constrói o JAR, faz push das imagens Docker
# e aplica o workflow no Argo.

# 6. Integre as mudanças de volta ao develop
git checkout develop
git pull origin develop
git merge --no-ff release/1.0.0
git push origin develop

# 7. Delete a branch de release
git branch -d release/1.0.0
git push origin --delete release/1.0.0
```

---

## 4. Hotfix — Correção urgente em produção

Um hotfix parte diretamente de `main`, corrige o problema e é integrado tanto em `main` quanto em `develop`.

```bash
# 1. Parta de main atualizado
git checkout main
git pull origin main

# 2. Crie a branch de hotfix
git checkout -b hotfix/1.0.1

# 3. Aplique a correção e faça commit
git add .
git commit -m "fix: descrição da correção urgente"

# 4. Envie e abra PR: hotfix/1.0.1 → main
git push -u origin hotfix/1.0.1

# --- Após aprovação e merge do PR em main ---

# 5. Após merge do PR em main, o pipeline CI/CD dispara automaticamente
# e cria a tag v1.0.1 a partir da versão no pom.xml

# 6. Integre a correção ao develop
git checkout develop
git merge --no-ff hotfix/1.0.1
git push origin develop

# 7. Delete a branch de hotfix
git branch -d hotfix/1.0.1
git push origin --delete hotfix/1.0.1
```

---

## 5. Diagrama do Fluxo

```
main       ─────────────●─────────────────────────────●──────●──▶
                         \                            /↑      /↑
release/1.0.0             ●────●────●────────────────● tag   /  │
                               ↑                            /   │
develop    ──────────────●─────●──────────────────●─────────●───●──▶
                          \                      /↑
feature/x                  ●────●────●───────────●
```

- Cada `●` em `main` com tag `v*.*.*` dispara o CI/CD.
- `release/*` e `hotfix/*` entram em `main` obrigatoriamente via PR.
- `feature/*` entra em `develop` obrigatoriamente via PR.

---

## 6. Regras de Branch Protection (configurar manualmente no GitHub)

Consulte o [resumo de branch protection rules](./branch-protection-rules.md) para os detalhes de configuração.
