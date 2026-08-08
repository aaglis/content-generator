# 009 — CLI: Comando `generate`

## Objetivo

Criar o primeiro fluxo funcional de ponta a ponta via CLI.

## Comando alvo

```bash
bun run generate -- --input examples/whatsapp.txt --source WHATSAPP --template WHATSAPP_CHAT --output output/video.mp4
```

## Localização

```text
viral-content-factory-backend/src/cli/generate.ts
```

## Argumentos obrigatórios

- `--input`: caminho para arquivo `.txt`.
- `--source`: tipo da fonte. Inicialmente `WHATSAPP` funcional.
- `--template`: template de vídeo. Inicialmente `WHATSAPP_CHAT` funcional.
- `--output`: caminho final do MP4.

## Pipeline executado pelo CLI

1. Ler arquivo input.
2. Resolver source adapter.
3. Criar `ContentSource`.
4. Gerar `Narrative` com `NarrativeEngine` + `MockLLMProvider`.
5. Salvar `narrative.json`.
6. Gerar `Storyboard`.
7. Salvar `storyboard.json`.
8. Gerar voice assets mockados.
9. Gerar `RenderPlan`.
10. Salvar `render-plan.json`.
11. Renderizar MP4 com `FFmpegVideoRenderer`.
12. Exibir resumo no terminal.

## Saídas esperadas

Considerando `--output output/video.mp4`:

```text
output/narrative.json
output/storyboard.json
output/render-plan.json
output/video.mp4
```

## Exemplo inicial

Criar:

```text
viral-content-factory-backend/examples/whatsapp.txt
```

Conteúdo sugerido:

```text
Ana: Você não vai acreditar no que aconteceu ontem.
Bruno: O que foi?
Ana: Mandei mensagem no grupo errado.
Bruno: Não...
Ana: Sim. Era sobre a surpresa de aniversário da Júlia.
Bruno: A Júlia estava no grupo?
Ana: Ela respondeu com um emoji de olho.
```

## Critérios de aceite

- Comando gera os três JSONs e o MP4.
- Erros de argumento são claros.
- Source/template não suportados falham com mensagem controlada.
