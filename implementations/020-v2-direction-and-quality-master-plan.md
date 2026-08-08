# 020 — Camada de Direção & Qualidade Viral (Master Plan v2)

> **Tipo:** documento de referência arquitetural (não é etapa de código).
> **Objetivo da fase:** transformar vídeos "corretos" em vídeos que pareçam editados por um criador humano experiente de TikTok. Foco: **retenção, emoção, ritmo, timing, voz, música e SFX** — não novas fontes.
> **Não-objetivo:** Reddit/Twitter/Instagram DM/novos adapters. Mantidos como placeholders.

---

## 0. Resumo executivo (decisão central)

Hoje o pipeline é uma cadeia linear de transformações onde cada engine reescreve o objeto inteiro. A proposta da v2 é introduzir **uma única camada intermediária — a Camada de Direção (Direction Layer)** — que produz **um único artefato novo: o `Score`** (a "partitura" do vídeo, equivalente a uma _Edit Decision List_ emocional).

```
ContentSource → Narrative → [ DIRECTION LAYER ⇒ Score ] → Storyboard → RenderPlan → Vídeo
```

Dentro da Direction Layer, as 7 engines pedidas (Emotion, Hook, Retention, Timing, Voice Direction, Music, Comic SFX) **não são 7 transformações em série**: são **anotadores que enriquecem o mesmo `Score`**, organizados por dependência (DAG). Apenas **1–2 chamadas de LLM por vídeo** (Emotion + Hook); todo o resto é **determinístico** (tabelas de mapeamento, máquinas de estado, fórmulas), portanto barato, reproduzível, testável e funciona offline via mocks.

A engine de emoção **não é construída do zero**: o `DialogueDirector` atual (LLM que já devolve `emotion` + `segments` por fala) é o protótipo dela e será **promovido**.

Ordenado por **impacto em retenção**, o roadmap é: **Hook → Emotion → Timing → Music → Comic SFX → Voice Direction → Retention** (detalhe na §18).

---

## 1. Diagnóstico do estado atual (o que já temos a favor)

Mapeamento do código real (`viral-content-factory-backend/src`) e o que cada peça já entrega:

| Já existe | Onde | Reaproveitamento na v2 |
|---|---|---|
| `EmotionType` com **exatamente** os 10 estados pedidos (`neutral, funny, suspense, anger, sadness, embarrassment, plot_twist, shock, romantic, dramatic`) | `core/types/emotion.ts` | Base do Emotion Engine — não muda. |
| `NarrativeScene` com `emotion`, `tensionLevel:number`, `isFunnyMoment`, `isPlotTwist`, `voiceTone`, `soundEffectSuggestions[]`, `musicSuggestion`, `speaker`, `speechSegments[{text,emphasis}]` | `core/entities/narrative-scene.ts` | Campos viram entrada do Score; `tensionLevel` é o embrião da intensidade; `segments` já dão granularidade **sub-cena** para pausas/SFX. |
| `DialogueDirector` (LLM context-aware) que devolve `emotion` + `segments` com `emphasis` por fala | `core/interfaces/dialogue-director.ts`, `providers/director/openai-compat-director.ts` | **É o proto-Emotion-Engine.** Estende-se para emitir `intensity` + `confidence`. Prompt já alerta contra "superatuar" — manter. |
| `VoiceCue` com `pace`, `emphasis`, `pauseBeforeMs/AfterMs`, `volume` | `core/entities/cues.ts` | Já existe o formato — **mas hoje NÃO chega ao TTS** (gap). Voice Direction passa a preenchê-lo e a sintetizar com ele. |
| `AudioCue` (`music`/`sfx`/`silence`) + storyboard emitindo cues de música/SFX | `cues.ts`, `storyboard-engine.ts` | Formato pronto; falta um produtor inteligente (Music/SFX Engines) e um consumidor no renderer. |
| SFX `byEmotion` via manifest (`buildSfxEvents`) | `modules/audio/sfx-manifest.ts` | Versão ingênua (1 SFX por emoção). Vira caso particular do Comic SFX Engine. |
| `RenderPlanScene.emotion` já trafega emoção até o render | `modules/renderer/render-plan.ts` | Canal de emoção→render já existe. |
| Duração da cena = duração do áudio de voz | `storyboard-engine.ts` (message-aware) | Bom, mas **ingênuo**: sem pausas dramáticas, sem silêncio pré-revelação. Timing Engine assume o relógio. |

**Gaps reais (o que a v2 resolve):**

1. **Hook = primeira mensagem crua** (`narrative-engine.ts: hook = firstText`). Zero reescrita. Maior alavanca de retenção, hoje vazia.
2. **Intensidade emocional inexistente** como modelo de primeira classe (só `tensionLevel` heurístico: última msg = 9, resto = 4). Sem `confidence`, sem **arco**.
3. **`VoiceCue` não conectado ao TTS** — `pace`/`pauseBefore`/`pauseAfter` são calculados e descartados. Voz só varia por pitch dos `emphasis` (Piper).
4. **Sem música** de fundo de verdade (cue existe, renderer ignora — é o pendente do plano 019).
5. **Timing ingênuo** — sem pausa antes da revelação, sem hold no punchline, sem curva de ritmo.
6. **Sem retenção** — nenhuma análise de revelação/cliffhanger além de "última mensagem = plot_twist".

