# 016 — Message-Aware Storyboard & Camera Engine

> **Status (implementado):** `CameraEngine.buildFocusMovement` (bbox → crop 9:16 com zoom
> adaptativo, clamp na imagem), narrativa com 1 cena por mensagem do OCR (sem cap de 8),
> `StoryboardEngine.generateMessageAware` (cue `camera-movement` por balão, duração = áudio da
> voz), `RenderPlanScene.cameraMovement` e renderer com `crop+scale` no balão. Validado no
> `print.jpg` (9 cenas, zoom por balão). **Adiado:** animação intra-cena (push-in start→end) e
> `HighlightCue` — hoje o crop é estático por cena (`startCrop === endCrop`).

## Objetivo

Refatorar o Storyboard Engine para que **cada cena = uma mensagem individual** da conversa (extraída do OCR), com zoom coordenado na região exata da mensagem na imagem. Eliminar o Ken Burns genérico e substituir por movimentos de câmera propositais.

## Contexto

Hoje:
- `NarrativeEngine` agrupa mensagens em "scenes" (parágrafos)
- `StoryboardEngine` gera cenas genéricas com Ken Burns alternado
- O zoom é aleatório (in/out/in/out) e não tem relação com o conteúdo

Futuro:
- Cada mensagem é uma cena
- O zoom foca exatamente na bounding box da mensagem
- A duração da cena é baseada na duração do áudio TTS (ou fallback por tamanho do texto)
- A transição entre cenas é um movimento suave de uma mensagem para outra

## Escopo

### 1. Refatorar `NarrativeEngine` para preservar estrutura de mensagens

Quando o source vem de `WHATSAPP_IMAGE`, o `Narrative` deve ter uma scene por mensagem, não agrupar em parágrafos.

```ts
// src/modules/narrative/narrative-engine.ts
async generate(contentSource: ContentSource, template: VideoTemplateType): Promise<Narrative> {
  if (contentSource.metadata?.ocrMessages) {
    // Modo imagem: uma scene por mensagem
    return this.generateFromOCR(contentSource, template);
  }
  // Modo texto: comportamento atual
  return this.generateFromText(contentSource, template);
}
```

### 2. Criar `CameraEngine`

Novo módulo responsável por converter bounding boxes em parâmetros de câmera para o FFmpeg.

```ts
// src/modules/camera/camera-engine.ts
export interface CameraMovement {
  startCrop: { x: number; y: number; width: number; height: number };
  endCrop: { x: number; y: number; width: number; height: number };
  durationSeconds: number;
  easing: 'linear' | 'ease-in-out';
}

export class CameraEngine {
  /**
   * Gera movimento de câmera para focar em uma mensagem específica.
   * Adiciona padding para não cortar o texto rente à borda.
   */
  buildFocusMovement(
    message: OCRMessage,
    imageWidth: number,
    imageHeight: number,
    videoWidth: number,
    videoHeight: number,
    durationSeconds: number
  ): CameraMovement {
    // 1. Calcular crop que foca na mensagem com padding (ex: 40px)
    // 2. Garantir aspect ratio do vídeo (9:16 vertical)
    // 3. Clamp para não sair dos limites da imagem
    // 4. Retornar CameraMovement
  }

  /**
   * Gera transição suave entre duas mensagens.
   */
  buildTransitionMovement(
    from: OCRMessage,
    to: OCRMessage,
    imageWidth: number,
    imageHeight: number,
    videoWidth: number,
    videoHeight: number,
    durationSeconds: number
  ): CameraMovement {
    // startCrop foca em 'from', endCrop foca em 'to'
  }
}
```

**Regras de cálculo do crop:**
- Padding: 30-50px ao redor da mensagem
- Aspect ratio: manter 9:16 (width:height = 9:16)
- Se mensagem for muito larga (ex: mensagem longa do usuário à direita), centralizar horizontalmente na área da mensagem
- Se mensagem for curta, dar mais contexto vertical (mostrar mensagens próximas)

