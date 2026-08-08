# 031 — Recorte acumulado dos balões do print (anti-spoiler definitivo)

> Feedback (sobre o 030): cobrir tudo abaixo da mensagem com a cor média do print é frágil
> — depende de `sampleColor` não-nulo, fundo ~uniforme e bbox preciso. Tema escuro / wallpaper /
> agrupamento ruim do Tesseract → bloco chapado visível ou o próximo balão vaza = spoiler volta.
>
> Feedback (sobre a 1ª tentativa do 031): **não reconstruir a UI** (satori desenhando balões) —
> fica com cara que não é exatamente WhatsApp e prejudica a experiência. O vídeo deve usar os
> **recortes reais dos balões** que ele transcreveu.

## Decisão (final)

Usar os **pixels reais do print**. Por cena N, recortar o print de `y=0` até logo após a
mensagem N (coordenadas do OCR) e compor esse recorte num canvas 9:16 ancorado perto da base.
Os balões abaixo da mensagem N **nunca entram no recorte**.

> Pixels reais ⇒ cara idêntica ao WhatsApp (bubble, horário, checks, emoji, cor). Futuro fora do
> recorte ⇒ spoiler impossível por construção. Sem reconstruir UI, sem hack de cor.

Scroll natural: o recorte cresce a cada cena; quando passa da altura do quadro, mostra só a parte
**de baixo** (mensagens novas), as antigas saem pelo topo — como um chat rolando.

## Stack

- **`sharp`** (já era dep): `extract` (recorte do topo até `revealBottomY`) → `resize` (largura do
  vídeo, lanczos3) → `composite` sobre canvas de cor sólida (cor amostrada do print). Sem satori/resvg.

## Implementado

- `core/types/chat-frame.ts`: `ChatFrameSpec` = `{ printPath, printWidth/Height, revealBottomY,
  width/height, bgColor, outputPath }`.
- `core/interfaces/chat-scene-renderer.ts`: interface `ChatSceneRenderer` (core não importa SDK).
- `providers/chat-renderer/sharp-chat-renderer.ts`: `SharpChatRenderer` (recorta o print até
  `revealBottomY`, escala à largura do vídeo, ancora a base com margem segura de 5% — se passa da
  altura, mostra só a faixa de baixo) + `samplePrintBgColor` (cor de fundo p/ o canvas).
- `RenderPlanScene.framePath` + `RenderPlanOptions.frames` (map sceneId→PNG); builder anexa.
- `ffmpeg-video-renderer`: branch `frameOk` (precede 030) — usa o PNG como visual cheio
  (`scale=W:H`, sem crop/cover). `bgImagePath = frame ?? print`. Degrada pro 030 se o frame
  faltar. Corrigido `fontCandidates` (paths reais + `existsSync`) — hook cards renderizavam sem fonte.
- `pipeline.renderChatFrames`: espelha o mapeamento cena↔mensagem do storyboard message-aware
  (hook não tem balão) e o `revealBottomY` (meio do gap pro próximo balão; último = `bottom+16`);
  amostra a cor 1× e gera 1 PNG/cena em `output/.../frames/`. Chamado nas 3 rotas (gerar / editar
  SFX / editar texto). Flag `RERENDER_CHAT` (default on).

## Validação

- `typecheck` ok. Ponta-a-ponta offline no `whatsapp-print.jpg` (a piada "E ele morreu" no fim):
  9 frames; vídeo h264 1080×1920 ~26s. Frame 1 = só o balão verde real "pai, vou fazer almoço"
  (horário 11:30 + checks azuis), fundo bege casando; frame 9 = conversa inteira com a piada no
  fim. Vídeo @1s mostra só a msg 1 — sem spoiler. Amostra: `output/test031c/video.mp4`.

## Pendências

- Cor de fundo do canvas é amostrada (chapada); no tema claro com doodle há uma leve costura entre
  o recorte (com doodle) e o espaço vazio acima (cor lisa). Opção futura: tilar uma faixa de fundo
  puro do próprio print.
- Marca d'água do print de exemplo (`blog.rawrflash.com`) aparece no último recorte — usar prints
  limpos, ou recortar margem inferior.
- Upscale de prints pequenos (505px → 1080) suaviza um pouco — inerente à fonte.
- Frame estático por cena (corte entre cenas). Animar o scroll suave entre mensagens é o próximo
  polish. Karaokê palavra-a-palavra (sol. 2) viria como camada sobre o recorte com word-timestamps.
