# Docker DB — PostgreSQL local

Container PostgreSQL para desenvolvimento do Viral Content Factory.

## Como usar

```bash
cd docker-db
cp .env.example .env    # ajuste credenciais se precisar
docker compose up -d    # sobe o Postgres em background
```

O banco fica disponível em `localhost:5433`.

## Conectar do backend

A `DATABASE_URL` no `.env.example` já está no formato Prisma:

```
postgresql://vcf:vcf@localhost:5433/viral_content_factory
```

Copie essa URL para o `.env` do backend quando for usar Prisma ou outro ORM.

## Comandos

| Ação | Comando |
|---|---|
| Subir | `docker compose up -d` |
| Parar | `docker compose down` |
| Parar e apagar dados | `docker compose down -v` |
| Ver logs | `docker compose logs -f postgres` |
| Status | `docker compose ps` |

## Credenciais padrão

- Usuário: `vcf`
- Senha: `vcf`
- Banco: `viral_content_factory`
- Porta: `5433`

> Altere no `.env` antes de subir se quiser credenciais diferentes.