### 3. Refatorar `StoryboardEngine`

```ts
// src/modules/storyboard/storyboard-engine.ts
async generate(narrative: Narrative, template: VideoTemplateType, options?: StoryboardOptions): Promise<Storyboard> {
  if (options?.ocrOutput) {
    return this.generateMessageAware(narrative, template, options);
  }
  return this.generateGeneric(narrative, template);
}

private generateMessageAware(narrative: Narrative, template: VideoTemplateType, options: StoryboardOptions): Storyboard {
  const { ocrOutput, videoWidth, videoHeight } = options;
  const cameraEngine = new CameraEngine();

  let elapsed = 0;
  const scenes: StoryboardScene[] = narrative.scenes.map((scene, index) => {
    const message = ocrOutput.messages[index];
    const durationSeconds = scene.durationSeconds; // já calculado pelo TTS ou fallback

    // Criar visual cue de câmera com bounding box da mensagem
    const cameraMovement = cameraEngine.buildFocusMovement(
      message, ocrOutput.imageWidth, ocrOutput.imageHeight,
      videoWidth, videoHeight, durationSeconds
    );

    // Se não for a última cena, adicionar transition cue
    const nextMessage = index < narrative.scenes.length - 1 ? ocrOutput.messages[index + 1] : null;

    return {
      ...storyboardSceneBase,
      visualCues: [
        { type: 'camera', cameraMovement },  // NOVO: cameraMovement em vez de CameraCue genérico
        { type: 'highlight', boundingBox: message.boundingBox },  // NOVO: highlight da mensagem ativa
        // background é a própria imagem, não cor sólida
      ],
      durationSeconds,
      startTimeSeconds: elapsed,
    };
  });
}
```

### 4. Atualizar tipos de Cue

```ts
// src/core/entities/cues.ts
export interface CameraMovementCue {
  type: 'camera-movement';
  startCrop: { x: number; y: number; width: number; height: number };
  endCrop: { x: number; y: number; width: number; height: number };
  easing: 'linear' | 'ease-in-out';
}

export interface HighlightCue {
  type: 'highlight';
  boundingBox: { x: number; y: number; width: number; height: number };
  color: string;      // ex: rgba(255,255,255,0.3)
  borderRadius: number;
}

export type VisualCue =
  | CameraCue           // legado (genérico)
  | CameraMovementCue   // novo (coordenado)
  | TextCue
  | ColorCue
  | HighlightCue;       // novo
```

### 5. Atualizar `RenderPlan` e `RenderPlanBuilder`

```ts
// src/modules/renderer/render-plan.ts
export interface RenderPlanScene {
  // ...campos existentes...
  cameraMovement?: CameraMovementCue;  // novo
  highlight?: HighlightCue;            // novo
  imagePath: string;                   // obrigatório no modo imagem
}
```

### 6. StoryboardOptions

```ts
// src/modules/storyboard/storyboard-options.ts
export interface StoryboardOptions {
  ocrOutput?: OCROutput;
  videoWidth?: number;
  videoHeight?: number;
  imagePath?: string;
}
```

## Critérios de aceitação

- [ ] `CameraEngine` calcula crops corretos com padding e aspect ratio 9:16
- [ ] Cada mensagem do OCR vira uma scene no storyboard
- [ ] Bounding boxes são preservadas do OCR até o RenderPlan
- [ ] `generateMessageAware` produz storyboard diferente de `generateGeneric`
- [ ] JSON de storyboard mostra `cameraMovement` e `highlight` por scene
- [ ] Fallback: se OCR falhar, volta para comportamento genérico

## Dependências

- Etapa 015 (OCR Engine) — precisa das coordenadas
- Etapa 017 (fundação de áudio) — a duração da cena vem da duração do áudio

## Próximo passo

Etapa 018: Voz real & arquitetura multi-voz (uma voz por remetente)
