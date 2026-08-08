# 034 — Render Options por job (UI), saindo do .env global

## Problema

Escolhas criativas/de saída do vídeo (resolução, fps, intensidade de movimento, lado da voz
grave, liga/desliga de hook/música/SFX/direção/voz dirigida) eram **só env global**. Mudar
uma exigia editar `.env` e **reiniciar o servidor** — errado para uma ferramenta de criador,
e impossível variar entre dois vídeos.

## Escopo

Criar `RenderOptions` que viaja na requisição do job, com fallback nos defaults do `env`.
Painel de opções no frontend. Re-renders (regenerate, editor de SFX/script) reusam as opções
escolhidas. Não migra casting de voz por interlocutor (fase 2).

## Mudanças — backend

- `core/types/render-options.ts` (novo): `RenderOptions` (tudo opcional) + `ResolvedRenderOptions`.
  `core` não importa env — só a forma.
- `shared/config/render-options.ts` (novo): `resolveRenderOptions(o)` faz merge sobre o env +
  clamp (width 240–2160, height 240–3840, fps 15–60, scale 0–3).
- `core/types/inputs.ts`: `VideoGenerationInput.options?`.
- `pipeline`: resolve 1× no início; usa `opts.*` no lugar de `env.*` (hook, voiceDeepSide,
  voiceDirection, direction, dims, rerenderChat); passa flags de render pro builder; **persiste
  `options.json`** no projeto. `loadOptions()` recarrega nos re-renders (`renderEditedSfx`/`Script`).
- `render-plan(.ts/-builder)`: `RenderPlan` ganha `musicEnabled`/`sfxEnabled`/`cameraMotion`/
  `cameraMotionScale`; renderer lê do plan (fallback env). `buildPushInFilter` recebe os flags
  por parâmetro (não lê env direto). `mixMusic`/`mixSfx` gateiam pelo plan; SFX off pula até o
  fallback byEmotion.
- `api/routes/job-routes.ts`: `renderOptionsSchema` (zod) no body; POST repassa; **regenerate
  recarrega `options.json`** pra não perder o formato.

## Mudanças — frontend

- `core/types/generate-video.ts`: `RenderOptions` + `CreateJobRequest.options`.
- `components/VideoOptionsPanel.tsx` (novo): painel colapsável no tema "pipeline noturno"
  (tokens `bg-card`/`accent`/`text-*`). Seções: **Formato** (9:16/4:5/1:1 → width/height),
  **Taxa de quadros** (30/60), **Movimento** (switch + Sutil/Padrão/Forte → scale 0.6/1/1.6),
  **Voz grave** (Esquerda/Direita), **Camadas** (switches: Hook, Trilha, Efeitos, Direção,
  Voz dirigida). Linha-resumo quando fechado. Subcomponentes `Segmented`/`Switch`/`Row`
  acessíveis (`aria-pressed`/`role=switch`). `DEFAULT_OPTIONS` espelha os defaults do backend.
- `HomePage`: estado `options`, painel sob o dropzone, repassa no `createJob`.

## Validação

- Backend `typecheck` + frontend `tsc -b && vite build` → ok.
- `resolveRenderOptions`: default vem do env (1080×1920/30), override aplicado, clamp
  (fps 999→60, scale 50→3, width 10→240).
- E2E do pipeline com `{1080×1080, music:false, scale:1.6}` → `options.json` salvo + MP4 na
  resolução escolhida (ver histórico do README).

## Resultado

De ~35 envs, ~13 viram opção de job (UI); `.env` fica só com infra/segredo. Mais 5 envs mortos
já removidos (etapa anterior). Fase 2 (casting de voz por interlocutor) fica pendente.
