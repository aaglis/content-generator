# 001 — Estrutura de Diretórios e Separação de Serviços

## Objetivo

Criar a estrutura inicial de diretórios do projeto sem implementar lógica de negócio ainda.

## Estrutura alvo

```text
content-generator/
  implementations/
  viral-content-factory-backend/
    src/
      core/
        entities/
        types/
        interfaces/
      modules/
        sources/
          whatsapp/
          reddit/
          twitter/
          instagram-dm/
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
    examples/
    output/
    temp/
    .env.example
    .gitignore
    package.json
    tsconfig.json
    README.md
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
    .env.example
    .gitignore
    package.json
    tsconfig.json
    vite.config.ts
    README.md
```

> **Nota de revisão (2026-06-19):** a estrutura do frontend foi alinhada ao `CLAUDE.md`, usando camadas `core/services/pages/components/app/lib` em vez da estrutura feature-sliced anterior.

## Regras

- Não criar monorepo workspace.
- Não criar `packages/` compartilhado agora.
- Não criar serviço Python agora.
- Não criar banco de dados agora.
- Cada serviço deve poder ser aberto, instalado e versionado separadamente.

## Arquivos a criar

### Raiz

- `implementations/` já existe com os planos.
- Opcional: `README.md` na raiz explicando que existem serviços separados.

### Backend

- `viral-content-factory-backend/package.json`
- `viral-content-factory-backend/tsconfig.json`
- `viral-content-factory-backend/.gitignore`
- `viral-content-factory-backend/.env.example`
- `viral-content-factory-backend/README.md`
- diretórios vazios listados na estrutura alvo.

### Frontend

- `viral-content-factory-frontend/package.json`
- `viral-content-factory-frontend/tsconfig.json`
- `viral-content-factory-frontend/vite.config.ts`
- `viral-content-factory-frontend/.gitignore`
- `viral-content-factory-frontend/.env.example`
- `viral-content-factory-frontend/README.md`
- diretórios listados na estrutura alvo.

## Critérios de aceite

- Existem dois serviços separados: backend e frontend.
- Cada serviço tem `.env.example` versionável.
- Cada serviço ignora `.env`, outputs, caches e dependências.
- Nenhuma lógica pesada foi implementada nesta etapa.
