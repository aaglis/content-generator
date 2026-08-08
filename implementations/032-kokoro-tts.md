# 032 — TTS local via Kokoro-FastAPI (remove a API de voz externa)

> Decisão do usuário: remover a integração de voz com API externa (ElevenLabs) e usar uma API
> **local** — Kokoro-FastAPI (remsky/Kokoro-FastAPI) rodando em `http://localhost:8880` — para
> ter tudo local e mais controle. Alinha com o restante do pipeline já local (OCR, render, Piper).

## O que mudou

- **Removido:** `ElevenLabsTTSProvider` + envs `TTS_API_KEY`/`ELEVEN_MODEL`/`ELEVEN_VOICE_A`/`B`.
  Nenhuma chamada de TTS na nuvem.
- **Novo:** `KokoroTTSProvider` — fala via `POST {KOKORO_BASE_URL}/v1/audio/speech`
  (OpenAI-compat), `response_format=wav`, sem key. Uma voz Kokoro por interlocutor do chat.
- Factory `createTTSProvider`: `case 'kokoro'`. Default do código continua `piper` (zero-setup);
  o `.env` seleciona `kokoro`.

## Mapeamento de direção (paridade com o Piper)

- `direction.rate` → Kokoro `speed` (clamp 0.6–1.6). Sem direção: `EMOTION_SPEED` por emoção.
- `direction.pitchSemitones` → Kokoro **não tem pitch**; aplicado depois via rubberband
  (`applyPitchShiftSemitones`), igual ao Piper.
- Sem timestamps por caractere (o endpoint OpenAI não dá) → não há deep+slow por palavra como no
  ElevenLabs; a expressividade vem do texto dirigido + rate/pitch. (Futuro: `/dev/captioned_speech`
  do Kokoro dá word-timestamps — habilitaria ênfase por palavra e karaokê do 031.)
- Casting inalterado: `voice_a` (deepSide) → `KOKORO_VOICE_A` (`pm_alex`), senão `KOKORO_VOICE_B`
  (`pf_dora`). Vozes pt-BR do Kokoro: `pm_alex`/`pm_santa` (masc.), `pf_dora` (fem.).

## Config

```env
TTS_PROVIDER=kokoro
# KOKORO_BASE_URL=http://localhost:8880   # default
# KOKORO_VOICE_A=pm_alex  KOKORO_VOICE_B=pf_dora
# KOKORO_LANG=            # vazio = infere pela voz (p = pt-BR)
```

Subir o servidor — **compose na raiz do projeto** (`docker-compose.yml`):
```bash
docker compose up -d            # imagem CPU; logs: docker compose logs -f kokoro
# GPU: KOKORO_IMAGE=ghcr.io/remsky/kokoro-fastapi-gpu:latest + bloco deploy (ver o arquivo)
```
Serviço `kokoro` (container `vcf-kokoro`), porta `${KOKORO_PORT:-8880}:8880`, healthcheck na lista
de vozes, `restart: unless-stopped`. Postgres segue separado em `docker-db/` (opcional, só Prisma).
Se o backend for containerizado, `KOKORO_BASE_URL=http://kokoro:8880`.

## Robustez

- `postWithRetry`: backoff em 5xx e erros de conexão (container esquentando). Se off, erro claro
  `TTS_FAILED` com a dica de subir o container e conferir `KOKORO_BASE_URL`.

## Validação

- `typecheck` ok; sem refs órfãs a ElevenLabs no `src`. **Pendente:** smoke real com o container de
  pé (estava desligado na implementação) — `curl localhost:8880/v1/audio/voices` p/ confirmar vozes
  pt-BR e gerar 1 vídeo ponta-a-ponta com `TTS_PROVIDER=kokoro`.

## Pendências

- Confirmar nomes exatos das vozes pt-BR na sua build do Kokoro (variam por versão do modelo);
  ajustar `KOKORO_VOICE_A/B` se preciso.
- Avaliar `/dev/captioned_speech` p/ word-timestamps (ênfase por palavra + karaokê do 031).
- LLM (director/hook/sfx) ainda é externo (OpenCode Zen) — fora do escopo desta etapa.
