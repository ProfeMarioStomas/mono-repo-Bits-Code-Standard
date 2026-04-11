# Este repositorio es una plantilla para un monorepo que cuenta con Backend y Frontend

Contiene reglas para que Claude Code siga sin necesidad de divagar por su cuenta

## Stack Tecnológico Backend
| Layer       | Technology                               |
| ----------- | ---------------------------------------- |
| Runtime     | Cloudflare Workers (edge)                |
| Framework   | Hono.js                                  |
| Database    | Neon PostgreSQL (serverless HTTP driver) |
| ORM         | Drizzle ORM                              |
| Validation  | Zod v4                                   |
| Cache       | In-memory API cache at the service layer |
| Local dev   | Wrangler + Miniflare                     |
| Testing     | Vitest + Miniflare E2E                   |

## Stack Tecnológico Frontend
| Layer           | Technology                   |
| --------------- | ---------------------------- |
| Bundler         | Vite                         |
| UI Framework    | React 19 (TypeScript)        |
| Server State    | TanStack Query v5            |
| Forms           | TanStack Form v1             |
| HTTP Client     | Axios                        |
| Styling         | Tailwind CSS v4              |
| Validation      | Zod v4                       |
