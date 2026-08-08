# 007 — Providers Mockados: LLM, TTS e Storage Local

## Objetivo

Permitir execução funcional do pipeline sem APIs externas.

## Localização

```text
viral-content-factory-backend/src/providers/
  llm/
  tts/
  storage/
```

## MockLLMProvider

Arquivo:

```text
src/providers/llm/mock-llm-provider.ts
```

Responsabilidade:

- Implementar `LLMProvider`.
- Gerar uma `Narrative` determinística a partir do texto bruto.
- Dividir o texto em cenas simples.
- Criar hook inicial.
- Alternar emoções de forma previsível para teste.

Regras sugeridas:

- Primeira cena usa hook forte.
- Cenas intermediárias usam `suspense`, `funny`, `embarrassment` quando fizer sentido.
- Última cena pode usar `plot_twist` ou `dramatic`.

## MockTTSProvider

Arquivo:

```text
src/providers/tts/mock-tts-provider.ts
```

Responsabilidade:

- Implementar `TTSProvider`.
- Não chamar API externa.
- Pode gerar arquivo `.txt` simulando fala ou retornar caminho placeholder.
- Idealmente gerar áudio simples/silencioso com FFmpeg em etapa posterior, se necessário para render.

## LocalStorageProvider

Arquivo:

```text
src/providers/storage/local-storage-provider.ts
```

Responsabilidade:

- Implementar `StorageProvider` usando filesystem local.
- Garantir criação de diretórios.
- Salvar JSON formatado.
- Ler input `.txt`.

## Critérios de aceite

- Pipeline roda sem chaves externas.
- Outputs são determinísticos para facilitar teste.
- `.env` não precisa conter provider real.
