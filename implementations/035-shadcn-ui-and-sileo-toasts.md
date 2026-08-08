# 035 — Migração para shadcn/ui + toasts Sileo centralizados

## Objetivo

Trocar os componentes ad-hoc do frontend pelos componentes do **shadcn/ui** e adicionar
**toasts do Sileo** (`sileo.aaryan.design`) centralizados na tela para sucesso/erro de requisições.

## Setup

- Deps: `sileo`, `class-variance-authority`, `clsx`, `tailwind-merge`, `tailwindcss-animate`,
  `lucide-react`, `@radix-ui/react-{slot,switch,toggle-group,label,dialog,collapsible,slider,separator}`.
- `lib/utils.ts` (`cn`); `tailwind.config.js` ganha tokens shadcn (`primary`/`muted`/`destructive`/
  `popover`/`background`/`foreground`/`card`(obj)/`input`/`ring`) **mapeados à paleta existente**
  (âmbar = primary) coexistindo com os tokens atuais; `--radius` no `index.css`; plugin
  `tailwindcss-animate` + keyframes de collapsible. `main.tsx` importa `sileo/styles.css`.

## Componentes base (`src/components/ui/`)

`button`, `card`, `switch`, `toggle-group`, `label`, `dialog`, `collapsible`, `slider`,
`separator`, `badge`, `textarea` — fonte shadcn adaptada ao tema "pipeline noturno".

## Migrações

- `VideoOptionsPanel`: `Collapsible` + `ToggleGroup` (segmented) + `Switch` + `Separator`.
- `HomePage`: `Button` + `toast.success` ao iniciar geração.
- `Sidebar`: `Button` (ghost/outline) + ícones lucide.
- `Dropzone`: ícone lucide + `cn`.
- `SfxBeatModal`: `Dialog` + `Slider` (volume) + `Button`; remove overlay/Escape manuais.
- `ProjectPage`: `Button`/`Card`/`Badge` + toasts (download/regenerar/excluir/terminal).
- `ScoreEditorPage`: `Button`/`Textarea`/`Dialog` (prévia) + `toast` ao aplicar trilha; ícones lucide.

## Toasts centralizados

- `lib/toast.ts`: wrapper `toast.{success,error,info,warning}` sempre `position: 'top-center'`.
- `<Toaster position="top-center" theme="dark" />` montado no `App`.
- Erros de requisição **centralizados no http client** (`post`/`del`/`download` → `toast.error`,
  inclusive offline). Sucessos nos call sites (criar/baixar/regenerar/excluir/aplicar trilha).

## Validação

- `tsc -b` + `vite build` ok (2274 módulos). `vite preview` serve 200 com assets. Tema preservado.

## Pendência

Browser runtime test manual (build/serve verdes, sem teste E2E de UI no headless).
