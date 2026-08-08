# 019 — Música de fundo & efeitos cômicos

## Objetivo

Adicionar **música de fundo** (em alguns momentos, com ducking sob a narração) e **efeitos sonoros cômicos famosos** (ex: o "FAAAAH") disparados por emoção/keyword da conversa. É a camada de "viralização".

## Contexto

- A narrativa já produz `musicSuggestion` e `soundEffectSuggestions` por cena (`NarrativeScene`), e o storyboard já gera `AudioCue` (`type: 'music' | 'sfx' | 'silence'`) — hoje ignorados pelo renderer.
- Etapas 017/018 entregaram a trilha de voz. Falta misturar música + SFX por cima.

## Escopo

### 1. Biblioteca de assets local

```text
viral-content-factory-backend/assets/
  music/        # faixas de fundo (mp3/wav) por mood: upbeat, tension, dramatic...
  sfx/          # efeitos: faaah.mp3, gasp.mp3, dramatic_hit.mp3, laugh_track.mp3...
  manifest.json # mapeia mood/keyword -> arquivo + ganho default
```

- Versionado (assets são parte do produto). `manifest.json` é a fonte de verdade do mapeamento.

### 2. `AudioMixEngine` (modules/audio)

- Resolve cada `AudioCue` para um arquivo via manifest.
- Música: escolhe faixa pelo `overallTone`/`musicSuggestion`, com fade in/out e **ducking** (abaixa a música quando há voz — `sidechaincompress` no FFmpeg).
- SFX: posiciona no início da cena correspondente (ex: `dramatic_hit` em plot twist; `faaah` em `funny`/`shock`). O timing/colocação pode ser refinado pelo LLM.

### 3. Mix no FFmpeg

- Construir mixagem final com `-filter_complex` (`amix`/`sidechaincompress`/`adelay` para offset dos SFX) combinando: trilha de voz (017/018) + música + SFX.
- Aplicar como passo de áudio sobre o vídeo concatenado (ou por cena, mantendo params uniformes para o concat).

### 4. Controle

- Env/flags para ligar/desligar música e SFX. Volume relativo no manifest.

## Critérios de aceitação

- [ ] Música de fundo audível, com ducking sob a narração.
- [ ] SFX cômico dispara na cena certa pela emoção (ex: `faaah`).
- [ ] Mixagem não estoura clipping; voz continua inteligível.
- [ ] Assets ausentes degradam com aviso, sem quebrar o render.

## Dependências

- 017 (trilha de voz), 016 (cenas por mensagem para timing), 018 (voz real, ideal).

## Próximo passo

Polimento: templates visuais por categoria de fonte, legendas estilizadas, presets de "meme".
