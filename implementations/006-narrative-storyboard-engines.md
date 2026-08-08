# 006 — Narrative Engine e Storyboard Engine

## Objetivo

Criar os módulos responsáveis por transformar fonte normalizada em narrativa estruturada e depois em storyboard.

## Localização

```text
viral-content-factory-backend/src/modules/narrative/
viral-content-factory-backend/src/modules/storyboard/
```

## Narrative Engine

Arquivo sugerido:

```text
src/modules/narrative/narrative-engine.ts
```

Responsabilidade:

- Receber `ContentSource`.
- Usar `LLMProvider` para gerar `Narrative`.
- Validar que a narrativa tem hook e cenas.
- Aplicar defaults se mocks retornarem campos opcionais ausentes.

Campos mínimos por narrativa:

- `hook`
- `summary`
- `scenes`
- `overallTone`
- `estimatedDurationSeconds`

Campos mínimos por cena:

- `text`
- `emotion`
- `tensionLevel`
- `voiceTone`
- `soundEffectSuggestions`
- `musicSuggestion`

## Storyboard Engine

Arquivo sugerido:

```text
src/modules/storyboard/storyboard-engine.ts
```

Responsabilidade:

- Receber `Narrative` + `VideoTemplateType`.
- Gerar `Storyboard` com cenas temporizadas.
- Criar captions curtas.
- Criar visual cues conforme template.
- Criar audio cues baseadas nas emoções.
- Criar voice cues por cena.

## Regras iniciais de storyboard

- Vídeo vertical: 1080x1920.
- Cada cena: 4 a 8 segundos inicialmente.
- Hook deve ser destacado na primeira cena.
- Captions devem ser curtas o suficiente para tela vertical.
- Template `WHATSAPP_CHAT` deve sugerir visual de conversa/chat.

## Critérios de aceite

- A engine narrativa não conhece FFmpeg.
- A storyboard engine não chama TTS nem renderiza vídeo.
- Outputs são serializáveis como JSON.
