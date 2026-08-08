# 011 — Fundação do Frontend React + Vite + Tailwind

## Objetivo

Criar frontend independente para operar a geração de vídeos.

## Stack

- React
- Vite
- TypeScript
- Tailwind CSS

## Localização

```text
viral-content-factory-frontend/
```

## Env vars

Arquivo: `.env.example`

```env
VITE_API_BASE_URL=http://localhost:3333
```

## Estrutura sugerida

```text
src/
  app/
    App.tsx
  components/
    Button.tsx
    Card.tsx
    Textarea.tsx
    Select.tsx
  features/
    generate-video/
      GenerateVideoPage.tsx
      api.ts
      types.ts
  lib/
    env.ts
    http.ts
  types/
```

## Scripts esperados

No `package.json`:

```json
{
  "scripts": {
    "dev": "vite",
    "build": "tsc -b && vite build",
    "preview": "vite preview",
    "typecheck": "tsc --noEmit"
  }
}
```

## Regras

- Frontend não executa FFmpeg.
- Frontend não conhece providers de LLM/TTS.
- Frontend só chama API.
- Começar com UI simples, funcional e limpa.

## Critérios de aceite

- `npm/bun run dev` sobe tela local.
- Env base URL configurável.
- Build/typecheck passam.
