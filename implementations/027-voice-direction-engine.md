# 027 — Voice Direction Engine (rate/pitch por emoção no TTS)

> Fase v2 (ver `020`). Conecta `VoiceDirection` (que existia mas era ignorada) ao TTS. Depende de `023`.

## Objetivo

Dirigir **como** a voz fala — velocidade, pitch, estilo — por emoção+intensidade, e fazer o TTS honrar isso (hoje só havia um `length_scale` fixo por emoção no Piper).

## Escopo implementado

### Engine (`modules/direction/voice-direction`)
- `VoiceDirectionEngine` (`core/interfaces`): `direct(emotion, intensity) → VoiceDirection`.
- `RuleVoiceDirectionEngine` (tabela §10.2): por emoção `{rate, pitch, style}` (suspense lento/grave, anger rápido/agudo, plot_twist grave, funny deadpan…); **intensidade amplifica o desvio** (`k=(intensity-5)/5`): rate e pitch ficam mais extremos e `gainDb` sobe quando intenso. Mantido **sutil** (rate 0.8–1.2, pitch ±0–2) p/ não robotizar.
- `bucketToIntensity(bucket)`: proxy p/ a síntese pré-Score (low/med/high → 3/6/9).

### Wiring
- `SpeechSynthesisInput.direction?`.
- Pipeline calcula a direção na síntese (de `scene.emotion` + bucket) e injeta `VoiceDirectionEngine`.
- DirectionLayer grava `Beat.voice = direct(emotion, intensity_preciso)` no Score.
- **Piper** honra: `length_scale = 1/rate` (clamp 0.7–1.4) + **pitch via `rubberband=pitch:formant=preserved`** (`applyPitchShiftSemitones` em `audio-effects`). espeak/mock/ElevenLabs ignoram (graceful). Flag `VOICE_DIRECTION_ENABLED` (off = `length_scale` antigo).

## Validação
- `typecheck` exit 0.
- `score.json` `Beat.voice` varia por emoção e intensidade: suspense rate 0.90/pitch −1, funny 1.06/0, shock(int10) 1.07/+2, plot_twist(int10) 0.86/−2.
- Piper direto (mesmo texto): rate dá ordem correta de duração (0.85→3.15s, 1.0→3.01s, 1.15→2.81s); pitch (rubberband) roda e gera o áudio. **Efeito real, sutil** (o `length_scale` do Piper é sublinear — natural, não robótico).

## Observações
- Efeito intencionalmente **sutil**; se quiser mais marcado, amplificar o desvio de `length_scale` (compensar a sublinearidade do Piper) ou aumentar os pitches da tabela. Tunável sem mudar arquitetura.
- ElevenLabs (SSML/`voice_settings`) é o caminho de fidelidade máxima — futuro.

## Próximo
`028-retention-engine.md` — otimizador de arco (reorder/trim/stretch), última engine.
