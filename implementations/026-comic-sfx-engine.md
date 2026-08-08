# 026 — Comic SFX Engine (seleção por contexto via LLM)

> Fase v2 (ver `020`). Absorve SFX do `019`. Usa **apenas a paleta de sons do usuário**. Reescrito p/ seleção contextual via LLM (a versão por regras virou fallback offline).

## Objetivo

Colocar efeitos sonoros que **combinam com o conteúdo** de cada mensagem (FAAAH na revelação, deboche → quack/xaropinho, absurdo → train_whistle…), no volume certo (abaixo da voz) e **sincronizados com a palavra** da piada.

## Decisão (feedback do usuário)
A regra fixa ("FAAAH em todo punchline") não combinava. Agora **o LLM avalia a paleta + o contexto** e escolhe o som que encaixa, com parcimônia. Só sons fornecidos pelo usuário.

## Implementado

### Paleta (`assets/sfx/manifest.json` → `sounds`)
Registro `nome → {file, desc, durationSeconds}`. A **descrição** é o que o LLM lê p/ escolher (editável; principalmente `cggasa`/`meme1`, desconhecidos). Sons: faaah, quack, mario_1up, xaropinho, fart, whip, train_whistle, sneeze, tatuagem, cggasa, meme1. `byEmotion` = fallback sem Score.

### Engines
- `ComicSfxEngine` (`core`): `trigger(beats, timing) → Promise<SfxTrigger[]>` (async).
- **`LLMComicSfxEngine`** (`providers/sfx`, default c/ key): manda paleta (nome+desc+duração) + conversa (texto/emoção/papel) ao LLM; recebe `{fx:[{i,sound}]}`; valida contra a paleta. Prompt pede parcimônia + casamento por sentido (resposta na lata→whip, absurdo→train_whistle, revelação→faaah, deu ruim→fart, vitória→mario_1up, deboche→quack/xaropinho) e cuidado com sons longos.
- `RuleComicSfxEngine` (`modules/direction/sfx`): fallback offline determinístico (meme em punchline/reveal, whip em onset≥8).
- `createComicSfxEngine()` (`providers/sfx`): LLM se key, senão regra. Injetado na DirectionLayer (constructor por `DirectionLayerDeps`).
- Placement compartilhado (`sfx-placement.ts`): `punchTime` (sync na palavra de ênfase) + `resolveConflicts` (colisão 400ms, teto 3/10s).

### Volume + duração
- `gainDb −8` (voz Piper ≈ −14.7 LUFS, sons ≈ −11.7 → ~5 dB sob a fala).
- Renderer limita cada SFX a **4 s** (`atrim`) p/ sons longos (sneeze 18s…) não dominarem.
- `resolveScoreSfx` resolve `trackId → sounds[nome].file`.

### Limpeza
- **Removidos placeholders sintéticos** `boom/riser/pop/heartbeat` (SFX) **e os stingers de música** `riser/impact_swell` (eram sons sintéticos fora da lista do usuário).

## Validação
- `typecheck` exit 0.
- LLM no print do "satanás" escolheu, por contexto: `xaropinho`→"Oferecendo pro satanás né"; `quack`→"tua comida é pior que despacho"; `train_whistle`→setup absurdo da história; `faaah`→"E ele morreu" (revelação). 4 efeitos, só sons do usuário, sincronizados na palavra.
- Render OK (vídeo + áudio).

## Pendências
- Descrever `cggasa`/`meme1` (desconhecidos) p/ o LLM usá-los bem.
- Beds de música seguem placeholder sintético (trocar por royalty-free).
