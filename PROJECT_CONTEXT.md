# Viral Content Factory — Contexto Completo do Projeto

## 1. Ideia do projeto

O **Viral Content Factory** é uma ferramenta interna/local para gerar vídeos verticais automatizados para TikTok, Instagram Reels e YouTube Shorts a partir de textos narrativos ou conversas.

O primeiro caso suportado é **WhatsApp Story**: o usuário fornece uma conversa em texto ou imagem/print, e o sistema transforma esse conteúdo em um vídeo vertical com narrativa, storyboard, voz, efeitos e renderização em MP4.

O projeto foi desenhado para começar simples no MVP, mas com arquitetura extensível para novas fontes e providers.

Fontes futuras previstas:

- Instagram DM;
- Twitter/X Threads;
- Reddit Stories;
- comentários de TikTok/Instagram;
- notícias;
- histórias enviadas por usuários.

Fluxo conceitual:

```text
Content Source
↓
Narrative Engine
↓
Storyboard Engine
↓
Voice Engine
↓
Audio/Music Engine
↓
Video Renderer
↓
Output MP4
```

---

## 2. Organização geral

Este projeto **não usa monorepo workspace** neste momento. Backend e frontend ficam em diretórios independentes dentro da raiz.

Estrutura principal:

```text
content-generator/
  CLAUDE.md
  PROJECT_CONTEXT.md
  docker-db/
  implementations/
  skills-lock.json
  viral-content-factory-backend/
  viral-content-factory-frontend/
```

Diretórios principais:

- `implementations/`: planos numerados de implementação e status do projeto;
- `viral-content-factory-backend/`: backend Bun + TypeScript + Elysia;
- `viral-content-factory-frontend/`: frontend React + Vite + TypeScript + Tailwind;
- `docker-db/`: suporte local para banco/Postgres;
- `CLAUDE.md`: contexto operacional original do projeto;
- `PROJECT_CONTEXT.md`: este documento consolidado.

Cada serviço possui seus próprios arquivos independentes:

- `.env.example` versionado;
- `.env` local, não versionado;
- `.gitignore`;
- `package.json`;
- `tsconfig.json`;
- `README.md`.

---

## 3. Status das implementações

O status principal está em:

```text
implementations/README.md
```

Planos registrados:

| Etapa | Arquivo | Status | Descrição |
|---|---|---|---|
| 000 | `000-master-plan.md` | `planned` | Plano mestre e direção arquitetural |
| 001 | `001-repository-structure.md` | `implemented` | Backend e frontend separados |
| 002 | `002-backend-foundation.md` | `implemented` | Fundação Bun + TS + Elysia |
| 003 | `003-core-domain-types.md` | `implemented` | Tipos e entidades do domínio |
| 004 | `004-core-interfaces.md` | `implemented` | Interfaces abstratas do core |
| 005 | `005-source-adapters.md` | `implemented` | Source adapters, WhatsApp funcional |
| 006 | `006-narrative-storyboard-engines.md` | `implemented` | Engines de narrativa e storyboard |
| 007 | `007-mock-providers.md` | `implemented` | MockLLM, MockTTS e LocalStorage |
| 008 | `008-render-plan-and-ffmpeg-renderer.md` | `implemented` | Render plan + FFmpeg renderer |
| 009 | `009-cli-generate-command.md` | `implemented` | CLI `generate` |
| 010 | `010-backend-api-elysia.md` | `implemented` | API Elysia para jobs locais |
| 011 | `011-frontend-foundation.md` | `implemented` | Fundação React + Vite + Tailwind |
| 012 | `012-frontend-generation-ui.md` | `implemented` | Interface de geração |
| 013 | `013-validation-and-next-steps.md` | `implemented` | Validação do MVP / fase 1 completa |
| 014 | `014-prisma-postgres-persistence.md` | `implemented` | Persistência Prisma/Postgres |
| 015 | `015-ocr-image-parsing.md` | `partial` | OCR via Tesseract + heurísticas; organizador via LLM ainda futuro |
| 016 | `016-message-aware-storyboard.md` | `implemented` | Storyboard ciente de mensagens + zoom por bolha |
| 017 | `017-audio-foundation.md` | `implemented` | TTS WAV real + mux AAC |
| 018 | `018-multi-voice-tts.md` | `partial` | Multi-voz neural local feita; vozes cloud ainda pendentes |
| 019 | `019-music-and-comic-sfx.md` | `planned` | Música de fundo + efeitos cômicos |

