# 029 — Editor de trilha sonora (UI de mapeamento de SFX)

> Pós-fase v2. Interface pra revisar a direção e **escolher onde cada efeito sonoro cai**.

## Objetivo

Tela onde o usuário vê a conversa transformada em beats (texto + emoção + voz + pausas) e **edita qual SFX cai em cada fala**, com preview de áudio e re-render rápido (sem re-sintetizar voz).

## Backend
- `SfxTrigger.beatId` — mapa SFX↔beat (LLM/rule engines + helper preenchem).
- Pipeline: persiste `voice-assets.json`; método **`renderEditedSfx({projectDir, outputPath, template, sourceType, jobId, imagePath, sfxByBeat})`** — reconstrói `score.sfx` das escolhas (`{beatId: som}`) com placement via catálogo (`punch`/`after`), salva score/storyboard/render-plan e re-renderiza **reusando narrativa + voz** (sem re-síntese). Só o SFX muda.
- Rotas (`job-routes`): `GET /jobs/:id/score`, `GET /sfx/catalog`, `GET /sfx/file/:name` (preview), `POST /jobs/:id/render-sfx`.

## Frontend (`viral-content-factory-frontend`)
- `core/types/score.ts`; `services/api` (+`getScore`/`getSfxCatalog`/`sfxFileUrl`/`renderSfx`).
- **`ScoreEditorPage`**: timeline vertical da conversa (bolhas por lado, WhatsApp-like); por beat: espinha de emoção + intensidade, chips de voz (lenta/grave/impacto) e palavra de ênfase realçada (violeta), **cue de SFX âmbar preso na palavra** (ou "depois da fala") com play/remover; **pausa dramática como gap visível** ("⏸ 0,5s"); soundboard com a paleta do usuário (preview de áudio, badge de placement); monitor de prévia (vídeo 9:16); botão "Aplicar e regenerar".
- Interação: clica a fala → seleciona; clica um som na paleta → aplica + toca. App view `editor`; ProjectPage botão "Editar trilha sonora".

## Design
Skill `interface-design`, intent-first (console de edição de meme, não formulário). Assinaturas: cue na palavra, silêncio visível, espinha de emoção, chips de voz. Tema "pipeline noturno" (âmbar=SFX, violeta=voz).

## Validação
- typecheck backend + frontend; `vite build` ok.
- `renderEditedSfx` testado: override `{train_whistle@beat, faaah@beat}` → `score.sfx` exato (train "after" t=7.2s, faaah "punch" t=29.8s) + vídeo 1080×1920 re-renderizado reusando voz.

## Modelo de clip + edição por balão (simplificado)
> A timeline horizontal do vídeo inteiro foi considerada complexa demais (feedback) e **removida**. A edição voltou pro chat, com um **modal focado por balão**.

- Modelo de **clip** por fala: `{sound, atSeconds, lengthSeconds, gainDb}` (1 som por beat). `SfxTrigger.lengthSeconds`; renderer faz `atrim=0:length` por efeito (corte/trim). `renderEditedSfx`/`render-sfx` recebem **`clips`** (posição/trim/volume manuais; respeita a colocação do usuário).
- Frontend: a conversa em chat é a tela; cada balão tem um chip de SFX (som · palavra · duração) → abre **`components/SfxBeatModal`** (editor de UM efeito): a fala com a palavra de impacto realçada; **paleta** (preview); uma **mini-faixa local do beat** (banda de pausa + fala, marcador violeta na palavra, clip arrastável com snap em fala/palavra/fim + alça de **corte/trim**); leitura "começa Xs → termina Ys" e "cai Ys antes/depois da palavra"; **slider de volume**; remover. Topo: "Prévia" (modal de vídeo) + "Aplicar e regenerar".

## Edição de texto da fala (re-narração)
- Cada balão tem um lápis → edita o texto inline (textarea); "Aplicar" manda `edits {beatId:texto}`.
- Backend `renderEditedScript`: aplica os textos novos na narrativa, **re-sintetiza a voz só das falas mudadas** (TTS + voice direction + casting), re-conduz a direção determinística (re-timing + música; sem LLM, via `new PassthroughDirectionLayer()`), re-aplica os SFX do usuário no novo timing (posição reencaixada na palavra) e re-renderiza. Rota `render-sfx` ramifica: com `edits` → `renderEditedScript`, senão → `renderEditedSfx`. Pós-apply o editor recarrega o score (texto/timing novos).
- Limitação: falas re-narradas perdem o split de ênfase antigo (viram 1 segmento); emoção/voz mantidas.

## Correções de robustez
- **Vídeo no browser:** renderer agora **H.264 (libopenh264)** em vez de mpeg4 (que dava tela preta no `<video>`). Endpoint `/video` com `Cache-Control: no-store` + Range/206; editor usa URL única por load. `dev` do backend com `--watch`.

## Pendências
- Múltiplos sons por fala; reordenar/cortar falas; editar ênfase/pausas manualmente.
