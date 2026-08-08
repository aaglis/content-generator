# Viral Content Factory — Project Context

## O que é este projeto

**Viral Content Factory** é uma ferramenta interna/local para gerar vídeos verticais automatizados para TikTok, Reels e Shorts a partir de textos narrativos.

O primeiro caso suportado será **WhatsApp Story**, mas a arquitetura deve permitir adicionar novas fontes depois:

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

## Decisão de organização

Este projeto **não usa monorepo workspace** neste momento.

Os serviços devem ser separados em diretórios independentes:

```text
content-generator/
  CLAUDE.md
  implementations/
  viral-content-factory-backend/
  viral-content-factory-frontend/
```

Cada serviço deve ter seus próprios arquivos:

```text
.env.example      # versionado
.env              # nunca versionado
.gitignore
package.json
tsconfig.json
README.md
```

## Tecnologias planejadas

### Backend

- Bun.js;
- TypeScript;
- Elysia;
- FFmpeg para renderização;
- filesystem local no MVP;
- Prisma + PostgreSQL apenas quando persistência real for necessária;
- BullMQ + Redis apenas quando jobs assíncronos/fila forem necessários.

### Frontend

- React;
- Vite;
- TypeScript;
- Tailwind CSS.

### Providers futuros

O core não deve acoplar diretamente providers reais como OpenAI, Anthropic, ElevenLabs etc.

Usar interfaces abstratas:

```ts
LLMProvider
TTSProvider
StorageProvider
VideoRenderer
SourceAdapter
```

MVP deve começar com:

```text
MockLLMProvider
MockTTSProvider
LocalStorageProvider
FFmpegVideoRenderer
WhatsAppSourceAdapter
```

## Padrão arquitetural do backend

O backend deve seguir princípios de **Clean Architecture**.

Direção de dependência desejada:

```text
api / cli / providers
↓
modules / use cases / pipeline
↓
core / entities / interfaces / types
```

Regras:

- `core` não deve importar Elysia, Bun APIs específicas, FFmpeg, Prisma ou SDKs externos.
- `core` contém entidades, tipos e interfaces.
- `modules` contém lógica de aplicação: narrative, storyboard, voice, audio, renderer, pipeline.
- `providers` implementam interfaces externas: LLM, TTS, storage, renderer.
- `api` apenas expõe HTTP e chama casos de uso/pipeline.
- `cli` apenas parseia argumentos e chama o mesmo pipeline usado pela API.
- Não colocar regra de negócio dentro de controllers Elysia.
- Não colocar regra de narrativa/renderização dentro do frontend.

Estrutura planejada:

```text
viral-content-factory-backend/
  src/
    core/
      entities/
      types/
      interfaces/
    modules/
      sources/
      narrative/
      storyboard/
      voice/
      audio/
      renderer/
      pipeline/
    providers/
      llm/
      tts/
      storage/
    api/
      routes/
    cli/
    shared/
```

## Padrão arquitetural do frontend

O frontend deve usar uma organização clara e comum para apps React:

```text
viral-content-factory-frontend/
  src/
    core/
      types/
      schemas/
      helpers/
    services/
      api/
    pages/
      public/
      private/
    components/
    app/
    lib/
  public/
```

Regras:

- `core/types`: tipos compartilhados no frontend.
- `core/schemas`: schemas de validação, preferencialmente com Zod quando necessário.
- `core/helpers`: funções puras e utilitárias.
- `services/api`: clientes HTTP e integração com backend.
- `pages/public`: telas públicas ou sem autenticação.
- `pages/private`: telas futuras que exigirem autenticação.
- `components`: componentes reutilizáveis.
- `app`: composição principal da aplicação, rotas e providers React.
- Frontend não executa FFmpeg.
- Frontend não chama LLM/TTS diretamente.
- Frontend deve tratar backend offline com mensagem clara.

## Diretório de implementações

O diretório `implementations/` contém os planos numerados de implementação.

Cada arquivo `.md` representa uma etapa planejada para outro modelo/agente implementar.

Regra de trabalho:

1. Ler `implementations/README.md` para ver o status geral.
2. Escolher a próxima atividade marcada como `pending`.
3. Ler o arquivo numerado correspondente.
4. Implementar somente o escopo daquele arquivo.
5. Atualizar `implementations/README.md` mudando o status para `implemented`, `partial` ou `blocked`.
6. Se houver mudança de plano, criar novo arquivo numerado `.md` em vez de sobrescrever decisões importantes sem registro.

## Status root das implementações

O arquivo principal de status é:

```text
implementations/README.md
```

Ele deve mostrar:

- número da etapa;
- arquivo do plano;
- status atual;
- breve descrição;
- observações/bloqueios.

Statuses permitidos:

```text
planned      # planejado, ainda não implementado
in_progress  # implementação em andamento
partial      # parcialmente implementado
implemented  # concluído conforme plano
blocked      # bloqueado por decisão, dependência ou erro
changed      # substituído por plano posterior
```

## CLI alvo do MVP

```bash
bun run generate -- --input examples/whatsapp.txt --source WHATSAPP --template WHATSAPP_CHAT --output output/video.mp4
```

Saídas intermediárias:

```text
output/narrative.json
output/storyboard.json
output/render-plan.json
output/video.mp4
```

## Critério de conclusão da primeira fase

A primeira fase estará concluída quando:

- backend estiver separado e funcional;
- frontend estiver separado e funcional;
- CLI gerar JSONs intermediários;
- CLI gerar um MP4 vertical simples;
- API Elysia expuser jobs locais;
- frontend conseguir iniciar geração via API;
- `.env.example` existir nos dois serviços;
- `.env`, outputs e arquivos temporários não forem versionados.
