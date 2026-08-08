# 014 — Prisma + PostgreSQL persistence

## Objetivo

Persistir jobs de renderização no PostgreSQL via Prisma, substituindo o store em memória usado pela API.

## Escopo implementado

- Adicionar Prisma ao backend.
- Criar schema Prisma com `RenderJob` e `VideoOutput`.
- Criar migration inicial para PostgreSQL.
- Criar singleton `PrismaClient` em `src/shared/database/prisma.ts`.
- Refatorar `src/api/job-store.ts` para usar Prisma em vez de `Map`.
- Atualizar rotas de job para chamadas assíncronas e persistência de `rawText`.
- Expor `GET /jobs` para listar jobs persistidos.
- Expor `DATABASE_URL` no `.env.example` do backend.

## Validação

- `bun run typecheck`: passou.
- `bun run db:generate`: passou.
- `bun run db:migrate -- --name init`: passou.
- Job criado pela API sobreviveu ao restart do servidor.
- Download do MP4 persistido validado.

## Observações

- PostgreSQL local usa `docker-db/` em `localhost:5433`.
- Prisma e `@prisma/client` fixados em `6.19.3`.