---

## 2. Decisão arquitetural central — o `Score`

### 2.1 Por que um artefato único e não 8 transformações em série

A ÁREA 8 propõe uma cadeia linear de 8 engines. Como CTO, recomendo **não** implementar literalmente assim. Problemas da cadeia serial pura:

- cada estágio reabre/reescreve o objeto inteiro → acoplamento e re-leitura;
- se cada estágio fosse uma chamada de LLM, seriam ~8 chamadas/vídeo → caro, lento, não-determinístico;
- difícil testar isoladamente; difícil sincronizar áudio/voz/SFX/música porque cada um calcula seu próprio relógio.

**Decisão:** um único artefato enriquecido progressivamente — o `Score` — com **um relógio único** (`TimingPlan`) que todos leem. As engines são **anotadores idempotentes** sobre o Score, ordenados por dependência (DAG), não um cano rígido.

O `Score` é o **artefato intermediário novo**, persistido como `score.json` ao lado de `narrative.json` / `storyboard.json` / `render-plan.json`.

### 2.2 O que é o `Score`

A "partitura" do vídeo: uma timeline de **beats** (batidas narrativas), cada beat anotado com emoção+intensidade, direção de voz, e referências para as timelines de música e SFX, tudo amarrado por um plano de tempo.

- **Beat** = unidade da partitura. Inicialmente **1 beat ≈ 1 mensagem do OCR** (granularidade que já temos). Os `segments` com `emphasis` dão sub-beats para colocar pausas/SFX dentro da fala.
- O Score carrega o **arco emocional** (a curva), a **timeline de música**, a **lista de SFX** e o **TimingPlan**.

### 2.3 Direction Layer = orquestrador da DAG

Um módulo `direction-layer.ts` roda as engines na ordem de dependência e monta o Score. Se a Direction Layer estiver desligada ou falhar, **degrada graciosamente** para o comportamento atual (mesmo padrão do `try/catch` que já envolve o `dialogueDirector` na pipeline hoje).

---

## 3. Nova arquitetura v2

### 3.1 Diagrama de alto nível

```
                         ┌─────────────────────────────────────────────┐
                         │            DIRECTION LAYER                    │
ContentSource            │  (produz e enriquece o Score, 1 artefato)     │
   │                     │                                               │
   ▼                     │   Emotion ──► Retention ──► Hook              │
Narrative ───────────────┼──►  (LLM)     (determ.)     (LLM)            │
   │                     │      │            │           │               │
   │                     │      └────────────┴─────► Timing (determ.)    │
   │                     │                              │                │
   │                     │            ┌─────────────────┼──────────────┐ │
   │                     │            ▼                 ▼              ▼ │
   │                     │     VoiceDirection         Music        ComicSFX
   │                     │       (determ.)          (determ.)     (determ.)
   │                     │            └─────────────────┴──────────────┘ │
   │                     │                         ▼                      │
   │                     │                    ===> Score <===             │
   └─────────────────────┴───────────────────────┬───────────────────────┘
                                                  ▼
                       (voz é sintetizada usando VoiceDirection)
                                                  ▼
                                            Storyboard
                                                  ▼
                                           RenderPlanBuilder
                                                  ▼
                                          FFmpegVideoRenderer ──► MP4
```

### 3.2 Ordem real (DAG) vs. a cadeia literal da ÁREA 8

A cadeia pedida foi `Narrative → Hook → Emotion → Retention → Voice → Music → SFX → Timing → Storyboard`. A ordem por **dependência** é melhor e está justificada:

| Etapa | Depende de | Por quê |
|---|---|---|
| **1. Emotion** | Narrative | Tudo lê emoção+intensidade. Roda primeiro. |
| **2. Retention** | Emotion | Precisa do arco para achar clímax/revelação/cliffhanger e decidir pacing. |
| **3. Hook** | Retention | Um bom hook **provoca a revelação** — precisa saber onde está o clímax para teasá-lo. (Hook v1 pode ser standalone; v2 fica mais inteligente após Retention.) |
| **4. Timing** | Emotion + Retention | Converte pacing+intensidade em pausas, silêncios e durações → **vira o relógio único**. |
| **5–7. Voice / Music / SFX** | Emotion + Timing | São **funções puras** do Score já fixado. Rodam **em paralelo**. |
| **8. Storyboard** | Score completo | Consome tudo e produz cenas visuais/câmera + timeline de áudio mesclada. |

**Vantagem:** só 2 nós usam LLM (Emotion, Hook); os 5 restantes são determinísticos e paralelizáveis. Custo previsível, saída reproduzível, cada engine testável com fixtures.

---

## 4. Entidades novas (camada `core`)

> Contratos de **design** (TypeScript), não implementação. `core` permanece sem FFmpeg/LLM/Prisma.

### 4.1 `EmotionScore` — bate com o JSON-exemplo do enunciado

```ts
// core/entities/emotion-score.ts
export interface EmotionScore {
  emotion: EmotionType;     // os 10 já existentes
  intensity: number;        // 0..10  (modelo de intensidade — §6.3)
  confidence: number;       // 0..1
}
// Exemplo: { "emotion": "suspense", "intensity": 8, "confidence": 0.94 }
```

### 4.2 `Beat` — unidade do Score

