# 021 — Esqueleto: entidade `Score` + Direction Layer (passthrough)

> Habilitador da fase v2 (ver `020-v2-direction-and-quality-master-plan.md`). **Sem mudança de comportamento de vídeo.** Pré-requisito de 022–028.

## Objetivo

Introduzir o artefato **`Score`** (a "partitura" do vídeo) e a **Direction Layer** entre Narrative e Storyboard, em modo **passthrough**: mapeia a narrativa atual para o formato do Score, salva `score.json`, e **não** altera storyboard/render/áudio. As engines reais (Emotion/Hook/Retention/Timing/Voice/Music/SFX) substituem os mapeamentos passthrough nas etapas seguintes.

## Escopo

### 1. Entidades novas (`core/entities`)
`emotion-score.ts`, `voice-direction.ts`, `beat.ts`, `emotion-arc.ts`, `music-timeline.ts`, `sfx-trigger.ts`, `timing-plan.ts`, `score.ts` — conforme §4 do plano 020.

### 2. Interface (`core/interfaces`)
`direction-layer.ts`: `DirectionLayer.conduct(narrative, ctx) → Score`, com `DirectionContext { language, voiceAssets }`.

### 3. Módulo (`modules/direction`)
`PassthroughDirectionLayer`:
- **beats** ← `narrative.scenes` (emoção = `scene.emotion`; `intensity` = `clamp(scene.tensionLevel,0,10)`; `confidence` = 0.5; `segments` = `scene.speechSegments` ou `[{text,emphasis:false}]`; `role` heurístico; `isReveal` = `scene.isPlotTwist`).
- **arc** ← pontos `t = index/(n-1)`, `climaxBeatId` = maior intensidade.
- **timing** ← duração = áudio real (`voiceAssets[sceneId].durationSeconds`), **sem pausas** (`pre/post/hold = 0`) → idêntico ao comportamento atual.
- **hookBeat** ← espelha `narrative.hook` (ainda **não** inserido em `beats[]`).
- **music/sfx** ← vazios.
- `meta = { generator: 'passthrough', version: '0.1.0' }`.

### 4. Wiring (`modules/pipeline`)
Após síntese de voz, antes do storyboard: `directionLayer.conduct(...)` + salvar `score.json`. Flag `DIRECTION_ENABLED` (default `true`). CLI lista `score.json` nas saídas.

## Não-escopo
Storyboard/renderer **não** consomem o Score ainda. Sem LLM novo. Sem pausas/música/SFX reais.

## Critérios de aceitação
- [ ] `bun run typecheck` limpo.
- [ ] `bun run generate` (texto WHATSAPP) emite `score.json` válido junto dos artefatos.
- [ ] Vídeo/JSONs existentes **inalterados** (mesma duração/cenas).
- [ ] `DIRECTION_ENABLED=false` pula o passo sem quebrar.

## Próximo passo
`022-hook-engine.md`.
