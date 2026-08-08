# 002 — Fundação do Backend Bun + TypeScript + Elysia

## Objetivo

Configurar o backend como serviço independente usando Bun, TypeScript e Elysia.

## Escopo

Esta atividade prepara o backend para receber CLI, API e pipeline.

## Dependências recomendadas

Runtime/dev:

- `typescript`
- `@types/bun`

API:

- `elysia`
- `@elysiajs/cors`

CLI/parsing:

- usar parser simples próprio ou biblioteca leve como `commander`

Validação futura:

- `zod`

## Scripts esperados

No `viral-content-factory-backend/package.json`:

```json
{
  "scripts": {
    "dev": "bun run src/api/server.ts",
    "generate": "bun run src/cli/generate.ts",
    "typecheck": "tsc --noEmit"
  }
}
```

## Env vars iniciais

Arquivo: `viral-content-factory-backend/.env.example`

```env
NODE_ENV=development
API_HOST=localhost
API_PORT=3333
STORAGE_ROOT=./output
TEMP_ROOT=./temp
FFMPEG_PATH=ffmpeg
DEFAULT_VIDEO_WIDTH=1080
DEFAULT_VIDEO_HEIGHT=1920
DEFAULT_VIDEO_FPS=30
```

## Arquivos principais a criar

```text
src/api/server.ts
src/shared/config/env.ts
src/shared/errors/app-error.ts
src/shared/utils/file-system.ts
```

## Responsabilidades

### `src/api/server.ts`

- Inicializar Elysia.
- Configurar CORS.
- Registrar rotas futuras.
- Expor endpoint básico `GET /health`.

### `src/shared/config/env.ts`

- Ler variáveis de ambiente.
- Aplicar defaults seguros.
- Exportar objeto `env` tipado.

### `src/shared/errors/app-error.ts`

- Criar erro base da aplicação.
- Incluir `code`, `message`, `statusCode` opcional.

### `src/shared/utils/file-system.ts`

- Helpers para criar diretórios.
- Helpers para salvar JSON formatado.
- Helpers para validar existência de arquivos.

## Critérios de aceite

- `bun run dev` sobe API local.
- `GET /health` responde `{ "ok": true }`.
- `bun run typecheck` passa.
- Nenhum provider real foi acoplado.