```ts
// core/entities/beat.ts
export type BeatRole =
  | 'hook' | 'setup' | 'build' | 'reveal' | 'punchline' | 'resolution' | 'cliffhanger';

export interface Beat {
  id: string;
  order: number;
  narrativeSceneId: string;     // liga ao NarrativeScene de origem
  role: BeatRole;
  text: string;                 // texto exibido
  speaker?: string;             // lado/locutor (casting de voz)
  emotion: EmotionScore;        // substitui scene.emotion + tensionLevel
  segments: Array<{ text: string; emphasis: boolean }>; // sub-beats (já existem)
  isReveal: boolean;
  isCliffhanger: boolean;
  voice?: VoiceDirection;       // preenchido pela Voice Direction Engine
}
```

### 4.3 `EmotionArc` — a curva

```ts
// core/entities/emotion-arc.ts
export interface EmotionArc {
  points: Array<{ beatId: string; t: number; intensity: number }>; // t∈[0,1] posição
  climaxBeatId: string;
  setupBeatIds: string[];
  valleyBeatIds: string[];      // vales de tensão (candidatos a corte/aceleração)
}
```

### 4.4 `VoiceDirection`

```ts
// core/entities/voice-direction.ts
export interface VoiceDirection {
  rate: number;            // multiplicador de velocidade (0.8 lento … 1.2 rápido)
  pitchSemitones: number;  // deslocamento de pitch
  pauseBeforeMs: number;
  pauseAfterMs: number;
  style?: 'calm' | 'tense' | 'angry' | 'excited' | 'sad' | 'whisper' | 'deadpan';
  gainDb: number;          // intensidade/volume relativo
}
```

### 4.5 `MusicTimeline`

```ts
// core/entities/music-timeline.ts
export type MusicMood = 'ambient' | 'tension' | 'dramatic' | 'upbeat' | 'sad' | 'romantic' | 'eerie';

export interface MusicSegment {
  startSeconds: number;
  endSeconds: number;
  mood: MusicMood;
  trackId: string;             // resolvido pelo manifest
  gainDb: number;
  fadeInMs: number;
  fadeOutMs: number;
  duckUnderVoice: boolean;     // sidechain quando há voz
}
export interface MusicStinger {  // impacto pontual (plot twist)
  atSeconds: number;
  trackId: string;             // ex: "tension_riser", "impact_swell"
  gainDb: number;
}
export interface MusicTimeline {
  segments: MusicSegment[];
  stingers: MusicStinger[];
}
```

### 4.6 `SfxTrigger`

```ts
// core/entities/sfx-trigger.ts
export type SfxCategory =
  | 'impact' | 'riser' | 'pop' | 'record_scratch' | 'dramatic_hit'
  | 'meme' | 'notification' | 'heartbeat' | 'laugh' | 'whoosh';

export interface SfxTrigger {
  id: string;
  atSeconds: number;
  category: SfxCategory;
  trackId: string;             // arquivo via manifest
  gainDb: number;
  priority: number;            // 0..100 — resolução de conflito (§12.4)
  reason: string;              // 'plot_twist' | 'keyword:traição' | 'punchline' | ...
}
```

### 4.7 `TimingPlan` — o relógio único

```ts
// core/entities/timing-plan.ts
export interface BeatTiming {
  beatId: string;
  startSeconds: number;
  speechDurationSeconds: number;  // duração do áudio TTS
  preSilenceMs: number;           // silêncio dramático ANTES (revelação)
  postSilenceMs: number;          // respiro DEPOIS
  holdMs: number;                 // segurar último frame (punchline)
  totalSeconds: number;           // pre + speech + post + hold
}
export interface TimingPlan {
  beats: BeatTiming[];
  totalSeconds: number;
}
```

### 4.8 `Score` — artefato mestre (`score.json`)

```ts
// core/entities/score.ts
export interface Score {
  id: string;
  narrativeId: string;
  hookBeat: Beat;            // beat de abertura gerado/reescrito
  beats: Beat[];            // inclui o hook na posição 0
  arc: EmotionArc;
  timing: TimingPlan;
  music: MusicTimeline;
  sfx: SfxTrigger[];
  meta: { generator: string; version: string };
}
```

---

## 5. Interfaces das engines (camada `core/interfaces`)

```ts
// Cada engine é um anotador do Score. As determinísticas são funções puras.

export interface EmotionEngine {            // LLM (promove DialogueDirector)
  score(narrative: Narrative, lang: string): Promise<Beat[]>; // beats c/ EmotionScore
}
export interface RetentionEngine {          // determinístico
  plan(beats: Beat[]): { arc: EmotionArc; beats: Beat[] };    // marca roles/reveals
}
export interface HookEngine {               // LLM
  generate(beats: Beat[], arc: EmotionArc, lang: string): Promise<Beat>;
}
export interface TimingEngine {             // determinístico
  schedule(beats: Beat[], arc: EmotionArc, speechDurations: Map<string, number>): TimingPlan;
}
export interface VoiceDirectionEngine {     // determinístico, puro
  direct(beat: Beat, arc: EmotionArc): VoiceDirection;
}
export interface MusicEngine {              // determinístico
  compose(beats: Beat[], arc: EmotionArc, timing: TimingPlan): MusicTimeline;
}
export interface ComicSfxEngine {           // determinístico
  trigger(beats: Beat[], timing: TimingPlan): SfxTrigger[];
}

// Orquestrador
export interface DirectionLayer {
  conduct(narrative: Narrative, ctx: { lang: string; speechDurations: Map<string, number> }): Promise<Score>;
}
```

