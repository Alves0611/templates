# Terraform Pipeline

Repositório centralizado de pipelines Terraform para GitHub Actions. Este repositório fornece workflows reutilizáveis e actions compostas para automatizar validação, planejamento e aplicação de infraestrutura como código com Terraform na AWS.

## 🎯 Objetivo

Repositórios de projetos **NÃO devem copiar steps** de Terraform. Em vez disso, devem apenas chamar o workflow central:

```yaml
uses: ORG/terraform-pipeline/.github/workflows/core-terraform.yml@v1
```

O `core-terraform.yml` orquestra automaticamente o fluxo correto baseado no evento (PR, push para main, ou workflow_dispatch).

## 📋 Estrutura do Repositório

```
.
├── .github/
│   └── workflows/
│       ├── core-terraform.yml        # Orquestrador principal (chamado pelos consumidores)
│       ├── terraform-plan.yml        # Executa scans, validações e plan
│       └── terraform-apply.yml       # Executa apply
├── actions/
│   ├── setup-terraform/
│   │   ├── action.yml                # Composite action: Setup Terraform
│   │   └── README.md
│   └── render-summary/
│       ├── action.yml                # Composite action: Step Summary
│       └── README.md
└── README.md
```

## 🚀 Como Usar

### Exemplo Básico

Crie um arquivo `.github/workflows/iac.yml` no seu repositório de projeto:

```yaml
name: Infrastructure as Code

on:
  pull_request:
    branches:
      - main
  push:
    branches:
      - main

jobs:
  terraform:
    uses: ORG/terraform-pipeline/.github/workflows/core-terraform.yml@v1
    with:
      aws_region: us-east-1
      working_directory: terraform
      tf_version: "1.7.5"
      enable_infracost: true
      enable_trivy: true
      enable_checkov: true
      enable_tflint: true
      enable_tfsec: false
      plan_on_push_main: true
      apply_on_push_main: true
      comment_plan_on_pr: true
    secrets:
      AWS_ASSUME_ROLE: ${{ secrets.AWS_ASSUME_ROLE }}
      INFRACOST_API_KEY: ${{ secrets.INFRACOST_API_KEY }}
```

### Inputs Obrigatórios

| Input | Tipo | Descrição |
|-------|------|-----------|
| `aws_region` | string | Região AWS a ser usada |

### Secrets Obrigatórias

| Secret | Descrição |
|--------|-----------|
| `AWS_ASSUME_ROLE` | ARN da role OIDC para todas as operações (plan e apply) |

### Inputs Opcionais

| Input | Tipo | Default | Descrição |
|-------|------|---------|-----------|
| `tf_version` | string | `1.7.5` | Versão do Terraform |
| `working_directory` | string | `.` | Diretório de trabalho do Terraform |
| `backend_config_file` | string | `""` | Caminho opcional para arquivo de configuração do backend |
| `enable_infracost` | boolean | `true` | Habilitar estimativa de custo com Infracost |
| `enable_trivy` | boolean | `true` | Habilitar scan de segurança com Trivy |
| `enable_checkov` | boolean | `true` | Habilitar scan de segurança com Checkov |
| `enable_tflint` | boolean | `true` | Habilitar TFLint |
| `enable_tfsec` | boolean | `false` | Habilitar scan de segurança com TFSec |
| `plan_on_push_main` | boolean | `true` | Executar plan no push para main |
| `apply_on_push_main` | boolean | `true` | Executar apply no push para main |
| `comment_plan_on_pr` | boolean | `true` | Comentar resumo do plan no PR |

### Secrets

| Secret | Obrigatório | Descrição |
|--------|-------------|-----------|
| `AWS_ASSUME_ROLE` | Sim | ARN da role OIDC para todas as operações (plan e apply) |
| `INFRACOST_API_KEY` | Não | Chave da API do Infracost para estimativa de custo |

## 🔐 Autenticação AWS via OIDC