---

## 4. Backend

Diretório:

```text
viral-content-factory-backend/
```

Tecnologias:

- Bun.js;
- TypeScript;
- Elysia;
- FFmpeg para renderização;
- Prisma + PostgreSQL para persistência;
- Tesseract para OCR;
- providers abstratos para LLM, TTS, OCR, storage e renderização.

Scripts principais:

- `bun run dev`: sobe a API;
- `bun run generate`: executa a geração via CLI;
- `bun run typecheck`: valida TypeScript;
- `bun run db:generate`, `db:migrate`, `db:push`, `db:studio`: comandos Prisma.

### 4.1 Estrutura do backend

```text
viral-content-factory-backend/
  assets/
    sfx/
    tts-pronunciation.json
  examples/
    whatsapp.txt
    whatsapp-print.jpg
  scripts/
    setup-piper.sh
    setup-sfx.sh
  src/
    api/
    cli/
    core/
    modules/
    providers/
    shared/
  .env.example
  .gitignore
  package.json
  README.md
  tsconfig.json
```

### 4.2 Arquitetura backend

O backend segue princípios de **Clean Architecture**.

Direção desejada de dependência:

```text
api / cli / providers
↓
modules / use cases / pipeline
↓
core / entities / interfaces / types
```

Regras arquiteturais:

- `core` não deve depender de Elysia, Bun APIs específicas, FFmpeg, Prisma ou SDKs externos;
- `core` contém entidades, tipos e interfaces;
- `modules` contém lógica de aplicação;
- `providers` implementam integrações externas;
- `api` apenas expõe HTTP e chama casos de uso/pipeline;
- `cli` apenas parseia argumentos e chama o mesmo pipeline da API;
- regra de negócio não deve ficar dentro de controllers Elysia.

### 4.3 Core

Diretório:

```text
src/core/
  entities/
  interfaces/
  types/
```

Responsabilidades:

- entidades do domínio;
- tipos compartilhados;
- contratos abstratos como `LLMProvider`, `TTSProvider`, `StorageProvider`, `VideoRenderer`, `SourceAdapter`.

### 4.4 Módulos

Diretório:

```text
src/modules/
```

Módulos principais:

- `sources/`: adapta entrada bruta para conteúdo estruturado;
- `narrative/`: transforma conteúdo fonte em narrativa;
- `storyboard/`: transforma narrativa em cenas, cues e estrutura visual;
- `voice/`: resolve vozes por personagem/remetente;
- `audio/`: resolve efeitos e eventos sonoros;
- `camera/`: calcula crop/zoom para focar mensagens;
- `renderer/`: cria render plan e renderiza vídeo;
- `pipeline/`: orquestra todo o fluxo.

#### Sources

Implementações atuais:

- `WhatsAppSourceAdapter`: parseia conversas em texto no formato `remetente: mensagem`;
- `WhatsAppImageSourceAdapter`: usa OCR para extrair mensagens de print de WhatsApp;
- `SourceAdapterRegistry`: seleciona o adapter pelo tipo da fonte.

Placeholders futuros:

- Reddit;
- Twitter/X;
- Instagram DM.

#### Narrative Engine

Responsável por gerar narrativa a partir do conteúdo estruturado.

Comportamento atual:

