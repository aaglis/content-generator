# 004 — Interfaces Abstratas do Core

## Objetivo

Definir contratos que permitem trocar providers reais sem alterar o pipeline principal.

## Localização

```text
viral-content-factory-backend/src/core/interfaces/
```

## Interfaces a criar

### `llm-provider.ts`

Responsabilidade:

- Receber prompt/input estruturado.
- Retornar narrativa estruturada ou JSON validável.

Métodos sugeridos:

- `generateNarrative(input): Promise<Narrative>`

Não incluir dependência direta de OpenAI, Anthropic etc.

### `tts-provider.ts`

Responsabilidade:

- Receber texto e tom de voz.
- Gerar ou simular áudio.

Métodos sugeridos:

- `synthesizeSpeech(input): Promise<VoiceAsset>`

`VoiceAsset` deve conter:

- `sceneId`
- `audioPath`
- `durationSeconds`
- `voiceTone`

### `storage-provider.ts`

Responsabilidade:

- Salvar e ler arquivos intermediários.
- Esconder detalhes de filesystem local hoje e storage remoto futuro.

Métodos sugeridos:

- `saveJson(path, data)`
- `readText(path)`
- `writeText(path, content)`
- `ensureDirectory(path)`
- `resolveOutputPath(...)`

### `video-renderer.ts`

Responsabilidade:

- Receber render plan.
- Gerar arquivo MP4.

Métodos sugeridos:

- `render(renderJob): Promise<VideoOutput>`

### `source-adapter.ts`

Responsabilidade:

- Transformar texto bruto em `ContentSource` normalizado.

Métodos sugeridos:

- `supports(sourceType): boolean`
- `parse(input): Promise<ContentSource>`

### `pipeline.ts`

Responsabilidade:

- Definir contrato de execução de geração.

Métodos sugeridos:

- `generateVideo(input): Promise<VideoOutput>`

## Critérios de aceite

- Providers reais podem ser adicionados sem mudar entidades.
- Pipeline depende de interfaces, não implementações concretas.
- Interfaces ficam no core.
