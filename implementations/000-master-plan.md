# 000 — Plano Mestre: Viral Content Factory

## Objetivo

Planejar a implementação inicial do projeto **Viral Content Factory** como uma ferramenta local/interna para gerar vídeos verticais automatizados a partir de textos narrativos.

Este diretório contém planos numerados. Cada arquivo `.md` representa uma atividade implementável de forma independente ou sequencial por outro modelo/agente.

## Decisão arquitetural atual

O projeto será separado em **diretórios independentes**, não monorepo:

```text
content-generator/
  implementations/                  # planos de implementação
  viral-content-factory-backend/     # Bun + TypeScript + Elysia + CLI + FFmpeg
  viral-content-factory-frontend/    # React + Vite + TypeScript + Tailwind
```

Cada serviço terá seus próprios arquivos:

```text
.env.example      # versionado
.env              # não versionado
.gitignore
package.json
tsconfig.json
README.md
```

## Princípios

1. **Backend é o núcleo do produto.**
   - Pipeline, CLI, entidades, providers, renderização e API vivem no backend.

2. **Frontend é cliente operacional.**
   - Interface para colar texto, escolher source/template, iniciar job e baixar outputs.

3. **Sem acoplamento direto com OpenAI, ElevenLabs ou providers reais no core.**
   - Usar interfaces: `LLMProvider`, `TTSProvider`, `StorageProvider`, `VideoRenderer`.

4. **Começar com mocks.**
   - `MockLLMProvider` e `MockTTSProvider` devem permitir fluxo funcional sem APIs externas.

5. **Sem Python no primeiro MVP.**
   - FFmpeg será chamado pelo backend TypeScript.
   - Serviço Python pode ser criado futuramente se houver necessidade de ML local, Whisper, OpenCV, MoviePy ou GPU.

6. **Sem banco no primeiro MVP.**
   - Usar filesystem local para outputs e metadados.
   - Prisma/PostgreSQL entram depois se houver histórico, multiusuário, filas persistentes ou SaaS.

## Fluxo alvo

```text
Raw Text Input
↓
Content Source Adapter
↓
Narrative Engine
↓
Storyboard Engine
↓
Voice Engine
↓
Audio/Music Planner
↓
Render Plan Builder
↓
Video Renderer
↓
Output MP4 + JSON intermediários
```

## CLI alvo

```bash
bun run generate -- --input examples/whatsapp.txt --source WHATSAPP --template WHATSAPP_CHAT --output output/video.mp4
```

## Saídas esperadas

```text
output/narrative.json
output/storyboard.json
output/render-plan.json
output/video.mp4
```

## Ordem recomendada dos planos

1. `001-repository-structure.md`
2. `002-backend-foundation.md`
3. `003-core-domain-types.md`
4. `004-core-interfaces.md`
5. `005-source-adapters.md`
6. `006-narrative-storyboard-engines.md`
7. `007-mock-providers.md`
8. `008-render-plan-and-ffmpeg-renderer.md`
9. `009-cli-generate-command.md`
10. `010-backend-api-elysia.md`
11. `011-frontend-foundation.md`
12. `012-frontend-generation-ui.md`
13. `013-validation-and-next-steps.md`

## Critério de sucesso do MVP técnico

O projeto estará pronto para a próxima fase quando:

- backend instala dependências com Bun;
- CLI aceita input WhatsApp;
- narrativa mockada é gerada em JSON;
- storyboard é gerado em JSON;
- render plan é gerado em JSON;
- FFmpeg gera um MP4 vertical simples;
- frontend consegue iniciar job via API e baixar/ver outputs;
- `.env.example` existe em backend e frontend;
- `.env` e outputs locais não são versionados.
