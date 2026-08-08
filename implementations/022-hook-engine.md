# 022 — Hook Engine (cena 0: card de texto + narração)

> Fase v2 (ver `020`). **Maior ROI de retenção.** Depende de `021`.

## Objetivo

Substituir o `hook = primeira mensagem crua` por um **gancho de abertura** gerado/reescrito, renderizado como **cena 0**: card de texto centralizado **+ narração por voz**, antes da conversa.

## Decisões (confirmadas com o usuário)
- **Forma:** card de texto **+ falado** (cena 0).
- **Motor:** **LLM + fallback template** (LLM com key; template determinístico sem key).

## Escopo implementado

### Motor (`providers/hook`)
- `HookEngine` (`core/interfaces/hook-engine.ts`): `generate({narrative,language}) → HookResult{text,spoken,strategy,candidates}`. Estratégias: `curiosity_gap | stakes_number | in_medias_res | contradiction | controversial | cliffhanger_open`.
- `LLMHookEngine`: OpenAI-compat (igual ao director), gera 4 candidatos, escolhe pelo **score de scroll-stop**.
- `TemplateHookEngine`: templates determinísticos rankeados pelo mesmo score (offline).
- `scoreHook` (`hook-strategies.ts`): premia frase curta (≤12 palavras), número/aposta, curiosity gap (léxico pt-BR); penaliza CAIXA ALTA / "!!!".
- `createHookEngine()`: LLM se `LLM_PROVIDER∈{deepseek,openai,openai-compat}` + key/base/model; senão template.

### Injeção (`modules/pipeline`)
Após o director, **antes da voz**: gera hook, prepende cena `isHook` à narrativa (`narrative.scenes`), seta `narrative.hook`. Flag `HOOK_ENABLED` (default on). Não-fatal (try/catch).

### Renderização da cena 0
- `NarrativeScene.isHook` / `StoryboardScene.isHook` / `RenderPlanScene.isHook` (novos campos).
- Storyboard **genérico**: hook vira cena (caption=hook, duração 3s).
- Storyboard **message-aware**: hook = card (sem crop de balão); `messageIndex` com **offset** preserva o alinhamento cena↔balão do OCR.
- Renderer: card de hook = **fundo sólido forçado** (slate `#0d1117` quando bg é preto; cor do template senão) + texto **multi-linha centralizado** (`wrapText` + várias `drawtext` empilhadas). Voz narrada via voiceAsset da cena.

## Validação
- `typecheck` exit 0.
- **Texto** (`whatsapp.txt`, mock TTS): cena 0 = card (fundo template) + 7 cenas; hook contextual via LLM.
- **Imagem** (`whatsapp-print.jpg`, OCR + mock TTS): 9 msgs → **10 cenas** (1 hook + 9), 9 crops alinhados, hook = card slate; frames conferidos (hook em t=1, balão em t=8).
- `HOOK_ENABLED=false` pula o hook; `score.json` segue gerado.

## Pendências / próximo
- Card ainda estático (sem push-in/fade in/out animado) — polimento futuro.
- Duração do hook fixa (genérico 3s) / = áudio (imagem); a **pausa dramática** real entra no Timing Engine (`024`).
- Próximo: `023-emotion-engine.md`.
