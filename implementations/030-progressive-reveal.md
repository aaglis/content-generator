# 030 — Reveal progressivo do print (anti-spoiler)

> Feedback: o crop focava a mensagem certa, mas como é 9:16 + largura do balão a janela é alta → mostrava a conversa inteira (spoiler). Deve revelar **um balão por vez** conforme o narrador fala.

## Solução

Manter o zoom/foco na mensagem, mas **cobrir tudo ABAIXO da mensagem atual** com a cor de fundo do chat → as mensagens futuras somem; o contexto acima (já narrado) continua visível → leitura contínua, sem spoiler.

## Implementado
- `CameraMovementCue.revealBottomY` (Y na imagem abaixo do qual cobrir).
- Storyboard (message-aware): `revealBottomY` = meio do gap até o próximo balão (ou logo abaixo, no último). Cobre o próximo balão inteiro.
- Renderer: amostra a **cor média do print** 1× (`sampleColor` → `scale=1:1` → 1 pixel RGB ≈ fundo do chat, que domina a imagem); por cena de mensagem, `drawbox=0:revealBottomY:iw:ih:color=…:t=fill` antes do crop. Degrada gracioso (sem cor → sem cobertura).

## Validação
- `typecheck` ok. Frames do print real: cedo = 3 balões + resto coberto; tarde = 7 balões revelados + resto coberto. Cor da cobertura blenda com o fundo (cinza-bege do WhatsApp). Sem spoiler.

## Pendências
- Scroll animado entre balões (hoje: corte por cena, cobertura estática). Amostrar a cor no gap exato (em vez da média) p/ casar fundos não-uniformes/tema escuro.
