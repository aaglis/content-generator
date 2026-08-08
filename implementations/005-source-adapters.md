# 005 — Source Adapters: WhatsApp Primeiro, Outros Preparados

## Objetivo

Implementar adaptação de fontes narrativas, começando por WhatsApp, mas deixando espaço claro para Reddit, Twitter/X e Instagram DM.

## Localização

```text
viral-content-factory-backend/src/modules/sources/
  whatsapp/
  reddit/
  twitter/
  instagram-dm/
```

## Arquivos a criar

```text
src/modules/sources/source-adapter-registry.ts
src/modules/sources/whatsapp/whatsapp-source-adapter.ts
src/modules/sources/reddit/reddit-source-adapter.placeholder.ts
src/modules/sources/twitter/twitter-source-adapter.placeholder.ts
src/modules/sources/instagram-dm/instagram-dm-source-adapter.placeholder.ts
```

## WhatsApp adapter

Responsabilidades:

- Receber texto bruto.
- Normalizar quebras de linha.
- Detectar mensagens com padrão simples quando possível.
- Preservar texto original.
- Retornar `ContentSource` com `type = WHATSAPP`.

Formato esperado de entrada inicial:

```text
Pessoa A: mensagem
Pessoa B: resposta
Pessoa A: outra mensagem
```

Também deve aceitar texto livre sem quebrar.

## Registry

Responsabilidade:

- Receber `ContentSourceType`.
- Retornar adapter compatível.
- Lançar erro claro se source ainda não for suportado.

## Placeholders

Para Reddit/Twitter/Instagram DM:

- Criar classes ou módulos placeholder.
- Não implementar parsing real.
- Mensagem explícita: `Source type not implemented yet`.

## Critérios de aceite

- WhatsApp é funcional.
- Outros sources existem na estrutura, mas falham de forma controlada.
- O pipeline não precisa conhecer detalhes do WhatsApp.
