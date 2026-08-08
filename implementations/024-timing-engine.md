# 024 — Timing Engine (pausas dramáticas / hold / relógio único)

> Fase v2 (ver `020`). **Primeiro passo onde o Score dirige o render.** Depende de `021`/`023`.

## Objetivo

Converter roles + intensidade (Emotion Engine) em **pausas concretas** e fazer o vídeo realmente tê-las: silêncio dramático **antes do reveal**, **hold** no punchline/plot twist, **respiro** após cada fala. O `TimingPlan` vira o **relógio único** — storyboard e renderer passam a segui-lo.

## Algoritmo (`DramaticTimingEngine`, §9.2 do `020`)

```
preSilenceMs  = role==='reveal' ? round(600*intensity/10) : role==='cliffhanger' ? 400 : 0
postSilenceMs = clamp(120, 500, round(200 + 30*intensity))
holdMs        = (role==='punchline' || isPlotTwist) ? round(500*intensity/10) : 0
totalSeconds  = pre/1000 + speechDuration + post/1000 + hold/1000
startSeconds  = soma acumulada
```

## Escopo implementado

- `TimingEngine` (`core/interfaces`) + `DramaticTimingEngine` (`modules/direction/timing`).
- `PassthroughDirectionLayer` injeta o `TimingEngine`; `buildTiming` delega a ele (substitui os zeros do `021`).
- `StoryboardScene`/`RenderPlanScene` (+`preSilenceMs?`, `holdMs?`).
- **Storyboard consome o Score**: `StoryboardOptions.score`; ambos os paths (genérico + message-aware) usam `score.timing.totalSeconds` como duração da cena e carregam `preSilenceMs`/`holdMs` (fallback p/ duração do áudio quando não há score).
- **Renderer**: voz atrasada por `adelay=preSilenceMs` (a fala "cai" após o silêncio); o `hold`/`post` vêm de graça por a cena durar `totalSeconds > fala` (apad preenche).
- Pipeline passa o `score` ao storyboard.

## Validação
- `typecheck` exit 0.
- **Relógio único**: `score.timing.totalSeconds = 26.15` ≈ vídeo **26.22s** (antes do 024 o storyboard ignorava o timing → não casava). O vídeo cresce exatamente pelo orçamento de pausa.
- Reveal (plot_twist, intensity 10): `pre 600 + speech + post 500 + hold 500` no `score.json`; `render-plan` carrega as pausas por cena.
- `DIRECTION_ENABLED=false` → sem score → storyboard volta à duração só-fala (sem pausas), sem quebrar.

## Observações
- Mock TTS = tom senoidal fraco; medir silêncio por janela de tempo é enganoso (drift de frame no concat + nível do tom). A pausa é audível de verdade com voz real (Piper); a prova robusta é o **relógio único** (duração = soma falas + pausas) + `adelay` na voz.
- `postSilenceMs` em toda fala dá ritmo; ajustável no modelo.

## Próximo
`025-music-engine.md` — bed emocional + stingers (primeiro consumidor do arco para áudio).
