# 017 — Fundação de Áudio (mux de voz no vídeo)

## Objetivo

Dar **áudio real** ao vídeo. Até aqui o MP4 saía mudo: o `MockTTSProvider` retornava um path `.mock.txt` falso e o `FFmpegVideoRenderer` ignorava `voiceAssetPath`. Esta etapa faz o mock gerar áudio real e o renderer multiplexar uma trilha de áudio por cena. É pré-requisito do zoom sincronizado (016) e da voz real (018).

## Contexto

- `MockTTSProvider` (`src/providers/tts/mock-tts-provider.ts`) não escrevia arquivo nenhum.
- `FFmpegVideoRenderer` montava segmentos só de vídeo e concatenava com `-c copy` — sem áudio.
- `RenderPlanScene.voiceAssetPath` já existia e era injetado pelo `RenderPlanBuilder`, mas descartado no render.

## Escopo (implementado)

### 1. `StorageProvider.writeBinary`

- Interface `src/core/interfaces/storage-provider.ts`: `writeBinary(path, data: Uint8Array): Promise<void>`.
- Impl `LocalStorageProvider`: `writeFile(path, data)` após `ensureFileDir`. (Habilita gravar áudio/binário no futuro; o mock atual escreve via FFmpeg direto em disco.)

### 2. `runCommand` compartilhado

- Extraído para `src/shared/utils/run-command.ts` (antes duplicado no renderer). Lança `Error` com a cauda do `stderr` no exit ≠ 0. Reusado por renderer e mock TTS.

### 3. `MockTTSProvider` gera WAV real

- Sintetiza `.wav` real via FFmpeg `lavfi`, dimensionado pela duração `Math.max(1, ceil(text.length / 15))`s, em `${tempRoot}/tts/${sceneId}.wav`.
- Modo por env `TTS_MOCK_MODE`: `tone` (default, `sine=220Hz` a volume 0.08) ou `silence` (`anullsrc`).
- Retorna `VoiceAsset` com `audioPath` real e `durationSeconds` exato.
- Provider pode usar FFmpeg (não é `core`); path vem de `env.ffmpegPath`.

### 4. Mux de áudio no `FFmpegVideoRenderer`

- Por segmento: se `voiceAssetPath` existe → `-i <wav>`; senão `-f lavfi -i anullsrc=r=44100:cl=stereo`.
- Streams mapeados (`-map 0:v:0 -map 1:a:0`), áudio uniforme `-c:a aac -ar 44100 -ac 2 -af apad`, `-t duration` para casar com o vídeo. Todos os segmentos ficam com params idênticos → concat demuxer `-c copy` preserva o áudio.
- Bug corrigido: `render()` não mascara mais erro de FFmpeg como `RENDER_PLAN_NOT_FOUND`; só falha de leitura do plano vira esse erro, o resto propaga a mensagem real.

### 5. Env

- `src/shared/config/env.ts`: novo `ttsMockMode` (`TTS_MOCK_MODE`, default `tone`).

## Critérios de aceitação

- [x] `bun run generate` produz MP4 com stream de áudio (`ffprobe` mostra `codec_type=audio`).
- [x] Duração do áudio == duração do vídeo == soma das cenas.
- [x] WAVs reais escritos em `temp/tts/`.
- [x] `bun run typecheck` limpo.

## Verificação executada

```bash
bun run generate -- --input examples/whatsapp.txt --source WHATSAPP --template WHATSAPP_CHAT --output output/cli-test/video.mp4
ffprobe -v error -show_entries stream=codec_type,codec_name output/cli-test/video.mp4   # video(mpeg4) + audio(aac)
# format=41.02s, audio=41.02s, em sync
```

## Não-objetivos

OCR (015), zoom por balão (016), voz real/multi-voz (018), música e SFX (019).

## Próximo passo

Etapa 015 (OCR híbrido) — ler os balões do print de verdade.
