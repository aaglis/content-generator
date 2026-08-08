# 033 — Camera Motion (push-in Ken Burns por intensidade)

## Problema

O path de render ativo (etapa 031, frames acumulados do print) compõe um PNG 9:16 e
o renderer apenas faz `scale` — **frame totalmente parado**. O path de crop (fallback)
usa só `endCrop` e o `CameraEngine` devolve `startCrop === endCrop` ("v1: static framing
… future polish", etapa 016). Resultado: vídeo estático, ritmo visual constante, espaço
morto. O `Score` já carrega `emotion.intensity` (0..10) por beat, mas isso nunca chegava
à câmera.

## Escopo

Adicionar um **push-in Ken Burns por cena**, com zoom-alvo escalado pela intensidade
emocional do beat (e um empurrão extra em `reveal`/`punchline`/`cliffhanger`), aplicado
em **todos os paths** do renderer. Sem refactor grande, compatível com o pipeline atual,
ligável/desligável por env.

## Mudanças

- `core/entities/storyboard-scene.ts`: `pushInZoom?` (zoom-alvo, 1.0 = nada) + `focusY?`
  (0 topo … 1 base; chat usa ~0.62 para focar o balão mais novo).
- `renderer/render-plan.ts`: idem em `RenderPlanScene`.
- `renderer/render-plan-builder.ts`: copia `pushInZoom`/`focusY` da cena.
- `storyboard/storyboard-engine.ts`: `buildBeatHintMap` (beatId→{intensity,role}) +
  `computePushIn` (`1.06 + i*0.12`, `+0.04` em reveal/punchline/cliffhanger, clamp
  1.04–1.26; hook 1.05 gentil). Setado nos dois geradores (genérico e message-aware) e
  no card de hook.
- `renderer/ffmpeg-video-renderer.ts`: `buildPushInFilter` (zoompan `1.0→z1` com `on-1`,
  `x` centrado, `y` por `focusY`). Aplicado no path do frame, no path de crop e no path
  genérico (substitui o Ken Burns alternado fraco quando há `pushInZoom`). Hook (texto
  sobre cor sólida) fica sem push-in de propósito.
- `shared/config/env.ts`: `CAMERA_MOTION` (default on) + `CAMERA_MOTION_SCALE` (default 1;
  0 desliga, >1 mais agressivo).

## Validação

- `bun run typecheck` → exit 0.
- Smoke FFmpeg do filtro isolado (`zoompan` 1.0→1.18, focusY 0.62, 2s@30fps, 1080×1920):
  encoda em libopenh264, 60 frames, 2.0s. Sintaxe válida, movimento presente.
- Pendente: smoke do pipeline ponta a ponta com TTS local (Kokoro/Piper) de pé.

## Impacto

Mata o problema nº1 (vídeo estático) e suaviza nº2 (espaço vazio) e nº10 (ritmo constante)
sem assets pagos. Punchline/reveal ganham zoom mais forte (parte do nº7). Reversível por flag.
