# 010 — API Elysia para Jobs Locais

## Objetivo

Criar API HTTP local para o frontend operar o pipeline.

## Localização

```text
viral-content-factory-backend/src/api/
```

## Endpoints mínimos

### `GET /health`

Resposta:

```json
{ "ok": true }
```

### `POST /jobs`

Cria e executa uma geração local.

Body sugerido:

```json
{
  "rawText": "Ana: ...",
  "source": "WHATSAPP",
  "template": "WHATSAPP_CHAT"
}
```

Resposta inicial:

```json
{
  "jobId": "...",
  "status": "COMPLETED",
  "outputPath": "..."
}
```

Para o MVP, pode ser síncrono. Depois vira assíncrono/fila.

### `GET /jobs/:jobId`

Retorna status e metadados locais.

### `GET /jobs/:jobId/files`

Retorna links/caminhos para:

- `narrative.json`
- `storyboard.json`
- `render-plan.json`
- `video.mp4`

### `GET /jobs/:jobId/download`

Retorna o MP4.

## Estrutura sugerida

```text
src/api/server.ts
src/api/routes/health-routes.ts
src/api/routes/job-routes.ts
src/api/http/http-error-handler.ts
```

## Regras

- API chama pipeline, mas não contém regra de narrativa/renderização.
- Validar body com Zod ou validação simples.
- Não expor arquivos fora do diretório de output.
- Erros devem ser JSON claros.

## Critérios de aceite

- Frontend consegue chamar `POST /jobs`.
- API consegue retornar status e baixar MP4.
- Pipeline segue compartilhado com CLI.
