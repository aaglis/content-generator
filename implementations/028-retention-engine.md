# 028 — Retention Engine (otimizador de arco / pacing)

> Fase v2 (ver `020`). **Última engine.** Depende de `023` (arco) / `024` (timing).

## Objetivo

Ajustar o ritmo à **curva-alvo de retenção**: detectar onde a história arrasta (intensidade ≪ alvo) e **cortar o ar morto** ali, protegendo o clímax.

## Escopo (e o que foi deferido)

As **maiores alavancas de retenção já foram entregues** nas engines anteriores (hook, pausas dramáticas, música, SFX, emoção). 028 é o **passo de acabamento** de pacing. **Deferido por restrição real**: reordenar beats quebraria a causalidade do chat; trim de fala não é possível com áudio já sintetizado. v1 = otimização de pacing + análise de arco.

## Implementado

### Engine (`modules/direction/retention`)
- `RetentionEngine` (`core/interfaces`): `plan(beats, arc) → RetentionPlan` (`{residual, isValley, paceMultiplier}` por beat + `valleyBeatIds`).
- `arc-curve.ts` `targetTension(t)` (§7.2): hook spike → setup → build → clímax(t≈0.85) → resolução.
- `ArcFitRetentionEngine`: `residual = intensity − target(t)`; **vale** = `residual < −2` fora da janela de clímax `[0.72, 0.92]`; vale → `paceMultiplier 0.6` (respiro a 60%).

### Integração
- DirectionLayer: `arc → retention → timing` (reordenado); `arc.valleyBeatIds` preenchido; `RetentionPlan` alimenta o `TimingEngine`.
- `TimingBeatInput.paceMultiplier` multiplica o `postSilenceMs` (só o respiro; pre-silence/hold dramáticos preservados). `meta.version=0.3.0`.

## Validação
- `typecheck` exit 0.
- 2 vales detectados (neutros chatos): int 2.2 @t0.14 (resid −4.5) e int 3.8 @t0.71 (resid −3.0) → `postMs` 266→160 e 314→188 (×0.6). Não-vales mantêm respiro cheio (funny +2.4 → 425; plot_twist clímax → 500). Curva/residual/vale/pace conferem na matemática.

## Fase v2 concluída
Todas as 8 engines (021–028) implementadas e validadas. Próximo (futuro): trocar assets placeholder de música por royalty-free; ElevenLabs+SSML; `loudnorm`/limiter final; animação intra-cena do hook; A/B real de retenção.
