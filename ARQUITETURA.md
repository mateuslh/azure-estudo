# azure-estudo — Documentação Completa

Documentação consolidada do projeto de estudo Microsoft Azure: arquitetura, infraestrutura cloud, DevOps, tecnologias e o processo completo de provisionamento e deploy.

---

## 1. Visão geral

O projeto é um **hub de três repositórios independentes** (Git submodules), que juntos formam uma aplicação fullstack completa na Azure — banco de dados, API e frontend — cada um com sua própria infraestrutura como código e pipeline de CI/CD.

| Repo | Papel | Serviço Azure |
|---|---|---|
| [adp-azure-estudo](https://github.com/mateuslh/adp-azure-estudo) | Banco de dados | PostgreSQL Flexible Server |
| [faa-azure-estudo](https://github.com/mateuslh/faa-azure-estudo) | API REST de Pessoas | Container Apps + Container Registry |
| [swa-azure-estudo](https://github.com/mateuslh/swa-azure-estudo) | Site técnico sobre Azure | Static Web Apps |

A escolha de submodules permite que cada projeto tenha ciclo de vida, pipeline e versionamento próprios, enquanto o hub oferece uma visão unificada e clone único:

```sh
git clone --recurse-submodules git@github.com:mateuslh/azure-estudo.git
```

---

## 2. Arquitetura na Azure

```
                         Internet
                            │
        ┌───────────────────┼───────────────────────┐
        ▼                                           ▼
┌──────────────────────┐                ┌──────────────────────────┐
│ Azure Static Web App │                │ Azure Container App      │
│ swa-azure-estudo     │                │ ca-faa-pessoas           │
│ React 19 + Vite      │                │ FastAPI (Python 3.11)    │
│ CDN global + HTTPS   │                │ GET /pessoas             │
│ SKU Free (eastus2)   │                │ POST /pessoas            │
└──────────────────────┘                │ 0.25 vCPU / 0.5 GB       │
                                        │ escala 0 → 3 réplicas    │
                                        └────────────┬─────────────┘
                                                     │ psycopg2 (SSL)
                                        ┌────────────▼─────────────┐
                                        │ PostgreSQL Flexible      │
                                        │ Server 16                │
                                        │ B_Standard_B1ms          │
                                        │ banco: adp_test          │
                                        │ tabela: pessoas          │
                                        └──────────────────────────┘

  Resource Groups: rg-azure-estudo (banco + API) e rg-swa-azure-estudo (frontend), ambos em brazilsouth
  Observabilidade: Log Analytics Workspace (log-faa-func)
  State Terraform: Azure Blob Storage (stterraformadpstate / tfstate)
```

### Recursos provisionados

| Recurso | Nome | SKU / Tier | Justificativa |
|---|---|---|---|
| Resource Group | `rg-azure-estudo` | — | Container lógico compartilhado por banco e API, em `brazilsouth` |
| Resource Group | `rg-swa-azure-estudo` | — | RG próprio do frontend, gerenciado pelo Terraform do swa |
| PostgreSQL Flexible Server | `psql-adp-test-<sufixo>` | `B_Standard_B1ms` (1 vCore, 2 GB, 32 GB disco) | Menor e mais barato; Flexible Server é o tier recomendado (Single Server descontinuado) |
| Container Registry | `acrfaaazure<sufixo>` | Basic, admin enabled | Armazena a imagem Docker da API |
| Log Analytics Workspace | `log-faa-func` | PerGB2018, retenção 30 dias | Logs do Container App Environment |
| Container App Environment | `cae-faa-func` | Consumption | Ambiente gerenciado dos containers |
| Container App | `ca-faa-pessoas` | 0.25 vCPU / 0.5Gi, 0–3 réplicas | Escala a zero = custo zero quando ocioso |
| Static Web App | `swa-azure-estudo` | Free (`eastus2`) | Free não disponível em `brazilsouth`; CDN global + HTTPS automático + previews por PR |

### Decisões de arquitetura

- **Resource Group compartilhado:** `rg-azure-estudo` é pré-existente e apenas *referenciado* (`data` source) pelos Terraforms do banco e da API — nenhum repo é dono dele, evitando que um `destroy` derrube tudo. Já o frontend cria e gerencia seu próprio RG (`rg-swa-azure-estudo`), pois não compartilha recursos com os demais.
- **Sufixos aleatórios** (`random_string`) nos nomes do servidor PostgreSQL e do ACR, porque ambos exigem nomes globalmente únicos na Azure.
- **Firewall do PostgreSQL** com regra `0.0.0.0–0.0.0.0`, que na Azure significa "somente serviços Azure" (Container App e runners de pipeline), não internet aberta.
- **Segredos como `secret` do Container App** (`db-password`, `acr-password`) injetados como env vars — nunca em texto plano no template.
- **`lifecycle.ignore_changes`** em dois pontos críticos: a `zone` do PostgreSQL (atribuída automaticamente pela Azure, geraria drift) e o `template` do Container App (a pipeline atualiza a imagem via CLI; sem o ignore, o próximo `apply` reverteria para o placeholder).

---

## 3. Infraestrutura como Código (Terraform)

Cada repositório tem seu próprio root module Terraform, com **state remoto no Azure Blob Storage** e lock automático:

| Repo | Diretório | Backend (storage / container / key) | Provider azurerm |
|---|---|---|---|
| adp | `terraform/` | `stterraformadpstate` / `tfstate` / `adp-postgres.tfstate` | `~> 3.110` |
| faa | `terraform/` | `stterraformadpstate` / `tfstate` / `faa-function.tfstate` | `~> 3.110` |
| swa | `infra/` | `stmateuslhtfstate` / `tfstate` / `swa-azure-estudo.tfstate` | `~> 4.0` |

O storage account do state foi criado uma única vez via `terraform/bootstrap.sh` (repo adp) — resolvendo o problema clássico de "quem cria a infra que guarda o state".

Padrões aplicados em todos os módulos:

- Separação `main.tf` / `variables.tf` / `outputs.tf` / `providers.tf`
- `terraform.tfvars.example` versionado; o `.tfvars` real fica fora do Git
- `.terraform.lock.hcl` versionado para builds reproduzíveis
- Tags padronizadas (`environment=study`, `owner=mateuslh`, `product`, `managed_by=terraform`)
- Outputs encadeados entre projetos: o FQDN do PostgreSQL (output do adp) vira o secret `DB_HOST` consumido pela pipeline do faa

---

## 4. DevOps — Pipelines de CI/CD (GitHub Actions)

Os três repos seguem o mesmo princípio: **push na `main` = deploy automático**; ações destrutivas só via `workflow_dispatch` manual. Autenticação na Azure por **Service Principal** (secrets `AZURE_CLIENT_ID`, `AZURE_CLIENT_SECRET`, `AZURE_SUBSCRIPTION_ID`, `AZURE_TENANT_ID` exportados como `ARM_*`).

### adp-azure-estudo — `terraform.yml`

```
PR (paths: terraform/**)  → init → fmt-check → validate → plan → comenta o plan no PR
push main                 → ... → plan → apply automático
manual (dispatch)         → escolhe plan | apply | destroy
```

Destaques: o resultado do `terraform plan` é postado como comentário no PR (`actions/github-script`), dando revisão de infraestrutura antes do merge; o job usa GitHub Environments (`production` na main, `preview` em PRs).

### faa-azure-estudo — `deploy.yml` (pipeline em 3 estágios)

```
push main
   │
   ▼
[1] terraform ──► apply + captura outputs (acr_login_server, acr_name, app_url)
   │
   ▼
[2] migrate ────► pip install psycopg2 → migrate.py roda migrations/*.sql em ordem
   │
   ▼
[3] build-and-deploy
        ├── az acr login
        ├── docker build -t pessoas-api:<git-sha> -t :latest
        ├── docker push (ambas as tags)
        └── az containerapp update --image ...:<git-sha>
```

Destaques:
- **Infra antes do código:** o Terraform garante que ACR e Container App existem antes do build.
- **Migration antes do deploy:** o schema é atualizado antes da nova versão da API subir, evitando código novo contra schema antigo.
- **Imagem taggeada com o SHA do commit** — cada deploy é rastreável até o commit exato, e `latest` acompanha por conveniência.
- **Padrão placeholder:** o Terraform cria o Container App com uma imagem hello-world da Microsoft (quebrando o ciclo "o app precisa da imagem, mas o ACR ainda não existe"); o estágio 3 troca para a imagem real.

### swa-azure-estudo — `deploy.yml` + `terraform.yml`

```
push main  → Azure/static-web-apps-deploy@v1 → npm ci → npm run build (Vite) → upload de dist/ para a CDN
PR aberto  → mesma action cria um ambiente de preview com URL própria
PR fechado → job "close" destrói o ambiente de preview
```

A autenticação usa o **deployment token** do SWA (secret `AZURE_STATIC_WEB_APPS_API_TOKEN`), extraído via Terraform output / `az staticwebapp secrets list` e gravado com `gh secret set` — o token nunca aparece no código.

### Secrets por repositório

| Secret | adp | faa | swa |
|---|:---:|:---:|:---:|
| `AZURE_CLIENT_ID` / `CLIENT_SECRET` / `SUBSCRIPTION_ID` / `TENANT_ID` | ✅ | ✅ | — |
| `DB_ADMIN_PASSWORD` | ✅ | — | — |
| `DB_HOST` (FQDN, output do adp) | — | ✅ | — |
| `DB_PASSWORD` | — | ✅ | — |
| `AZURE_STATIC_WEB_APPS_API_TOKEN` | — | — | ✅ |

---

## 5. Tecnologias por camada

### Banco (adp)
- **PostgreSQL 16** — Azure Database for PostgreSQL Flexible Server
- **Terraform** (azurerm + random) — todo o provisionamento
- Backup automático de 7 dias, sem geo-redundância (estudo)

### API (faa)
- **Python 3.11** + **FastAPI** + **uvicorn**
- **Pydantic** para validação do payload (`nome`, `idade`, `tecnologia_que_o_gledson_ama`)
- **psycopg2** com `sslmode=require` (TLS obrigatório até o banco)
- **Docker** — imagem `python:3.11-slim`, build minimalista
- **Migrations SQL puras** (`migrations/V1__create_pessoas_table.sql`, convenção tipo Flyway) executadas por um `migrate.py` de ~30 linhas

Endpoints:

| Método | Rota | Descrição |
|---|---|---|
| `GET` | `/pessoas` | Lista todas as pessoas |
| `POST` | `/pessoas` | Cadastra pessoa (201 + registro criado) |

### Frontend (swa)
- **React 19 + Vite** — SPA
- **Tailwind CSS v4** — estilo
- **Framer Motion** — animações
- **d3-geo + topojson-client** — mapa mundial das regiões Azure (Natural Earth 110m)
- `staticwebapp.config.json` — fallback SPA (`/* → index.html`), cache imutável de 1 ano para `/assets/*` (seguro porque o Vite põe hash no nome dos arquivos) e headers de segurança (`X-Frame-Options`, `X-Content-Type-Options`, etc.)

Conteúdo do site: apresentação técnica do Azure para arquitetos — história, arquitetura ARM, catálogo de serviços, comparativo com AWS/GCP, cobertura global, preços/FinOps e governança (Landing Zone, Well-Architected Framework).

**URL de produção:** https://wonderful-flower-06970450f.7.azurestaticapps.net

---

## 6. Processo completo — do zero ao ar

Ordem de provisionamento (respeitando as dependências):

```
1. BOOTSTRAP        bash adp/terraform/bootstrap.sh
                    └─ cria o storage account do state remoto

2. BANCO (adp)      push/apply → PostgreSQL Flexible Server + banco adp_test + firewall
                    └─ output: FQDN do servidor → vira secret DB_HOST no repo faa

3. API (faa)        push na main dispara a pipeline de 3 estágios:
                    ├─ terraform apply  → ACR + Log Analytics + Environment + Container App
                    ├─ migrate.py       → cria/atualiza tabela pessoas
                    └─ docker build/push + az containerapp update → API no ar

4. FRONTEND (swa)   terraform apply → Static Web App
                    ├─ deployment token → gh secret set AZURE_STATIC_WEB_APPS_API_TOKEN
                    └─ push na main → build Vite → upload para a CDN global
```

A partir daí o ciclo de vida é totalmente automatizado:

- **Mudança de infra** → PR → `terraform plan` comentado → merge → `apply`
- **Mudança na API** → push → migration → nova imagem por SHA → nova revisão do Container App
- **Mudança no site** → push → build → CDN; PRs ganham preview com URL própria, destruído ao fechar
- **Teardown** → `workflow_dispatch` com `destroy` em cada repo (nunca automático)

### Características FinOps do desenho

- Container App com `min_replicas = 0` — custo zero sem tráfego
- PostgreSQL no menor SKU burstable disponível
- Static Web App no tier Free
- Tags de custo/ownership em todos os recursos
- Destroy manual disponível em todas as pipelines para encerrar o ambiente de estudo
