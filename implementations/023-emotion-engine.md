# 023 — Emotion Engine (intensidade + confiança + arco)

> Fase v2 (ver `020`). Promove o `DialogueDirector`. Fundação de Timing/Music/SFX/Voice/Retention. Depende de `021`.

## Objetivo

Substituir o `intensity = tensionLevel`/`confidence = 0.5` ingênuo (passthrough do `021`) por um **modelo híbrido real** de `{emotion, intensity(0..10), confidence(0..1)}` por beat, e alimentar o **arco emocional** do `Score`.

## Escopo implementado

### Promoção do director
- `DirectedLine` (+`intensity?: low|medium|high`, +`confidence?: 0..1`). `IntensityBucket` em `core/types/emotion.ts`.
- `OpenAICompatDirector`: prompt pede `intensity` + `confidence` por fala; parse valida bucket + clampa confiança.
- `NarrativeScene` (+`emotionIntensityBucket`, +`emotionConfidence`); pipeline salva ambos no passo do director.

### Engine (`modules/direction/emotion`)
- `EmotionEngine` (`core/interfaces`): `scoreBeat({text,emotion,role,t,isPlotTwist,isHook,bucket?,llmConfidence?}) → EmotionScore`.
- `HybridEmotionEngine`: determinístico, com ou sem sinais do LLM.
- `intensity-model.ts` (fórmula §6.3 do `020`):
  ```
  intensity = clamp(0,10, EMOTION_BASE[emotion] + bucketBoost + punctuationBoost + lexiconBoost + roleBoost + arcPositionBoost)
  confidence = clamp(0,1, 0.5*(llmConfidence ?? 0.6) + 0.5*heuristicAgreement)
  ```
  `EMOTION_BASE`: neutral 2 … plot_twist/shock 8. `arcPositionBoost`: pico triangular em t≈0.85. `roleBoost`: reveal/punchline +1.5, hook/cliffhanger +1.
- `emotion-lexicon.pt.ts`: léxico viral pt-BR (`traição`, `mentiu`, `descobri`, `kkk`, `morreu`, `surpresa`…) → emoção candidata + peso. Usado p/ boost **e** p/ calibrar confiança (concordância com o label do director).

### Integração
- `PassthroughDirectionLayer` injeta `EmotionEngine` (default `HybridEmotionEngine`); `toBeat` calcula `role`+`t` e chama `scoreBeat`. `meta.generator='direction-layer'`, `version='0.2.0'`.

## Validação
- `typecheck` exit 0.
- **LLM on** (`whatsapp.txt`): intensidades variadas (2.8→10), confiança variada (0.43→0.73); `arc.climaxBeatId` = beat `reveal` (plot_twist). Calibração: linha "surpresa" rotulada `funny` cai p/ conf **0.43** (léxico→suspense discorda).
- **Offline** (`LLM_PROVIDER=mock`): determinístico, todos beats com score; conf **0.55** = `0.5·0.6 + 0.5·0.5` (sem LLM, sem léxico). Reproduzível.

## Observações
- Efeito ainda **não visível no vídeo** (esperado): intensidade/arco passam a alimentar Timing (`024`), Music (`025`), SFX (`026`), Voice (`027`), Retention (`028`).
- Limitação herdada: `PassthroughDirector` (sem LLM) achata `emotion→neutral`; o label de emoção continua sendo do LLM director.

## Próximo
`024-timing-engine.md` — pausas dramáticas / hold / relógio único (primeiro consumidor visível do arco).
