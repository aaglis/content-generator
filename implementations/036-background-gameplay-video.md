# 036 — Vídeo de fundo (gameplay) + card de chat centralizado

## Objetivo

Formato TikTok "fake texts over gameplay": clipe de fundo (Minecraft parkour, Subway, etc.)
em loop, com o chat do WhatsApp como **card arredondado centralizado** por cima. Mantém
narração + música + SFX (camadas inalteradas).

## Backend

- `RenderOptions.backgroundId` + `backgroundDim` (0..0.8, default 0.35) — por job.
  `resolveRenderOptions` valida/clampa. `env.backgroundsDir` (`assets/backgrounds`).
- Manifest `assets/backgrounds/manifest.json` (id/label/file) + `.gitignore` (clipes não
  versionados) + `scripts/setup-backgrounds.sh` (gera placeholders sintéticos via lavfi).
- `pipeline.resolveBackgroundPath(id)` → arquivo do manifest (se existir). `cardModeFor`
  (largura 84%, altura máx 80%, raio 3%). Quando há fundo, `renderChatFrames` gera **cards**
  (não frames 9:16). `RenderPlan.backgroundVideoPath`/`backgroundDim` (via builder). Vale nos
  3 caminhos (generate + renderEditedSfx/Script).
- `SharpChatRenderer` card mode: recorta o print revelado, escala p/ `cardWidth`, corta
  pelo topo se passar de `maxHeight` (mostra as mais novas), cantos arredondados via máscara
  SVG `dest-in` → PNG **com alpha**.
- `FFmpegVideoRenderer.renderBgSegment`: por cena, `[bg]scale cover→crop→eq(brightness=-dim)→fps`
  + `[card]format=rgba` + `overlay=(W-w)/2:(H-h)/2`. Hook = `drawtext` sobre o fundo. Áudio
  igual ao caminho de imagem (voz/silêncio + `adelay`). Fallback = só o fundo escurecido.
  Concat/música/SFX inalterados.
- API `GET /backgrounds` (lista só os que têm arquivo). `renderOptionsSchema` ganha
  `backgroundId`/`backgroundDim`.

## Frontend

- `RenderOptions` + `generateVideoApi.listBackgrounds()`.
- `VideoOptionsPanel`: seção **Fundo (gameplay)** (fetch no mount; `Nenhum` + clipes via
  `ToggleGroup`; hint quando vazio). `DEFAULT_OPTIONS.backgroundId=''`. Resumo mostra "com fundo".

## Validação

- Backend `typecheck` + frontend `vite build` ok.
- Card: 907×611 com alpha (cantos arredondados) a partir do print de exemplo.
- Composite FFmpeg (minecraft + card): MP4 1080×1920, 90 frames/3s; frame extraído confirma
  o card do WhatsApp real centralizado sobre o gameplay escurecido. `GET /backgrounds` → 3 itens.

## Revisão (upload do usuário, substitui o preset)

A 1ª versão usava um **manifest de presets** (minecraft/subway…) + endpoint `GET /backgrounds`
+ `setup-backgrounds.sh`. Feedback: **o usuário deve escolher o próprio vídeo**. Pivot:

- Removidos: `assets/backgrounds/` (manifest/placeholders/.gitignore), `setup-backgrounds.sh`,
  `env.backgroundsDir`, rota `GET /backgrounds`, `RenderOptions.backgroundId`,
  `resolveBackgroundPath` (manifest), `listBackgrounds` (FE).
- Upload: `createJob` aceita `backgroundVideoData` (base64 `video/*`) → salva
  `background.<ext>` no projeto. `VideoGenerationInput.backgroundVideoPath`;
  `findProjectBackground(projectDir)` acha `background.*` (persiste p/ regenerate/editor).
- **Fundo contínuo**: cada cena faz `-ss (startTimeSeconds % bgDuration)` com `-stream_loop -1`
  (probe da duração 1×), então o clipe roda contínuo entre as cenas e dá loop quando mais curto
  que o vídeo — cortado pela duração total.
- UI: `VideoOptionsPanel` troca o seletor de preset por **upload de vídeo** (file picker +
  nome + remover); `HomePage` envia o base64.

## Upload via multipart (substitui o base64)

Clipes podem ser grandes → base64 no JSON estourava memória.
- `POST /uploads/background` recebe **multipart/form-data** (campo `file`); o Elysia parseia em
  `body.file` (File) e `Bun.write` grava em `output/uploads/<uuid>.<ext>`; retorna `{ uploadId }`.
  `videoExt()` deriva a extensão (nome→type→mp4); `id` sanitizado contra path traversal.
- `createJob` recebe `backgroundUploadId`, **move** (`rename`) o upload p/ `projectDir/background.<ext>`.
- FE: `generateVideoApi.uploadBackground(file)` (`FormData`, sem Content-Type manual →
  CORS-safelisted, sem preflight); `HomePage` sobe o clipe e passa o `backgroundUploadId`.
- `server.ts`: `maxRequestBodySize: 2 GB` (clipes 1080p passam dos 128 MB default do Bun).

### Bug corrigido (NetworkError ao gerar)
A 1ª versão lia `request.body` cru em stream; o Elysia consome/trava o body antes do handler →
**500 no meio do upload** → conexão resetada → browser mostrava "NetworkError". multipart resolve
(parse nativo do Elysia) e o limite de body cobre clipes grandes.

## Notas

- Fundo exige `RERENDER_CHAT` on (os cards saem do `renderChatFrames`).
- Sem push-in no modo fundo (o gameplay já dá movimento).
