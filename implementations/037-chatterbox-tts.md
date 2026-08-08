# 037 — Chatterbox TTS (motor local com emoção + clonagem de voz)

## Status: implemented

## Contexto / motivação

O Kokoro ficou bem abaixo do ElevenLabs em emoção e naturalidade (feedback do usuário).
Testada a demo web do **Chatterbox Multilingual** (ResembleAI) em pt-BR — qualidade muito boa.
Decisão: adotar o Chatterbox como **motor local padrão**, mantendo Kokoro e ElevenLabs
selecionáveis por vídeo.

Diferença de voz vs ElevenLabs: o Chatterbox **não tem biblioteca de vozes nomeadas** (Adam,
Rachel…). É **zero-shot voice cloning** — a voz é definida por um arquivo de áudio de referência.
Na prática vira biblioteca de presets: cada `.wav`/`.mp3` na pasta `./voices/` do container é uma
voz selecionável (dropa `adam.wav` → vira a voz "adam"). Clonagem de pessoa específica = mesmo
fluxo via `./reference_audio/`.

Servidor escolhido: **devnen/Chatterbox-TTS-Server** — embrulha o mesmo modelo da demo num
servidor com Web UI + API (OpenAI-compat **e** `/tts` rico), docker compose (CPU/GPU), listagem
de vozes e upload. Encaixa quase 1:1 no padrão já usado pelo `KokoroTTSProvider`.

## Decisões

- **Hardware:** CPU (escolha do usuário). Compose builda a imagem CPU do repo oficial.
- **Papel:** novo default local (`TTS_PROVIDER=chatterbox`); Kokoro/Eleven continuam no toggle.
- **Endpoint:** `POST /tts` (rico) em vez do `/v1/audio/speech` — dá acesso ao `exaggeration`
  (emoção, o trunfo sobre o Kokoro), `language`, `speed_factor`.
- **Emoção:** mapa `emoção → exaggeration` (shock 1.4, anger 1.5, sad 0.45, suspense 0.6…),
  fallback `CHATTERBOX_EXAGGERATION` (0.5 neutro). `cfg_weight`/`temperature` só enviados se
  setados no env (senão usa o default do servidor — evita conflito de escala).
- **Paridade com Kokoro/Piper:** `direction.rate → speed_factor`; `pitchSemitones` via rubberband
  pós (Chatterbox não tem pitch); ênfase deep+slow nos spans marcados; retry/backoff; erro claro
  se o container estiver off.
- **Voz por interlocutor:** filename concreto escolhido na UI (per-side) vence; slot abstrato
  `voice_a..d` → env override, senão resolve pela lista do container (`/get_predefined_voices`),
  espelhando o padrão do `ElevenLabsTTSProvider`.

## Mudanças

### Backend
- `providers/tts/chatterbox-tts-provider.ts` — novo `ChatterboxTTSProvider`.
- `providers/tts/index.ts` — `case 'chatterbox'` + export; default da factory → chatterbox;
  fallback do `eleven` agora cai no Chatterbox (default local).
- `shared/config/env.ts` — `TTS_PROVIDER` default `chatterbox`; novas envs
  `CHATTERBOX_BASE_URL` (`http://localhost:8004`), `CHATTERBOX_LANG` (`pt`),
  `CHATTERBOX_VOICE_A..D`, `CHATTERBOX_EXAGGERATION`, `CHATTERBOX_CFG`, `CHATTERBOX_TEMPERATURE`.
- `core/types/render-options.ts` — `ttsProvider: 'chatterbox' | 'kokoro' | 'eleven'`.
- `shared/config/render-options.ts` — aceita `chatterbox` no merge.
- `modules/pipeline/video-generation-pipeline.ts` — `resolveVoiceId` aplica per-side também p/ chatterbox.
- `api/routes/job-routes.ts` — enum `ttsProvider` + chatterbox; rota
  `GET /tts/chatterbox/voices` (proxy de `/get_predefined_voices`, devolve `{id,label}`).
- `.env` / `.env.example` — default + bloco Chatterbox documentado.

### Infra
- `docker-compose.yml` (raiz) — serviço `chatterbox` (build CPU do git oficial `Dockerfile.cpu`,
  porta 8004, volumes `./chatterbox/{voices,reference_audio,outputs}` + cache HF, healthcheck).
- `chatterbox/voices/README.md` — instruções de como adicionar vozes/presets.

### Frontend
- `core/types/generate-video.ts` — union `ttsProvider` com chatterbox.
- `services/api/generate-video-api.ts` — `listChatterboxVoices()`.
- `components/VideoOptionsPanel.tsx` — toggle ganha **Chatterbox** (default); carrega a lista de
  vozes; `ChatterboxVoiceSelect` por lado (esq/dir); hint avisa quando não há vozes na pasta.

## Como usar

```bash
# 1) sobe o container (1ª vez builda no CPU — demora; depois fica em cache)
docker compose up -d chatterbox
docker compose logs -f chatterbox          # acompanha build + download do modelo

# 2) (opcional) adiciona vozes: dropa .wav/.mp3 (~10s limpos) em ./chatterbox/voices/
#    ex.: adam.wav → aparece como voz "adam" no frontend

# 3) gera normalmente; no painel "Opções do vídeo" o motor já vem em Chatterbox.
```

Backend no host fala com o container em `http://localhost:8004`. Se containerizar o backend,
trocar `CHATTERBOX_BASE_URL=http://chatterbox:8004`.

## Validação

- `tsc --noEmit` backend e frontend: exit 0.
- `vite build`: 2283 módulos, ok.
- `docker compose config`: válido.
- Smoke do provider contra stub HTTP: request `/tts` correto (`voice_mode=predefined`,
  `predefined_voice_id`, `language=pt`, `speed_factor`+`exaggeration` por emoção, normalize
  `pq vc`→`porque você`); slot abstrato `voice_b` resolvido via `/get_predefined_voices`;
  wav escrito + duração via ffprobe.

## Pendente

- Smoke real com o container de pé (1º build CPU é lento; download do modelo).
- Confirmar o code de idioma pt-BR esperado pelo modelo (`pt` assumido).
- UI opcional: slider de `exaggeration` por job e fluxo de **upload de voz/clonagem** pelo
  frontend (hoje as vozes entram pela pasta `./chatterbox/voices`).
- GPU: trocar o build CPU por CUDA + `deploy:` quando houver GPU (compose já comenta o bloco).