> **Nota de integração:** o `DialogueDirector` atual é **absorvido** pelo `EmotionEngine`. A impl default do `EmotionEngine` reusa `OpenAICompatDirector` (mesmo prompt, estendido para `intensity`+`confidence`). `PassthroughDirector` vira o `MockEmotionEngine` (heurística offline).

---

## 6. ÁREA 1 — Emotion Engine

### 6.1 Responsabilidades

- Ler a narrativa inteira **com contexto** e classificar **cada beat** com `{emotion, intensity, confidence}`.
- Produzir o material para o **arco emocional** (a Retention Engine fecha o arco).
- Ser a **única fonte de verdade emocional** — todas as outras engines a consomem.

### 6.2 Entidades/Interfaces

`EmotionScore` (§4.1), `Beat` (§4.2), `EmotionEngine` (§5).

### 6.3 Algoritmo de intensidade (modelo híbrido — determinístico + LLM)

Pedir um inteiro 0–10 cru ao LLM é ruidoso e não reproduzível. Modelo **híbrido**: o LLM dá emoção + um _bucket_ grosseiro + confiança; **features determinísticas** ajustam o valor final.

```
intensity = clamp(0, 10,
    EMOTION_BASE[emotion]          // expressividade-base por emoção
  + 2.0 * llmBucket                // low=0 | med=0.5 | high=1.0  (→ 0..2)
  + punctuationBoost               // "!!!", CAPS, "???"           (0..1.5)
  + lexiconBoost                   // léxico viral pt-BR matched    (0..2)
  + roleBoost                      // reveal/punchline +1.5; cliffhanger +1
  + arcPositionBoost )             // proximidade do clímax         (0..1)

confidence = clamp(0, 1, 0.5 * llmConfidence + 0.5 * heuristicAgreement)
```

`EMOTION_BASE` (ponto de partida, ajustável): `neutral 2 · romantic 4 · suspense 5 · sadness 5 · embarrassment 5 · funny 6 · anger 7 · dramatic 7 · shock 8 · plot_twist 8`.

`heuristicAgreement` = 1 se a emoção do LLM coincide com a sugerida pelo léxico/pontuação; 0.5 se neutra; 0 se conflita. Isso dá uma **confiança calibrada** e barata.

**Léxico viral pt-BR** (`emotion-lexicon.pt.ts`, editável): `traição, descobri, mentiu, terminei, bloqueou, kkk, morreu, chocada, não acredito, flagrei, print, ela viu, sumiu…` → cada termo mapeia para emoção candidata + peso. Mesmo padrão do dicionário `assets/tts-pronunciation.json` que já existe.

### 6.4 Implementações

- `LLMEmotionEngine` (default): reusa o prompt do `OpenAICompatDirector`, adiciona ao schema de saída `intensity_bucket` e `confidence`; aplica a fórmula §6.3 sobre a resposta.
- `HeuristicEmotionEngine` (mock/offline): só léxico+pontuação+posição. Garante funcionamento sem API (igual ao papel atual do `PassthroughDirector`).
- Fallback: se o LLM falhar, cai para o heurístico (não-fatal), como já é hoje.

---

## 7. ÁREA 7 — Retention Engine

### 7.1 Responsabilidades

Identificar **revelações, cliffhangers, tensão e resolução** e ajustar **duração/velocidade/intensidade** das cenas para casar com uma **curva de retenção** comprovada.

### 7.2 Modelo de arco (curva-alvo de tensão)

Tensão-alvo por posição normalizada `t∈[0,1]` — formato clássico de storytime viral:

```
target(t) =
   t < 0.05 → 9                     // HOOK: pico imediato
 | t < 0.25 → lerp(9 → 4)           // setup: assenta
 | t < 0.75 → lerp(4 → 7)           // build: subida
 | t < 0.90 → lerp(7 → 10)          // clímax/revelação
 | else     → lerp(10 → 6)          // resolução rápida
```

### 7.3 Algoritmo

1. Atribuir `BeatRole` a cada beat: maior `intensity` na região `t≈0.75–0.9` → `reveal`/`climax`; `plot_twist`/`shock` herdam papel; último beat com gancho aberto → `cliffhanger`; início → `setup`.
2. `residual(beat) = intensity(beat) − target(t(beat))`.
3. **Vales** (`residual < −2` em região de setup) → marcar para **aceleração/corte** (Timing reduz duração, Voice aumenta `rate`).
4. **Clímax** → **proteger e esticar** (Timing adiciona `preSilence`/`hold`; Voice desacelera).
5. Saída: `EmotionArc` (com `climaxBeatId`, `valleyBeatIds`) + beats com `role`/`isReveal`/`isCliffhanger`.

### 7.4 Regra de segurança (anti-quebra de causalidade)

Para fontes de **chat** (WhatsApp), a Retention Engine pode **encurtar/esticar/acelerar**, mas **NÃO reordena** beats — a conversa precisa permanecer cronológica. Reordenação fica habilitada só para fontes não-dialógicas (futuro).

