[English](README.md) | **Português**

# Palco

Uma plataforma de venda de ingressos para eventos, construída como projeto de estudo full-stack com Angular 22.

## Sobre

O Palco permite que o público encontre eventos e compre ingressos, oferece aos produtores um painel para gerenciar eventos, lotes e vendas, e dá à equipe da portaria um app para fazer o check-in dos participantes em tempo real.

O projeto gira em torno de um desafio central: **abrir as vendas de um show para 200 mil pessoas ao mesmo tempo, sem derrubar o sistema e sem vender o mesmo ingresso duas vezes.**

## Status

**Em desenvolvimento. Ainda sem código da aplicação.**

O projeto está na primeira fase: fundamentos de rede e HTTP, e definição do contrato da API antes de escrever qualquer código. O desenvolvimento do primeiro app (`palco-site`) começa na próxima etapa.

## Arquitetura planejada

Tudo abaixo está **planejado**. A construção é feita passo a passo, começando com um único app Angular e crescendo até um monorepo Nx.

| Parte | Para quem | O que faz |
| --- | --- | --- |
| `palco-site` | Público | Vitrine de eventos, busca, detalhe do evento, carrinho e checkout |
| `palco-admin` | Produtores de eventos | Eventos e lotes, pedidos, relatórios e dashboards |
| `palco-checkin` | Equipe da portaria | Lista de participantes, validação de ingressos em tempo real, programação do evento |
| `libs/ui` | Todos os apps | Design System compartilhado |
| `api` | Todos os frontends | API REST e WebSocket, autenticação |
| `worker` | Sistema | Emissão de ingressos, e-mails, PDFs e eventos de check-in |

## Tecnologias

- **Frontend:** Angular 22 (signals, zoneless, SSR), NgRx SignalStore e NgRx Store, Tailwind CSS, Spartan UI, Angular Aria, Angular Material e CDK, PrimeNG, ngx-echarts, Transloco
- **Backend:** NestJS, OpenAPI, JWT, WebSocket
- **Dados:** PostgreSQL com Prisma, MongoDB, Redis, RabbitMQ
- **Infraestrutura:** Nx, Docker, Kubernetes, Azure, GitHub Actions
- **Testes:** Vitest, Angular Testing Library, Playwright

## Documentação

As anotações de design e os diagramas ficam em [`docs/`](docs/). Os diagramas em [`docs/diagramas/`](docs/diagramas/) são arquivos do [Excalidraw](https://excalidraw.com): abra-os em excalidraw.com ou com a extensão do Excalidraw para o VS Code. O contrato da API da vitrine está em [`docs/api-contract.md`](docs/api-contract.md) (em inglês).

## Autor

**Pedro Martins**: [LinkedIn](https://www.linkedin.com/in/pedro2312/)