Este pipeline usa **OIDC (OpenID Connect)** para autenticação na AWS, eliminando a necessidade de armazenar access keys como secrets.

### Configuração da Role IAM

1. Crie uma role IAM com as permissões necessárias para Terraform.

2. Configure a trust policy da role para permitir o GitHub Actions assumir a role:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:ORG/terraform-pipeline:*"
        }
      }
    }
  ]
}
```

**Nota:** Ajuste o `sub` para corresponder ao seu repositório ou organização. Exemplos:
- `repo:ORG/*:*` - Todos os repositórios da organização
- `repo:ORG/my-repo:*` - Repositório específico
- `repo:ORG/my-repo:ref:refs/heads/main` - Apenas branch main

3. Para projetos consumidores, use:
```json
"token.actions.githubusercontent.com:sub": "repo:ORG/my-project:*"
```

### Variáveis Necessárias

Configure a seguinte secret no seu repositório GitHub (Settings → Secrets and variables → Actions):

- `AWS_ASSUME_ROLE`: ARN da role OIDC para todas as operações (ex: `arn:aws:iam::123456789012:role/github-actions-terraform`)

**Nota:** A ARN é armazenada como secret para não expor o account ID nos workflows. A mesma role é usada para operações de plan e apply.

## 🛡️ Environment Apply (Gate de Aprovação)

O pipeline implementa um **gate obrigatório** para operações de apply através de um GitHub Environment chamado `apply`.

### Como Configurar

1. No repositório, vá em **Settings** → **Environments**
2. Crie um novo environment chamado `apply`
3. Configure **Required reviewers** (adicione usuários ou equipes que devem aprovar)
4. Opcionalmente, configure **Deployment branches** para restringir a branches específicas

### Comportamento

- **Pull Requests**: Apenas executa plan e scans (sem apply)
- **Push para main**: Se `apply_on_push_main=true`, requer aprovação no environment `apply` antes de executar apply
- **workflow_dispatch**: Se `mode=apply` e `run_apply=true`, também requer aprovação no environment `apply`

O gate garante que nenhum apply seja executado sem aprovação manual explícita.

## 📦 Versionamento

O pipeline é versionado usando **tags Git**. Use tags semânticas:

- `v1` - Versão estável
- `v1.0.1` - Patch release
- `v1.1.0` - Minor release

**Exemplo de uso:**
```yaml
uses: ORG/terraform-pipeline/.github/workflows/core-terraform.yml@v1
```

Recomendamos fixar a versão major (`@v1`) para receber patches automaticamente, ou fixar versão específica (`@v1.0.1`) para máxima estabilidade.

## 🔍 Ferramentas e Scans

### Terraform

- **fmt**: Verifica formatação (`terraform fmt -check -recursive`)
- **validate**: Valida configuração (`terraform validate -no-color`)
- **init**: Inicializa backend (suporta `backend_config_file`)
- **plan**: Gera plano (`terraform plan -out=tfplan`)
- **apply**: Aplica mudanças usando artifact do plan

### TFLint

Linter para Terraform. Habilitado por padrão (`enable_tflint: true`).

### Security Scans

#### Trivy
- Scan de configuração Terraform
- Falha em vulnerabilidades CRITICAL ou HIGH
- Habilitado por padrão

#### Checkov
- Análise estática de infraestrutura
- Framework: Terraform
- Habilitado por padrão

#### TFSec
- Scanner de segurança específico para Terraform
- Desabilitado por padrão (`enable_tfsec: false`)
- Falha em vulnerabilidades HIGH ou CRITICAL

### Infracost

Estimativa de custo de infraestrutura:
- Gera relatório JSON
- Comenta automaticamente no PR (se habilitado)
- Requer `INFRACOST_API_KEY` secret

## 📊 Step Summary

Todos os workflows geram um **Step Summary** bonito no final da execução, incluindo:
- Status de cada etapa (fmt, validate, lint, scans, plan)
- Link para artifact do plan
- Informações sobre Infracost (se habilitado)
- Status geral do pipeline

O summary é visível na aba "Summary" da execução do workflow.

## 🔄 Fluxo de Execução

### Pull Request

1. Validação de inputs
2. **Terraform Plan** (`terraform-plan.yml`):
   - fmt check
   - validate
   - init
   - Security Scans (Trivy, Checkov, TFSec - se habilitados)
   - tflint (se habilitado)
   - plan
   - upload artifact
   - Infracost (se habilitado e API key presente)
   - Comentar no PR (se habilitado)
3. Step Summary

### Push para Main

1. Validação de inputs
2. **Gate de Aprovação** (environment `apply`)
3. **Terraform Plan** (`terraform-plan.yml`) - se `plan_on_push_main=true`
4. **Terraform Apply** (`terraform-apply.yml`) - se `apply_on_push_main=true`, depende do plan
5. Step Summary

### Workflow Dispatch

- **mode=pr**: Executa pipeline de PR (sem apply)
- **mode=apply**: 
  - Valida que está na branch `main`
  - Valida que `run_apply=true`
  - Passa pelo gate de aprovação
  - Executa plan + apply

## 🔧 Configuração do Backend

Se você precisar passar configurações customizadas para o backend do Terraform, use o input `backend_config_file`:

```yaml
with:
  backend_config_file: terraform/backend.hcl
```

O arquivo deve conter variáveis de backend, por exemplo:
```hcl
bucket = "my-terraform-state"
key    = "prod/terraform.tfstate"
region = "us-east-1"
```

O pipeline executará `terraform init -backend-config=backend_config_file`.

## ❓ FAQ

### Por que o apply precisa de aprovação?

O apply modifica infraestrutura em produção. O gate de aprovação garante que mudanças críticas sejam revisadas manualmente antes de serem aplicadas, reduzindo o risco de incidentes.

### Como habilitar/desabilitar scanners?

Use os inputs `enable_*`:

```yaml
with:
  enable_trivy: true
  enable_checkov: true
  enable_tfsec: false
  enable_tflint: true
```

### Como configurar backend_config_file?

Crie um arquivo de configuração (ex: `backend.hcl`) e passe o caminho:

```yaml
with:
  backend_config_file: terraform/backend.hcl
```

### Posso usar roles separadas para plan e apply?

Sim, mas por padrão o pipeline usa a mesma role (`AWS_ASSUME_ROLE`) para todas as operações. Se você precisar de roles separadas, você pode:

1. Criar duas roles com permissões diferentes:
   - Role para plan: Permissões de leitura (Get, List, Describe)
   - Role para apply: Permissões completas (Create, Update, Delete)

2. Modificar o pipeline para usar roles diferentes (requer alteração no código do pipeline)

**Recomendação:** Para a maioria dos casos, uma única role com permissões apropriadas é suficiente e mais simples de gerenciar.

### O que acontece se plan falhar?

O apply não será executado. O workflow falha na etapa de plan.

### Como desabilitar apply automático no push?

```yaml
with:
  plan_on_push_main: true
  apply_on_push_main: false
```

Isso executará apenas o plan no push para main.

### Posso usar este pipeline com outros clouds?

Atualmente, o pipeline é otimizado para AWS com OIDC. Para outros clouds, seria necessário adaptar a autenticação e os workflows.

### Como funciona o cache do Terraform?

O pipeline usa cache baseado no hash de `.terraform.lock.hcl`. Isso acelera `terraform init` em execuções subsequentes.

## 📝 Licença

Este repositório é fornecido como está. Ajuste conforme necessário para suas necessidades.

## 🤝 Contribuindo

Para melhorias ou correções, abra uma issue ou pull request no repositório.

---

**Nota:** Substitua `ORG` pelos nomes reais da sua organização e ajuste ARNs e configurações conforme seu ambiente.

