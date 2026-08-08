# 025 — Music Engine (state machine de mood + stingers)

> Fase v2 (ver `020`). **Absorve a parte de música do `019`.** Depende de `023` (arco) / `024` (timing).

## Objetivo

Dirigir música de fundo automaticamente: um **bed por mood** ao longo do arco, com **ducking** sob a voz e **stingers** (riser/impacto) nas viradas. Primeiro consumidor do arco para áudio.

## Escopo implementado

### Engine (`modules/direction/music`)
- `MusicEngine` (`core/interfaces`): `compose(beats, timing) → MusicTimeline`.
- `StateMachineMusicEngine`: agrupa beats consecutivos de mesmo mood num `MusicSegment` (sem trocar faixa a cada fala); dispara `MusicStinger` em reveal/plot twist (`impact_swell`) ou salto de intensidade ≥3 (`riser`), posicionado `start-0.3s` (cai na pré-pausa do reveal).
- `MOOD_BY_EMOTION`: neutral→ambient, suspense→tension, shock/anger/plot_twist/dramatic→dramatic, sadness→sad, funny→upbeat, romantic→romantic, embarrassment→ambient. `GAIN_BY_MOOD` (-16…-20 dB).

### Assets (placeholder, trocáveis)
- `assets/music/manifest.json`: mood→arquivo+gain e stingers→arquivo+gain.
- `scripts/setup-music.sh`: gera beds sintéticos (aevalsrc) + stingers (riser chirp, impact boom). **Trocar por faixas royalty-free reais** mantendo os nomes.

### Mix no renderer
- `RenderPlan.music` (do Score, via builder/pipeline).
- `mixMusic` (FFmpeg `-filter_complex`): por segmento `atrim+adelay+volume+afade`; `amix` dos segmentos → **`sidechaincompress`** com a voz como chave (ducking) → `amix(voz, bed-duckado, stingers)`. Roda entre o concat e o `mixSfx`.
- **Degradação graciosa**: `MUSIC_ENABLED=false`, manifest/arquivos ausentes ou qualquer falha → segue sem música (vídeo intacto).

## Validação
- `typecheck` exit 0.
- `score.music` segue o arco: `tension→ambient→dramatic→tension→dramatic`; stingers `riser@0/9.8/13.2`, **`impact_swell@21.5` no reveal** (reveal começa ~21.8).
- Bed preenche a timeline: silêncio total **~24s (024) → ~1.0s (025)** → `mixMusic` aplicado (não fallback). Loudness integrada ~-21 LUFS.

## Observações
- Beds são **placeholders sintéticos** (tons) — provam a mixagem (mood, ducking, stinger). Qualidade real = trocar os arquivos.
- `loudnorm`/limiter final e crossfade entre moods ficam como polimento.

## Próximo
`026-comic-sfx-engine.md` — boom/riser/record-scratch/FAAAH por beat, com prioridade/anti-poluição (consome `Score.sfx`).
