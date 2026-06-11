# azure-estudo

Hub dos projetos de estudo **Microsoft Azure**. Três repositórios independentes (Git submodules) que juntos formam uma aplicação fullstack na Azure — banco de dados, API e frontend — cada um com infraestrutura (Terraform) e pipeline (GitHub Actions) próprios.

> 📐 Documentação completa de arquitetura, cloud, DevOps e processo de deploy: **[ARQUITETURA.md](ARQUITETURA.md)**

## Projetos

| Repo | O que é | Stack |
|---|---|---|
| [adp-azure-estudo](https://github.com/mateuslh/adp-azure-estudo) | Banco de dados | Terraform, Azure PostgreSQL Flexible Server |
| [faa-azure-estudo](https://github.com/mateuslh/faa-azure-estudo) | API REST de Pessoas | Python, FastAPI, Docker, Azure Container Apps |
| [swa-azure-estudo](https://github.com/mateuslh/swa-azure-estudo) | Site técnico sobre Azure | React, Vite, Azure Static Web Apps |

## Arquitetura (resumo)

```
React (Static Web App) ──► FastAPI (Container App) ──► PostgreSQL (Flexible Server)

Resource Group: rg-azure-estudo (brazilsouth)
IaC: Terraform com state no Azure Blob Storage
CI/CD: GitHub Actions — push na main = deploy automático
```

## Como clonar

```sh
git clone --recurse-submodules git@github.com:mateuslh/azure-estudo.git
```

Se já clonou sem submodules:

```sh
git submodule update --init --recursive
```

Para atualizar os submodules:

```sh
git submodule update --remote --merge
```
