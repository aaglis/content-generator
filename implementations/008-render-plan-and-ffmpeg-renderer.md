# 008 — Render Plan e FFmpeg Video Renderer

## Objetivo

Criar uma etapa de plano de renderização e um renderer inicial usando FFmpeg para gerar MP4 vertical simples.

## Localização

```text
viral-content-factory-backend/src/modules/renderer/
```

## Arquivos a criar

```text
src/modules/renderer/render-plan.ts
src/modules/renderer/render-plan-builder.ts
src/modules/renderer/ffmpeg-video-renderer.ts
```

## RenderPlan

Deve ser uma estrutura JSON serializável contendo:

- `jobId`
- `width`
- `height`
- `fps`
- `durationSeconds`
- `template`
- `scenes`
- `outputPath`

Cada cena do render plan deve conter:

- `startTimeSeconds`
- `durationSeconds`
- `caption`
- `background`
- `visualStyle`
- `transition`
- `voiceAssetPath?`
- `audioCues`
- `visualCues`

## RenderPlanBuilder

Responsabilidade:

- Receber `Storyboard` e assets de voz.
- Criar plano final para renderização.
- Não executar FFmpeg.

## FFmpegVideoRenderer inicial

Responsabilidade:

- Implementar `VideoRenderer`.
- Receber render plan.
- Gerar MP4 9:16 simples.

Renderização mínima aceitável:

- fundo sólido ou gradiente simples;
- texto/caption centralizado;
- cenas sequenciais;
- resolução 1080x1920;
- 30fps;
- output `.mp4`.

## Importante

O primeiro renderer não precisa parecer produto final. Ele precisa provar o pipeline.

Melhor começar simples e evoluir depois:

1. fundo sólido + captions;
2. simular WhatsApp chat visual;
3. adicionar áudio TTS real;
4. adicionar música/SFX;
5. adicionar zoom/transições.

## Critérios de aceite

- `render-plan.json` é salvo antes do MP4.
- FFmpeg gera arquivo `.mp4` válido.
- Se FFmpeg não estiver instalado, erro é claro.