- para texto, usa `MockLLMProvider`;
- para OCR/chat, pode gerar uma cena por mensagem;
- mantém a lógica fora da API e fora do frontend.

#### Storyboard Engine

Responsável por criar cenas, cues visuais e cues de áudio.

Recursos atuais:

- storyboard genérico;
- storyboard ciente de mensagens;
- zoom/foco em bolhas específicas;
- pistas de câmera e áudio por cena.

#### Voice Engine

Responsável por mapear remetentes/personagens para vozes.

Recursos atuais:

- casting determinístico por speaker;
- suporte a multi-voice local;
- base para providers cloud no futuro.

#### Audio/Music Engine

Responsável por efeitos sonoros, cues e futura música de fundo.

Recursos atuais:

- manifesto local de SFX;
- eventos de SFX por emoção/cena;
- mixagem de SFX no renderer.

Pendente:

- engine completa de música de fundo;
- camada mais rica de comic SFX.

#### Renderer

Responsável por converter storyboard em render plan e renderizar MP4.

Componentes atuais:

- `render-plan.ts`: tipos do plano de renderização;
- `render-plan-builder.ts`: converte storyboard em render plan;
- `ffmpeg-video-renderer.ts`: renderiza vídeo vertical e mixa áudio/SFX com FFmpeg.

#### Pipeline

Arquivo principal:

```text
src/modules/pipeline/video-generation-pipeline.ts
```

Responsável por orquestrar:

1. leitura/adaptação da fonte;
2. geração de narrativa;
3. direção de diálogos;
4. seleção/síntese de voz;
5. geração de storyboard;
6. criação de render plan;
7. renderização final em MP4;
8. gravação dos artefatos intermediários.

---

## 5. Providers do backend

Diretório:

```text
src/providers/
```

Providers implementados:

### LLM

- `MockLLMProvider`: geração mockada de narrativa;
- estrutura preparada para providers reais.

### TTS

- `MockTTSProvider`: mock que gera áudio WAV simples;
- `EspeakTTSProvider`: TTS local via espeak;
- `PiperTTSProvider`: TTS neural local;
- `ElevenLabsTTSProvider`: provider cloud previsto/implementado de forma condicional;
- seleção por env via factory/index.

### OCR

- `TesseractOCRProvider`: extrai texto de imagens/prints e agrupa mensagens.

### Storage

- `LocalStorageProvider`: salva arquivos no filesystem local.

### Director

- `PassthroughDirector`: direção sem LLM, mantém conteúdo como está;
- `OpenAICompatDirector`: estrutura compatível com APIs estilo OpenAI para direção via LLM;
- seleção condicional por env.

---

## 6. CLI do backend

Entrada principal:

```text
src/cli/generate.ts
```

Comando alvo do MVP:

```bash
bun run generate -- --input examples/whatsapp.txt --source WHATSAPP --template WHATSAPP_CHAT --output output/video.mp4
```

Assinatura atual:

```bash
bun run generate -- \
  --input <path> \
  --source <type> \
  --template <type> \
  --output <path> \
  [--project <name>] \
  [--projects-root <path>]
```

Tipos relevantes de fonte:

- `WHATSAPP`;
- `WHATSAPP_IMAGE`.

Saídas intermediárias/finais:

```text
narrative.json
storyboard.json
render-plan.json
video.mp4
```

---

## 7. API do backend

Servidor:

```text
src/api/server.ts
```

Rotas principais:

```text
GET    /health
GET    /jobs
POST   /jobs
GET    /jobs/:jobId
GET    /jobs/:jobId/files
GET    /jobs/:jobId/download
GET    /jobs/:jobId/video
DELETE /jobs/:jobId
POST   /jobs/:jobId/regenerate
```

Responsabilidades da API:

- criar jobs locais de geração;
- listar jobs recentes;
- consultar status/progresso;
- expor arquivos gerados;
- servir vídeo final;
- permitir download;
- deletar projetos/jobs;
- regenerar vídeo.

