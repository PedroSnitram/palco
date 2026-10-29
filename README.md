**English** | [Português](README.pt-BR.md)

# Palco

A ticketing platform for live events, built as a full-stack Angular 22 study project.

## About

Palco lets the public discover events and buy tickets, gives organizers a dashboard to manage events, ticket batches and sales, and gives venue staff an app to check attendees in at the gate in real time.

The project is built around one central challenge: **opening ticket sales for 200,000 people at the same time without taking the system down and without selling the same ticket twice.**

## Status

**In progress. No application code yet.**

The project is in its first phase: networking and HTTP fundamentals, and designing the API contract before writing any code. Development of the first app (`palco-site`) starts next.

## Planned architecture

Everything below is **planned**. It will be built step by step, starting with a single Angular app and growing into an Nx monorepo.

| Part | For | What it does |
| --- | --- | --- |
| `palco-site` | Public | Event showcase, search, event details, cart and checkout |
| `palco-admin` | Event organizers | Events and ticket batches, orders, reports and dashboards |
| `palco-checkin` | Venue staff | Attendee list, real-time ticket validation, event schedule |
| `libs/ui` | All apps | Shared design system |
| `api` | All frontends | REST and WebSocket API, authentication |
| `worker` | System | Ticket issuing, emails, PDFs and check-in events |

## Tech stack

- **Frontend:** Angular 22 (signals, zoneless, SSR), NgRx SignalStore and NgRx Store, Tailwind CSS, Spartan UI, Angular Aria, Angular Material and CDK, PrimeNG, ngx-echarts, Transloco
- **Backend:** NestJS, OpenAPI, JWT, WebSocket
- **Data:** PostgreSQL with Prisma, MongoDB, Redis, RabbitMQ
- **Infrastructure:** Nx, Docker, Kubernetes, Azure, GitHub Actions
- **Testing:** Vitest, Angular Testing Library, Playwright

## Documentation

Design notes and diagrams live in [`docs/`](docs/). The diagrams in [`docs/diagramas/`](docs/diagramas/) are [Excalidraw](https://excalidraw.com) files: open them at excalidraw.com or with the Excalidraw extension for VS Code. The showcase API contract is in [`docs/api-contract.md`](docs/api-contract.md).

## Author

**Pedro Martins**: [LinkedIn](https://www.linkedin.com/in/pedro2312/)
