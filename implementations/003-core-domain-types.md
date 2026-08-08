# 003 — Tipos e Entidades do Domínio

## Objetivo

Modelar os tipos principais do domínio no backend, mantendo-os independentes de API, CLI, FFmpeg ou providers externos.

## Localização

```text
viral-content-factory-backend/src/core/
  types/
  entities/
```

## Enums/tipos principais

Criar em `src/core/types/content-source.ts`:

```ts
ContentSourceType:
  WHATSAPP
  REDDIT
  TWITTER
  INSTAGRAM_DM
  TIKTOK_COMMENTS
  INSTAGRAM_COMMENTS
  NEWS
  USER_SUBMITTED_STORY
```

Criar em `src/core/types/template.ts`:

```ts
VideoTemplateType:
  WHATSAPP_CHAT
  REDDIT_STORY
  TWITTER_THREAD
  INSTAGRAM_DM
```

Criar em `src/core/types/emotion.ts`:

```ts
EmotionType:
  neutral
  funny
  suspense
  anger
  sadness
  embarrassment
  plot_twist
  shock
  romantic
  dramatic
```

## Entidades/interfaces de domínio

Criar arquivos separados para evitar concentração em um único arquivo.

### `src/core/entities/content-source.ts`

Campos sugeridos:

- `id`
- `type`
- `rawText`
- `language`
- `metadata`
- `createdAt`

### `src/core/entities/narrative.ts`

Campos sugeridos:

- `id`
- `sourceId`
- `title`
- `hook`
- `summary`
- `scenes: NarrativeScene[]`
- `overallTone`
- `estimatedDurationSeconds`

### `src/core/entities/narrative-scene.ts`

Campos sugeridos:

- `id`
- `order`
- `text`
- `emotion`
- `tensionLevel`
- `isFunnyMoment`
- `isPlotTwist`
- `voiceTone`
- `soundEffectSuggestions`
- `musicSuggestion`

### `src/core/entities/storyboard.ts`

Campos sugeridos:

- `id`
- `narrativeId`
- `template`
- `scenes: StoryboardScene[]`
- `durationSeconds`

### `src/core/entities/storyboard-scene.ts`

Campos sugeridos:

- `id`
- `order`
- `narrativeSceneId`
- `startTimeSeconds`
- `durationSeconds`
- `caption`
- `visualCues`
- `voiceCue`
- `audioCues`
- `transition`

### `src/core/entities/cues.ts`

Definir:

- `VoiceCue`
- `AudioCue`
- `VisualCue`

### `src/core/entities/render-job.ts`

Campos sugeridos:

- `id`
- `source`
- `template`
- `status`
- `inputPath`
- `outputPath`
- `narrativePath`
- `storyboardPath`
- `renderPlanPath`
- `createdAt`
- `updatedAt`
- `errorMessage?`

### `src/core/entities/video-output.ts`

Campos sugeridos:

- `jobId`
- `outputPath`
- `durationSeconds`
- `width`
- `height`
- `fps`
- `format`
- `createdAt`

## Export barrel

Criar `src/core/index.ts` exportando tipos e entidades.

## Critérios de aceite

- Domínio não importa Elysia, Bun, FFmpeg ou providers.
- Tipos estão separados por responsabilidade.
- `typecheck` passa.