---

## 8. ÁREA 5 — Hook Engine

### 8.1 Responsabilidades

Os primeiros ~2s decidem o scroll. Gerar/reescrever a **abertura** para maximizar _curiosity gap_, **sem spoilar** a revelação. Substitui o `hook = primeira mensagem crua` atual.

### 8.2 Estratégias (tipos de hook)

| Estratégia | Padrão | Exemplo (do enunciado) |
|---|---|---|
| `curiosity_gap` | pergunta/promessa de revelação | "Foi assim que descobri a traição." |
| `stakes_number` | número concreto eleva aposta | "Meu namorado mentiu pra mim por 3 meses." |
| `in_medias_res` | começa no auge | "Quando li essa mensagem, congelei." |
| `contradiction` | quebra de expectativa | "Ele disse que estava trabalhando. O recibo dizia outra coisa." |
| `controversial` | afirmação polarizadora | "Terminei por causa de UMA mensagem." |
| `cliffhanger_open` | abre laço que só fecha no fim | "Você não vai acreditar no que veio depois." |

### 8.3 Seleção + métrica

- Seleção por gênero/emoção do clímax (vindo da Retention): `plot_twist/shock` → `curiosity_gap`/`in_medias_res`; relacionamento → `stakes_number`; deboche → `controversial`.
- **Score de hook** (heurística de "scroll-stop", testável/ordenável): `len ≤ 12 palavras (+)`, contém número/aposta (+), contém _curiosity gap_ sem spoiler (+), promete pagamento (+), clickbait vazio (−). Gera-se N candidatos via LLM, escolhe-se o de maior score.
- Saída: um `Beat role='hook'` na posição 0, que **também** vira o título on-screen e (opcionalmente) uma fala curta narrada.

### 8.4 Implementações

`LLMHookEngine` (default, 1 chamada, N candidatos) · `TemplateHookEngine` (mock: aplica templates determinísticos sobre o clímax — funciona offline).

---

## 9. ÁREA 6 — Timing Engine

### 9.1 Responsabilidades

O **relógio único** do vídeo. Converte pacing (Retention) + intensidade (Emotion) em **pausas, silêncios, holds e durações concretas**. Sincroniza voz, música e SFX porque **todos leem o mesmo `TimingPlan`** (resolve o gap de sincronização atual).

### 9.2 Algoritmo (determinístico)

Para cada beat:

```
preSilenceMs  = role==='reveal'    ? round(600 * intensity/10)
              : role==='cliffhanger'? 400
              : 0
postSilenceMs = clamp(120, 500, 200 + 30*intensity)
holdMs        = role==='punchline' || isPlotTwist ? round(500 * intensity/10) : 0
speechDur     = speechDurations.get(beatId)        // áudio real do TTS
total         = preSilenceMs + speechDur*1000 + postSilenceMs + holdMs
startSeconds  = soma acumulada dos anteriores
```

A **pausa antes da revelação** (`preSilence` no `reveal`) é a assinatura do storytime de TikTok — barata e altíssimo impacto percebido.

### 9.3 Sincronização

`TimingPlan` é a fonte única: Music Engine alinha transições às fronteiras de beat; Comic SFX posiciona `atSeconds` relativo a `startSeconds`/`hold`; o renderer estende a cena para `total` (não só `speechDur`), inserindo silêncio/hold. Substitui o atual `durationSeconds = voice.durationSeconds`.

---

## 10. ÁREA 4 — Voice Direction Engine

### 10.1 Responsabilidades

Dirigir **como** a voz fala — velocidade, pitch, pausas, intensidade, estilo — em função de `{emotion, intensity}`. Hoje `VoiceCue` existe mas **não chega ao TTS**; esta engine fecha esse circuito.

### 10.2 Mapa emoção → direção (tabela determinística)

| Emoção | rate | pitch (semitons) | style | pausa antes |
|---|---|---|---|---|
| neutral | 1.00 | 0 | calm | — |
| suspense | 0.90 | −1 | tense | grande |
| shock | 0.95 | +1 | excited | **grande (antes da revelação)** |
| anger | 1.12 | +2 | angry | curta |
| sadness | 0.88 | −2 | sad | média |
| funny | 1.05 | 0 | deadpan | — (timing faz a graça) |
| plot_twist | 0.85 | −1 | tense | **grande** |
| romantic | 0.92 | −1 | calm | média |
| dramatic | 0.90 | −1 | tense | média |
| embarrassment | 1.00 | 0 | calm | curta |

`rate`/`pitch` finais escalam com `intensity` (ex.: `pitch_final = base + sign(base)*round(intensity/5)`).

### 10.3 Wiring (mudança necessária)

`SpeechSynthesisInput` (em `core/types/inputs.ts`) ganha `direction?: VoiceDirection`. Providers consomem conforme capacidade:

- **Piper** (local, default): aplica `rate` (atempo) e `pitchSemitones` (rubberband — **já fazemos isso** para `emphasis`). Pausas viram silêncio inserido pelo Timing/renderer.
- **ElevenLabs** (cloud, qualidade máxima): mapeia `style`/`intensity` para `voice_settings` (stability/style) e usa **SSML** (`<break>`, `<prosody>`) onde suportado — caminho de fidelidade alta.
- **espeak/mock**: degradam (ignoram o que não suportam).

