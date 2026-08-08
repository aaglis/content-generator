# 012 — UI de Geração de Vídeo

## Objetivo

Criar a primeira tela operacional do frontend.

## Tela principal

Nome sugerido:

```text
GenerateVideoPage
```

## Campos

### Texto bruto

- Textarea grande.
- Placeholder com exemplo de conversa WhatsApp.

### Source

Select com opções:

- `WHATSAPP`
- `REDDIT` desabilitado ou marcado como futuro
- `TWITTER` desabilitado ou marcado como futuro
- `INSTAGRAM_DM` desabilitado ou marcado como futuro

### Template

Select com opções:

- `WHATSAPP_CHAT`
- `REDDIT_STORY` futuro
- `TWITTER_THREAD` futuro
- `INSTAGRAM_DM` futuro

### Botão

- `Generate Video`
- Estado loading durante chamada.

## Estados da tela

- idle
- generating
- completed
- failed

## Resultado exibido

Após gerar:

- Job ID.
- Status.
- Link/download do vídeo.
- Links para JSONs intermediários, se API expuser.

## API client

Arquivo sugerido:

```text
src/features/generate-video/api.ts
```

Funções:

- `createJob(payload)`
- `getJob(jobId)`
- `getJobFiles(jobId)`

## UX inicial

- Exibir erro claro se backend estiver offline.
- Desabilitar botão se texto estiver vazio.
- Mostrar que apenas WhatsApp está implementado no MVP.

## Critérios de aceite

- Usuário consegue colar texto e gerar vídeo via API.
- Estados de loading/sucesso/erro funcionam.
- UI não promete sources ainda não implementados como funcionais.