A API não contém regra de negócio pesada; ela aciona o pipeline e retorna estado/artefatos.

---

## 8. Frontend

Diretório:

```text
viral-content-factory-frontend/
```

Tecnologias:

- React;
- Vite;
- TypeScript;
- Tailwind CSS.

Scripts principais:

- `npm run dev` ou equivalente: servidor de desenvolvimento Vite;
- `npm run build`: build de produção;
- `npm run preview`: preview local;
- `npm run typecheck`: valida TypeScript.

### 8.1 Estrutura do frontend

```text
viral-content-factory-frontend/
  public/
  src/
    app/
    components/
    core/
      helpers/
      schemas/
      types/
    lib/
    pages/
      public/
      private/
    services/
      api/
    index.css
    main.tsx
  .env.example
  .gitignore
  package.json
  README.md
  tsconfig.json
  vite.config.ts
  tailwind.config.js
  postcss.config.js
```

### 8.2 Arquitetura frontend

Regras:

- frontend não executa FFmpeg;
- frontend não chama LLM/TTS diretamente;
- frontend chama apenas a API do backend;
- frontend deve lidar com backend offline com mensagem clara;
- lógica de renderização/narrativa fica no backend.

### 8.3 App e páginas

Arquivos principais:

- `src/app/App.tsx`: shell principal, layout, sidebar e troca entre home/projeto;
- `src/pages/public/HomePage.tsx`: tela inicial com upload de print e início da geração;
- `src/pages/public/ProjectPage.tsx`: acompanhamento de job, progresso, preview, download, regeneração e exclusão;
- `src/pages/private/`: reservado para telas futuras autenticadas.

### 8.4 Componentes

- `Sidebar.tsx`: lista projetos recentes;
- `Dropzone.tsx`: upload/drag-and-drop de imagem.

### 8.5 Serviços de API

Diretório:

```text
src/services/api/
```

Arquivos:

- `http.ts`: wrapper genérico de `fetch`;
- `generate-video-api.ts`: cliente específico do backend.

Endpoints consumidos pelo frontend:

```text
POST   /jobs
GET    /jobs
GET    /jobs/:jobId
GET    /jobs/:jobId/files
GET    /jobs/:jobId/download
GET    /jobs/:jobId/video
DELETE /jobs/:jobId
POST   /jobs/:jobId/regenerate
GET    /health
```

### 8.6 Env frontend

O frontend usa configuração em `src/lib/env.ts`, com fallback para URL local do backend.

---

## 9. Pipeline real implementado

Fluxo atual do código:

```text
Input text/image
↓
SourceAdapterRegistry
↓
WhatsAppSourceAdapter ou WhatsAppImageSourceAdapter
↓
NarrativeEngine
↓
DialogueDirector
↓
TTSProvider / Voice Casting
↓
StoryboardEngine
↓
RenderPlanBuilder
↓
FFmpegVideoRenderer
↓
Artifacts + MP4
```

### 9.1 Content Source

Entrada pode ser:

- texto de conversa WhatsApp;
- imagem/print de WhatsApp.

Para imagem:

- o OCR extrai texto;
- heurísticas agrupam mensagens;
- o fluxo segue para narrativa/storyboard.

### 9.2 Narrative Engine

Gera narrativa estruturada.

Atual:

- mock LLM para texto;
- modo especial para OCR/chat com uma cena por mensagem.

### 9.3 Dialogue Director

Pode usar:

- passthrough local;
- provider compatível com OpenAI se configurado.

### 9.4 Voice Engine / TTS

Pode usar:

- Piper local;
- espeak local;
- ElevenLabs;
- mock TTS.

Gera WAV e depois o renderer/mux converte/mixa conforme necessário.

### 9.5 Storyboard Engine

Gera cenas com:

- duração;
- texto;
- speaker;
- mensagem relacionada;
- cues visuais;
- cues de câmera;
- cues de áudio/SFX.