### 10.4 Implementação

`RuleVoiceDirectionEngine` (pura, tabela §10.2) — sem LLM. Determinística e testável.

---

## 11. ÁREA 2 — Music Engine

### 11.1 Responsabilidades

Dirigir **música automaticamente** ao longo da narrativa: escolher o _bed_ por mood, automatizar intensidade, transicionar nas viradas, aplicar **ducking** sob a voz e disparar **stingers** (riser/impacto) em plot twists.

### 11.2 Arquitetura — máquina de estados de mood

A música é uma **state machine** dirigida pelo arco emocional. Estados = `MusicMood`; transições por mudança de emoção/limiar de intensidade.

| Situação no Score | Ação de música |
|---|---|
| setup / neutral | `ambient` baixo, loop, ducking on |
| build (intensity ↑) | transição → `tension`, gain sobe gradual |
| suspense sustentado | `tension`/`eerie`, sem resolução |
| **plot_twist / reveal** | **corta o bed → `riser` (stinger) → swell de impacto → bed `dramatic`** |
| alívio/resolução | reduz intensidade → `ambient`/`upbeat`, fade out |
| romantic | `romantic` bed |

Regras: crossfade entre segments (≥300ms); **ducking** via `sidechaincompress` quando há voz (já previsto no 019); um único bed por vez; stingers somam por cima.

### 11.3 Entidades

`MusicTimeline`/`MusicSegment`/`MusicStinger` (§4.5). Resolução de `trackId` por **manifest** local (`assets/music/manifest.json`: mood → arquivos + gain default), espelhando o manifest de SFX que já existe.

### 11.4 Algoritmo

1. Varre beats em ordem; agrupa beats contíguos de mood semelhante em `MusicSegment` (alinhado ao `TimingPlan`).
2. Detecta saltos de intensidade (`Δintensity ≥ 3` ou troca para `plot_twist/shock`) → insere `MusicStinger` em `startSeconds` do beat.
3. Define automação de gain por segmento (build sobe, resolução desce); marca `duckUnderVoice=true` onde há fala.

### 11.5 Implementação & mix

`StateMachineMusicEngine` (determinístico). O **mix final** é responsabilidade do renderer (FFmpeg `-filter_complex`: `amix` + `sidechaincompress` + `adelay` para stingers + `loudnorm`/limiter anti-clipping). Isto **absorve e expande o plano 019** (que vira a fundação de mixagem).

---

## 12. ÁREA 3 — Comic SFX Engine

### 12.1 Responsabilidades

Sistema inteligente de efeitos: `boom, riser, pop, record_scratch, dramatic_hit, meme, notification, heartbeat, laugh, whoosh`. Decide **quais**, **quando** e **resolve conflitos** por prioridade. Generaliza o `buildSfxEvents` atual (1 SFX por emoção).

### 12.2 Gatilhos automáticos

| Gatilho | Origem no Score | SFX típico |
|---|---|---|
| Onset de emoção forte | `intensity ≥ 7` na transição | `impact`/`dramatic_hit` |
| Plot twist / reveal | `role==='reveal'` ou `isPlotTwist` | `riser` (antes) + `dramatic_hit` (no corte) |
| Punchline | `segment.emphasis` final + `funny` | `record_scratch` / `meme` (FAAAH) / `laugh` |
| Keyword viral | léxico (ex.: "bloqueou", "print") | `pop`/`notification` |
| Suspense sustentado | `suspense` por ≥2 beats | `heartbeat` (loop baixo) |
| Mensagem nova (chat) | troca de `speaker` | `notification` sutil |
| Aceleração (vale) | beat marcado p/ corte | `whoosh` na transição |

### 12.3 Timing

SFX se ancora no `TimingPlan`: `riser` termina no início do `preSilence` do reveal; `dramatic_hit` no instante do corte; `record_scratch` no fim do punchline antes do `hold`. (O timing pode ser refinado pelo LLM no futuro, opcional.)

### 12.4 Prioridade & anti-poluição (algoritmo)

```
1. Gera todos os triggers candidatos (com priority 0..100).
2. Janela de colisão (ex.: 400ms): se 2+ caem na mesma janela, mantém o de maior priority.
3. Rate limit: no máx. N SFX por 10s (ex.: N=3) — evita "vídeo de meme poluído".
4. Caps de ganho + ducking sob a voz para não mascarar a fala.
```

Tabela de prioridade (alta→baixa): `dramatic_hit(reveal) 90 · riser 80 · record_scratch(punchline) 70 · meme 60 · heartbeat 40 · pop/notification 30 · whoosh 20`.

### 12.5 Implementação

`RuleComicSfxEngine` (determinístico) + `assets/sfx/manifest.json` estendido (já existe `byEmotion`; adiciona `byCategory`/`byKeyword`). **Absorve o `sfx-manifest.ts` atual** (que vira o caso simples).

---

## 13. Fluxo ponta-a-ponta (sequência) e novos artefatos

