[English](README.md) | **Português**

# Palco

Uma plataforma de venda de ingressos para eventos, construída como projeto de estudo full-stack com Angular 22.

## Sobre

O Palco permite que o público encontre eventos e compre ingressos, oferece aos produtores um painel para gerenciar eventos, lotes e vendas, e dá à equipe da portaria um app para fazer o check-in dos participantes em tempo real.

O projeto gira em torno de um desafio central: **abrir as vendas de um show para 200 mil pessoas ao mesmo tempo, sem derrubar o sistema e sem vender o mesmo ingresso duas vezes.**

## Status

**Em desenvolvimento. Ainda sem código da aplicação.**

A primeira fase, fundamentos de rede e HTTP e o contrato da API, está concluída. O desenvolvimento do primeiro app (`palco-site`) começa na próxima etapa.

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

As anotações de design e os diagramas ficam em [`docs/`](docs/). Os diagramas em [`docs/diagramas/`](docs/diagramas/) são arquivos do [Excalidraw](https://excalidraw.com): abra-os em excalidraw.com ou com a extensão do Excalidraw para o VS Code. O contrato da API da vitrine está em [`docs/api-contract.md`](docs/api-contract.md), e a análise de sites de ingressos reais, em [`docs/ticket-sites-analysis.md`](docs/ticket-sites-analysis.md) (ambos em inglês).

## Diário de aprendizado

### Módulo 00: Client/Server, HTTP e rede

- **Aprendi:** o caminho de uma requisição (DNS, TCP, TLS, HTTP) e como ler cada etapa, e cada tipo de falha, na aba Network do DevTools.
- **Aprendi:** HTTP e REST: métodos, status codes, headers, cookies, cache (`Cache-Control`, `ETag` e `304`), recursos no plural, idempotência e paginação.
- **Aprendi:** CORS e a same-origin policy, e quando usar polling, SSE ou WebSocket. A análise do Sympla e da Eventim mostrou que nenhum dos dois faz polling da disponibilidade e que os dois usam uma sala de espera virtual na CDN.
- **Decisão:** escrevi o contrato da API antes de qualquer código, em vez de deixar o formato surgir durante a implementação.
- **Por quê:** a API simulada (Módulo 06) e a API real (Módulo 18) seguem o mesmo documento, então trocar uma pela outra quase não mexe no frontend.

## Autor

**Pedro Martins**: [LinkedIn](https://www.linkedin.com/in/pedro2312/)