### 9.6 Audio/SFX

Atualmente há:

- assets locais de SFX;
- manifesto de SFX;
- eventos sonoros por cena/emotion;
- muxagem no FFmpeg.

Ainda planejado:

- engine mais completa de música de fundo;
- efeitos cômicos mais ricos.

### 9.7 Video Renderer

O `FFmpegVideoRenderer` gera o vídeo vertical final.

Responsabilidades:

- montar cenas;
- aplicar visual/camera cues;
- sincronizar áudio;
- mixar SFX;
- gerar MP4.

---

## 10. Estado atual: mockado vs real

### Real/funcional

- backend Bun + Elysia;
- frontend React/Vite/Tailwind;
- CLI de geração;
- API de jobs;
- renderização via FFmpeg;
- persistência Prisma/Postgres;
- OCR via Tesseract;
- storage local;
- TTS local via Piper/espeak;
- estrutura multi-voice;
- preview/download/regenerate/delete no frontend.

### Mockado ou simples

- `MockLLMProvider` ainda é o principal caminho para geração narrativa local;
- `MockTTSProvider` existe como fallback/mock;
- `PassthroughDirector` existe como fallback sem LLM.

### Parcial/pendente

- organização de OCR via LLM ainda futura;
- vozes cloud mais completas ainda pendentes;
- música de fundo e comic SFX ainda planejados;
- novos adapters além de WhatsApp ainda placeholders.

---

## 11. Artefatos e exemplos

Backend contém:

```text
examples/whatsapp.txt
examples/whatsapp-print.jpg
assets/sfx/manifest.json
assets/sfx/pop.wav
assets/sfx/riser.wav
assets/sfx/boom.wav
assets/tts-pronunciation.json
scripts/setup-piper.sh
scripts/setup-sfx.sh
```

Saídas esperadas por geração:

```text
output/narrative.json
output/storyboard.json
output/render-plan.json
output/video.mp4
```

---

## 12. Critério de conclusão da primeira fase

A primeira fase é considerada concluída quando:

- backend está separado e funcional;
- frontend está separado e funcional;
- CLI gera JSONs intermediários;
- CLI gera um MP4 vertical simples;
- API Elysia expõe jobs locais;
- frontend consegue iniciar geração via API;
- `.env.example` existe nos dois serviços;
- `.env`, outputs e arquivos temporários não são versionados.

Pelo status atual das implementações, a fase 1 está marcada como concluída em `013-validation-and-next-steps.md`, e o projeto já avançou para persistência, OCR, storyboard ciente de mensagens, áudio real e multi-voice parcial.

---

## 13. Próximos passos naturais

Prioridades prováveis a partir do estado atual:

1. Implementar `019-music-and-comic-sfx.md` com música de fundo e comic SFX;
2. finalizar o organizador de OCR com LLM;
3. melhorar providers cloud de voz;
4. adicionar adapters reais para Instagram DM, Twitter/X e Reddit;
5. evoluir templates visuais além de WhatsApp Chat;
6. melhorar fila/execução assíncrona se jobs ficarem pesados;
7. endurecer validações, tratamento de erro e observabilidade;
8. refinar UI/UX para preview, histórico e configuração de geração.

---

## 14. Resumo executivo

O projeto já possui um MVP funcional e separado em backend/frontend.

O backend recebe conteúdo de WhatsApp em texto ou imagem, transforma em narrativa/storyboard, gera áudio, monta um render plan e renderiza vídeo vertical via FFmpeg. A API Elysia expõe jobs locais, e o CLI permite gerar vídeos diretamente pelo terminal.

O frontend permite enviar imagem, criar job, acompanhar progresso, visualizar resultado, baixar vídeo, regenerar e deletar projeto.

A arquitetura foi construída para crescer por meio de interfaces abstratas e providers substituíveis, evitando acoplamento direto com LLMs, TTSs, storage ou renderizadores específicos.
