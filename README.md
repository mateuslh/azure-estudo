# azure-estudo

Hub dos projetos de estudo Microsoft Azure. Cada submodule é um repositório independente com infraestrutura, código e pipeline próprios.

## Projetos

| Repo | O que é | Stack |
|---|---|---|
| [adp-azure-estudo](https://github.com/mateuslh/adp-azure-estudo) | Banco de dados PostgreSQL | Terraform, Azure PostgreSQL Flexible Server |
| [faa-azure-estudo](https://github.com/mateuslh/faa-azure-estudo) | API REST de Pessoas | Python, FastAPI, Azure Container App |
| [swa-azure-estudo](https://github.com/mateuslh/swa-azure-estudo) | Frontend estático sobre Azure | React, Vite, Azure Static Web App |

## Arquitetura

```
                    ┌─────────────────────────────┐
                    │  swa-azure-estudo            │
                    │  React + Azure Static Web App│
                    └─────────────────────────────┘

                    ┌─────────────────────────────┐
                    │  faa-azure-estudo            │
                    │  FastAPI + Container App     │
                    │  GET /pessoas                │
                    │  POST /pessoas               │
                    └──────────────┬──────────────┘
                                   │
                    ┌──────────────▼──────────────┐
                    │  adp-azure-estudo            │
                    │  PostgreSQL Flexible Server  │
                    │  tabela: pessoas             │
                    └─────────────────────────────┘

           Tudo dentro de: rg-azure-estudo (brazilsouth)
           State Terraform: stterraformadpstate / tfstate
```

## Clonando tudo de uma vez

```sh
git clone --recurse-submodules git@github.com:mateuslh/azure-estudo.git
```

Ou se já clonou sem submodules:

```sh
git submodule update --init --recursive
```

## Atualizando os submodules

```sh
git submodule update --remote --merge
```