```
1. SourceAdapter.parse → ContentSource (OCR metadata)        [igual]
2. NarrativeEngine.generate → Narrative                       [igual]
3. DirectionLayer.conduct(narrative):
   3a. EmotionEngine.score        → beats c/ EmotionScore     [LLM, promove DialogueDirector]
   3b. RetentionEngine.plan       → arc + roles/reveals       [determ.]
   3c. HookEngine.generate        → hookBeat (posição 0)      [LLM]
   3d. (sintetiza voz p/ obter speechDurations)               [TTS, c/ VoiceDirection]
   3e. TimingEngine.schedule      → TimingPlan                [determ.]
   3f. VoiceDirection / Music / SFX  (paralelo)               [determ.]
   3g. monta Score → salva score.json                         [NOVO artefato]
4. StoryboardEngine.generate(Score) → Storyboard              [consome Score]
5. RenderPlanBuilder.build → RenderPlan (c/ música+SFX+timing)
6. FFmpegVideoRenderer.render → MP4 (mix multi-track + loudnorm)
```

**Novos artefatos de saída:** `score.json` junto de `narrative.json` / `storyboard.json` / `render-plan.json` / `video.mp4`.

> Detalhe de ordem: a voz é sintetizada **dentro** da Direction Layer (3d) porque o Timing precisa da duração real do áudio — mantém o princípio atual ("voz antes do storyboard") mas agora a direção de voz (rate/pitch/pausas) entra na síntese.

---

## 14. Estrutura de diretórios proposta

```
viral-content-factory-backend/
  assets/
    music/
      manifest.json          # mood → faixas + gain
      *.mp3|wav              # beds royalty-free (tension, dramatic, upbeat, ...)
    sfx/
      manifest.json          # byEmotion (existe) + byCategory + byKeyword
      *.wav                  # boom, riser, pop, record_scratch, faaah, heartbeat...
  src/
    core/
      entities/
        emotion-score.ts  beat.ts  emotion-arc.ts  voice-direction.ts
        music-timeline.ts  sfx-trigger.ts  timing-plan.ts  score.ts
      interfaces/
        emotion-engine.ts  retention-engine.ts  hook-engine.ts
        timing-engine.ts  voice-direction-engine.ts  music-engine.ts
        comic-sfx-engine.ts  direction-layer.ts
    modules/
      direction/
        direction-layer.ts            # orquestrador da DAG → Score
        emotion/   emotion-engine.ts  intensity-model.ts  emotion-lexicon.pt.ts
        retention/ retention-engine.ts  arc-curve.ts
        hook/      hook-engine.ts  hook-strategies.ts  hook-score.ts
        timing/    timing-engine.ts
        voice-direction/ voice-direction-engine.ts  emotion-voice-map.ts
        music/     music-engine.ts  music-state-machine.ts  music-manifest.ts
        sfx/       comic-sfx-engine.ts  sfx-rules.ts  (move sfx-manifest.ts p/ cá)
    providers/
      emotion/   llm-emotion-engine.ts  heuristic-emotion-engine.ts  # absorve director/
      hook/      llm-hook-engine.ts  template-hook-engine.ts
```

> `modules/audio/sfx-manifest.ts` migra para `modules/direction/sfx/`. `providers/director/` é absorvido por `providers/emotion/`.

---

## 15. Mudanças necessárias em código existente

| Arquivo | Mudança | Motivo |
|---|---|---|
| `core/types/inputs.ts` (`SpeechSynthesisInput`) | + `direction?: VoiceDirection` | Levar rate/pitch/pausas ao TTS. |
| `providers/tts/piper-tts-provider.ts` | honrar `direction.rate`/`pitchSemitones` | Estende o pitch-shift que já existe p/ `emphasis`. |
| `providers/tts/elevenlabs-tts-provider.ts` | mapear `style`/SSML | Caminho de fidelidade alta. |
| `modules/pipeline/video-generation-pipeline.ts` | inserir `DirectionLayer` entre Narrative e Storyboard; salvar `score.json`; voz usa `VoiceDirection` | Wire da nova camada (com `try/catch` de degradação, como hoje). |
| `modules/storyboard/storyboard-engine.ts` | consumir `Score` (timing/roles/cues) em vez de só `tensionLevel` | Cenas dirigidas pela partitura. |
| `modules/renderer/render-plan-builder.ts` + `render-plan.ts` | trafegar `MusicTimeline`/`SfxTrigger[]`/`TimingPlan` | Render precisa da timeline de áudio. |
| `modules/renderer/ffmpeg-video-renderer.ts` | mix multi-track (`amix`+`sidechaincompress`+`adelay`) + `loudnorm`/limiter; estender cena p/ `preSilence/hold` | Música+SFX+pausas no vídeo final (absorve 019). |
| `providers/director/*` | renomear/migrar p/ `providers/emotion/*` | Promoção do director a Emotion Engine. |
| `shared/config/env.ts` | flags: `DIRECTION_ENABLED`, `MUSIC_ENABLED`, `SFX_ENABLED`, `HOOK_ENABLED`, `EMOTION_PROVIDER` | Liga/desliga por engine; mock-first. |

**Princípio mantido:** `core` continua sem FFmpeg/LLM/Prisma. Engines determinísticas vivem em `modules`; o único LLM (emotion/hook) fica atrás de provider.

---

## 16. Decisões técnicas (resumo)

