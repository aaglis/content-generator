# 013 — Validação, Checklist e Próximos Passos

## Objetivo

Definir como validar o MVP técnico e quais passos vêm depois.

## Checklist backend

Executar dentro de `viral-content-factory-backend/`:

```bash
bun install
bun run typecheck
bun run dev
bun run generate -- --input examples/whatsapp.txt --source WHATSAPP --template WHATSAPP_CHAT --output output/video.mp4
```

Validar:

- API responde `/health`.
- CLI gera JSONs intermediários.
- CLI gera MP4 válido.
- Erro claro quando FFmpeg não existe.
- `.env` não é versionado.

## Checklist frontend

Executar dentro de `viral-content-factory-frontend/`:

```bash
bun install
bun run typecheck
bun run dev
```

Validar:

- Tela abre.
- Campo de texto funciona.
- Source/template funcionam.
- Chamada para API funciona.
- Estado de erro aparece se API estiver offline.

## Checklist integração

1. Subir backend.
2. Subir frontend.
3. Colar exemplo WhatsApp.
4. Gerar job.
5. Confirmar que arquivos foram criados no backend.
6. Baixar MP4 pelo frontend.

## Próximos passos depois do MVP

### Melhorar renderização

- Template visual real de WhatsApp chat.
- Animação de mensagens aparecendo.
- Captions com safe area.
- Zooms/transições.
- Música e SFX reais.

### Providers reais

- Implementar `OpenAILLMProvider` ou outro provider real.
- Implementar `ElevenLabsTTSProvider` ou provider local.
- Manter mocks para testes.

### Persistência

- Adicionar Prisma/PostgreSQL quando precisar histórico real.
- Modelar `RenderJob`, `GeneratedAsset`, `VideoOutput`.

### Filas

- Adicionar BullMQ/Redis quando renderização ficar demorada.
- API cria job.
- Worker processa job.
- Frontend consulta status.

### Novas fontes

- Reddit stories.
- Twitter/X threads.
- Instagram DM.
- TikTok/Instagram comments.
- Notícias.
- User submitted stories.

### Serviço Python futuro

Só criar se houver necessidade concreta de:

- Whisper;
- OpenCV;
- MoviePy;
- modelos locais;
- GPU;
- processamento pesado isolado.

## Critério para considerar Fase 1 concluída

A Fase 1 termina quando o projeto gera um vídeo `.mp4` local a partir de `examples/whatsapp.txt`, mesmo que visualmente simples, com JSONs intermediários salvos e fluxo acionável por CLI e API.
