# 018 — Voz real & arquitetura multi-voz

> **Status (parcial):** arquitetura multi-voz pronta — `castVoices` (`modules/voice`) atribui
> voz distinta por remetente (esquerda vs direita), `speaker` em `NarrativeScene`, `voiceId` em
> `SpeechSynthesisInput`/`VoiceAsset`, pipeline aplica o casting, e o `MockTTSProvider` varia o
> timbre (frequência) por `voiceId` — duas vozes audíveis sem key. **Pendente:** provider real
> (ex: ElevenLabs) e duração da cena a partir do áudio real.

## Objetivo

Trocar o áudio sintético (tom) por **voz real** e dar **uma voz por interlocutor** da conversa (ninguém conversa sozinho). Cada balão é narrado pela voz do seu remetente.

## Contexto

- Etapa 017 deixou o pipeline com áudio real, mas é só um tom (`MockTTSProvider`).
- Etapa 015 entrega mensagens com `sender`/`senderSide`; etapa 016 faz uma cena por mensagem.
- Falta: casting de voz por remetente + provider de TTS real.

## Escopo

### 1. Estender contratos do `core`

```ts
// src/core/types/inputs.ts — SpeechSynthesisInput
voiceId?: string;   // voz a usar (mapeada a partir do remetente)
// src/core/entities/voice-asset.ts — VoiceAsset
voiceId?: string;
```

### 2. Casting de voz por remetente

- Novo helper (modules/voice) que mapeia remetentes → vozes distintas de forma estável:
  - 2 interlocutores: voz A (esquerda) / voz B (direita).
  - N interlocutores: round-robin sobre um pool de vozes do provider.
- O mapa de casting pode ser sugerido pelo LLM (gênero/tom por nome) — atrás de `LLMProvider`, opcional.

### 3. `ElevenLabsTTSProvider` (ou equivalente) implementando `TTSProvider`

- Selecionado por `env.ttsProvider` (`mock` | `elevenlabs`). Key em `env.ttsApiKey`.
- Baixa o áudio, grava via `StorageProvider.writeBinary` (etapa 017), retorna `VoiceAsset` com duração real (medida via ffprobe).
- Fallback para `MockTTSProvider` quando sem key.

### 4. Pipeline & módulo voice

- Mover a síntese para um `VoiceEngine` em `src/modules/voice/` (hoje a chamada está inline no pipeline). O engine recebe a narrativa + casting e devolve `VoiceAsset[]`.
- `core` continua sem SDK; o SDK vive no provider.

### 5. Sincronização

- A duração real do áudio (não mais `ceil(len/15)`) passa a definir `durationSeconds` da cena (consumido por 016 para o tempo do zoom).

## Critérios de aceitação

- [ ] Vozes distintas por remetente no MP4 final.
- [ ] `env.ttsProvider=elevenlabs` usa voz real; `mock` mantém tom (017).
- [ ] Duração da cena = duração real do áudio.
- [ ] Sem key → fallback para mock, sem quebrar o pipeline.

## Dependências

- 017 (mux de áudio), 015 (remetentes), 016 (cena por mensagem).

## Próximo passo

Etapa 019 — música de fundo + SFX cômicos.