1. **Um `Score`, não 8 transformações em série.** Relógio único (`TimingPlan`) resolve sincronização.
2. **Só Emotion + Hook usam LLM.** Voice/Music/SFX/Timing/Retention determinísticos → barato, reproduzível, testável, offline via mocks.
3. **Promover, não recriar.** `DialogueDirector` → `EmotionEngine`; `sfx-manifest` → `ComicSfxEngine`; `VoiceCue` finalmente conectado ao TTS.
4. **Beat ≈ mensagem; `emphasis segments` dão sub-beat.** Sem inventar granularidade nova no MVP.
5. **Degradação graciosa.** Direction Layer desligável; falha de LLM cai p/ heurística (padrão atual).
6. **Sem reordenar chat.** Causalidade cronológica protegida; só trim/stretch/accelerate.
7. **Mix com `loudnorm`/limiter** (EBU R128) — voz sempre inteligível; anti-clipping.
8. **Assets locais royalty-free**, versionados, resolvidos por manifest (música e SFX).

---

## 17. Riscos & mitigações

| Risco | Impacto | Mitigação |
|---|---|---|
| LLM não-determinístico / custo / latência | médio | Cache por hash do input; schema-validate; `HeuristicEmotionEngine` offline; só 1–2 chamadas/vídeo. |
| **Over-editing** (cringe: SFX demais, voz superatuada) | **alto** | Caps de intensidade; rate-limit de SFX (§12.4); prompt já proíbe superatuar; flag de "sutileza". |
| Licenciamento de música/SFX | alto (legal) | Só packs royalty-free/CC0 locais; documentar origem no manifest. |
| Piper não honra pitch/rate finos | médio | Voice Direction degrada; ElevenLabs+SSML como caminho premium. |
| Mix FFmpeg: clipping/perf | médio | `loudnorm`+limiter; pré-render de stems; manter params uniformes p/ o concat (como já é). |
| Timing quebra sync com legenda/câmera | médio | `TimingPlan` é fonte única; storyboard/câmera derivam dele. |
| Reordenação quebra causalidade do chat | alto | Proibida p/ fontes de chat (§7.4). |
| Complexidade da DAG | médio | Cada engine isolada e testável; Score versionado; rollout por flag, engine a engine. |

---

## 18. Roadmap por impacto em retenção (fases)

Ordenado por **retorno em retenção ÷ esforço**. Cada fase é um plano numerado a criar.

| Ordem | Fase | Plano | Por que essa posição | Esforço |
|---|---|---|---|---|
| **0** | **Score + DirectionLayer (esqueleto)** | `021` | Habilitador; refator sem mudança de comportamento (passthrough). | M |
| **1** | **Hook Engine** | `022` | **Maior ROI**: os 2 primeiros segundos decidem o scroll; hoje hook = msg crua. Baixo esforço. | B |
| **2** | **Emotion Engine** (intensity+confidence+arco) | `023` | Fundação de tudo; promove o director. Destrava 3–7. | M |
| **3** | **Timing Engine** | `024` | Pausa antes da revelação + ritmo = assinatura do storytime. Determinístico, salto enorme de qualidade percebida. | M |
| **4** | **Music Engine** | `025` | Bed emocional + swell na virada → imersão. Absorve 019. | M-A |
| **5** | **Comic SFX Engine** | `026` | boom/riser/record-scratch nos beats → viralização. Absorve 019. | M |
| **6** | **Voice Direction Engine** | `027` | rate/pitch/pausas por emoção; impacto médio no Piper, alto com cloud TTS. | M |
| **7** | **Retention Engine** (otimizador de arco) | `028` | Mais sofisticado (reorder/trim/stretch); risco de over-edit → por último, sobre base sólida. | A |

> **Ordem de execução:** `021 → 022 → 023 → 024 → 025 → 026 → 027 → 028`.
> Hook (022) entrega valor imediato mesmo antes do Emotion completo (v1 standalone; v2 fica mais esperto após 023).

---

## 19. Métricas de sucesso (como saber que melhorou)

- **Proxy de retenção offline:** hook score (§8.3), aderência do arco realizado à curva-alvo (§7.2), densidade de pausas dramáticas em reveals, nº de SFX por minuto dentro do teto.
- **Qualidade de áudio:** LUFS integrado dentro de faixa-alvo, sem clipping, voz acima do bed em ducking.
- **A/B real (quando publicar):** retenção 3s, watch-time médio, replays — comparando com/sem cada engine via flags.
- **Determinismo:** mesmo input (mesmo hash) → mesmo Score nas engines determinísticas (teste de regressão).

---

## 20. Conexão com os planos existentes

- **019 (música+SFX):** marcado `changed` — sua arquitetura de "mix único no FFmpeg" é **absorvida e expandida** pelas engines de Música (025) e Comic SFX (026). O trabalho de `-filter_complex`/ducking previsto no 019 é a **fundação de mixagem** dessas fases.
- **018 (multi-voz):** Voice Direction (027) constrói **sobre** o casting já feito; ElevenLabs+SSML é o caminho de fidelidade.
- **015/016 (OCR / storyboard message-aware):** inalterados; os beats nascem das mensagens do OCR; o `TimingPlan` agora dirige a câmera por bolha (que já existe).

---

### Próximo passo

Criar `021-score-and-direction-layer-skeleton.md` (artefato `Score` + orquestrador passthrough, sem mudança de comportamento) e seguir o roadmap §18. Nenhum código nesta etapa — este documento é a referência da fase.
