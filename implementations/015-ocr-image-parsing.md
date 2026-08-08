# 015 — OCR Engine & Image Parsing

> **Status (parcial):** implementada a metade **Tesseract + heurística** — `TesseractOCRProvider`
> (texto + caixas via tesseract.js; lado por cor verde/branco com `sharp`; balões por gap
> vertical), `WhatsAppImageSourceAdapter`, source `WHATSAPP_IMAGE`, pipeline/CLI aceitam imagem
> e salvam `ocr.json`. Validado no `print.jpg` (9 mensagens, lados corretos). **Pendente:** o
> organizador via LLM (seção 4 abaixo) e o wiring no frontend/API.

## Objetivo

Transformar o pipeline para aceitar **imagem como fonte primária** em vez de texto bruto. Implementar OCR para extrair não apenas o texto, mas também as **coordenadas (bounding boxes)** de cada mensagem na imagem do WhatsApp, preservando a associação texto ↔ posição.

## Contexto

Atualmente o pipeline recebe `rawText` (ex: `examples/whatsapp.txt`) e o `WhatsAppSourceAdapter` faz parse linha a linha. O usuário quer fazer upload de uma **screenshot do WhatsApp** e o sistema deve:

1. Extrair o texto de cada bolha de mensagem
2. Saber onde (coordenadas x, y, w, h) cada mensagem está na imagem
3. Identificar qual lado da tela (esquerda/direita = remetente diferente)
4. Gerar um `ContentSource` enriquecido com esses metadados

## Escopo

### 0. Abordagem: HÍBRIDA (decisão travada)

Tesseract sozinho é frágil para agrupar linhas em balões e inferir remetente (emoji, tema escuro, replies). LLM sozinho não dá coordenadas confiáveis. Então:

- **Tesseract** → bounding boxes precisos por palavra/linha (coordenadas).
- **LLM** (visão/texto) → organiza as linhas em balões, infere `sender`, `senderSide`, ordem e `emotion`.
- O `OCRProvider` combina os dois: roda Tesseract, manda as linhas+caixas (e/ou a imagem) ao `LLMProvider`, e devolve `OCROutput` com texto correto **e** coordenadas.

### 1. Instalar dependência OCR

```bash
cd viral-content-factory-backend
bun add tesseract.js sharp
bun add -d @types/sharp
```

`sharp` para pré-processar (resize/contraste) e melhorar a precisão do Tesseract.

### 2. Criar tipos de domínio no `core`

```ts
// src/core/types/ocr-result.ts
export interface OCRMessage {
  id: string;
  sender: string;        // nome ou identificador do remetente
  text: string;
  boundingBox: {
    x: number;           // pixels, canto superior esquerdo
    y: number;
    width: number;
    height: number;
  };
  senderSide: 'left' | 'right';  // inferido pela posição horizontal
  confidence: number;    // confiança do OCR
}

export interface OCROutput {
  messages: OCRMessage[];
  imageWidth: number;
  imageHeight: number;
}
```

### 3. Criar interface `OCRProvider` no `core`

```ts
// src/core/interfaces/ocr-provider.ts
export interface OCRProvider {
  recognize(imagePath: string): Promise<OCROutput>;
}
```

### 4. Implementar `HybridOCRProvider`

```ts
// src/providers/ocr/hybrid-ocr-provider.ts
export class HybridOCRProvider implements OCRProvider {
  constructor(private readonly llm: LLMProvider) {}
  async recognize(imagePath: string): Promise<OCROutput> {
    // 1. sharp: pré-processa (resize/contraste)
    // 2. tesseract.js: OCR em modo palavra/linha → linhas com texto + boundingBox
    // 3. LLM: recebe as linhas+caixas (e/ou a imagem) e organiza em mensagens,
    //    inferindo sender, senderSide, ordem e emotion
    // 4. Reconcilia: cada mensagem do LLM recebe a boundingBox = união das caixas
    //    das linhas que a compõem
    // 5. Retorna OCROutput
  }
}
```

**Reconciliação texto↔caixa:**
- Tesseract dá a geometria; o LLM dá a estrutura semântica (quem falou, ordem).
- A boundingBox da mensagem = união das caixas das linhas atribuídas a ela.

**Heurística de senderSide (fallback quando o LLM não decidir):**
- WhatsApp: mensagens do usuário à direita, do contato à esquerda
- `x + width/2 < imageWidth / 2` → `left` (contato), senão `right` (usuário)

> Nota: o `LLMProvider` precisa ganhar um caminho multimodal/estruturado para esta etapa (hoje só `generateNarrative`). Manter atrás da interface — `core` não importa SDK. Começar com um `MockOCRProvider` (caixas sintéticas) para destravar 016 sem key.

### 5. Criar `WhatsAppImageSourceAdapter`

```ts
// src/modules/sources/whatsapp/whatsapp-image-source-adapter.ts
export class WhatsAppImageSourceAdapter implements SourceAdapter {
  supports(sourceType: ContentSourceType): boolean {
    return sourceType === 'WHATSAPP_IMAGE';
  }

  async parse(input: SourceParseInput): Promise<ContentSource> {
    // 1. input.rawText contém o path da imagem
    // 2. Chamar ocrProvider.recognize(input.rawText)
    // 3. Converter OCROutput para ContentSource
    // 4. Metadata enriquecido com messages + boundingBoxes + imagePath
  }
}
```

Adicionar ao `SourceAdapterRegistry`.

### 6. Adicionar novo `ContentSourceType`

```ts
// src/core/types/content-source.ts
export type ContentSourceType =
  | 'WHATSAPP'
  | 'WHATSAPP_IMAGE'   // NOVO
  | 'REDDIT'
  | ...
```

### 7. Atualizar `VideoGenerationInput`

```ts
// src/core/types/inputs.ts
export interface VideoGenerationInput {
  inputPath: string;        // path do arquivo de entrada (texto ou imagem)
  outputPath: string;
  sourceType: ContentSourceType;
  template: VideoTemplateType;
  projectDir?: string;
  imagePath?: string;       // imagem de background (pode ser a mesma do input)
  language?: string;
}
```

### 8. Testes manuais

- Criar `examples/whatsapp-screenshot.png` (mock ou screenshot real)
- Rodar: `bun run generate --input examples/whatsapp-screenshot.png --source WHATSAPP_IMAGE --template WHATSAPP_CHAT --output output/video.mp4`
- Verificar se `narrative.json` contém mensagens com bounding boxes
- Verificar se OCR agrupou corretamente as mensagens

## Critérios de aceitação

- [ ] `TesseractOCRProvider` extrai texto de imagem PNG/JPG
- [ ] Bounding boxes são calculados para cada mensagem agrupada
- [ ] `senderSide` é inferido corretamente (left/right)
- [ ] `WhatsAppImageSourceAdapter` funciona ponta a ponta
- [ ] CLI consegue gerar pipeline a partir de imagem
- [ ] JSON intermediários mostram coordenadas das mensagens

## Dependências

- Etapa 014 (Prisma/PostgreSQL) — para persistir jobs com imagem
- Etapa 017 (fundação de áudio) — pré-requisito do pipeline narrado/sincronizado

## Próximo passo

Etapa 016: Message-Aware Storyboard & Camera Engine (usa as coordenadas do OCR)
