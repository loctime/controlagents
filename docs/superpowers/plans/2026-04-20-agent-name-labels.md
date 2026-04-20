# Agent Name Labels Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the "Always Show Labels" HTML overlay behavior with always-visible canvas-rendered name labels (blue text, white stroke, no background) above each agent's head, controlled by the existing toggle.

**Architecture:** Add a `renderNameLabels()` function in `renderer.ts` that draws text directly on the game canvas. The `renderFrame()` function gains a `showNameLabels` boolean param. `ToolOverlay` no longer uses `alwaysShowOverlay` — it only shows on hover/select. The `alwaysShowOverlay` state in `App.tsx` becomes `showNameLabels` passed to `OfficeCanvas`.

**Tech Stack:** TypeScript, React, Canvas 2D API, FS Pixel Sans font (already loaded via CSS @font-face).

---

### Task 1: Add name-label constants

**Files:**

- Modify: `webview-ui/src/constants.ts`

- [ ] **Step 1: Add 4 constants at end of the visual constants section**

Open `webview-ui/src/constants.ts` and add after the existing `TEAM_ROLE_COLOR` line (around line 163):

```ts
export const NAME_LABEL_COLOR = '#66aaff';
export const NAME_LABEL_STROKE_COLOR = 'white';
export const NAME_LABEL_STROKE_WIDTH = 2;
export const NAME_LABEL_FONT_SIZE_BASE = 5; // multiplied by zoom; clamped to min 8px
export const NAME_LABEL_VERTICAL_OFFSET_PX = 18; // px above ch.y in sprite coords
```

- [ ] **Step 2: Verify TypeScript compiles**

```bash
cd webview-ui && npx tsc --noEmit
```

Expected: no errors.

- [ ] **Step 3: Commit**

```bash
git add webview-ui/src/constants.ts
git commit -m "feat: add NAME_LABEL constants for canvas name tags"
```

---

### Task 2: Add `renderNameLabels()` to renderer and thread `showNameLabels` through `renderFrame`

**Files:**

- Modify: `webview-ui/src/office/engine/renderer.ts`

- [ ] **Step 1: Add the import for new constants**

At the top of `renderer.ts`, add to the existing import from `'../../constants.js'`:

```ts
  NAME_LABEL_COLOR,
  NAME_LABEL_FONT_SIZE_BASE,
  NAME_LABEL_STROKE_COLOR,
  NAME_LABEL_STROKE_WIDTH,
  NAME_LABEL_VERTICAL_OFFSET_PX,
```

- [ ] **Step 2: Add the helper + render function before the `renderBubbles` section (around line 507)**

Insert this block immediately before the `// ── Speech bubbles ──` comment:

```ts
// ── Name labels ─────────────────────────────────────────────────

function getCharacterLabel(ch: Character): string | null {
  if (ch.isSubagent) return null;
  if (ch.teamName) return ch.isTeamLead ? (ch.teamName ?? 'LEAD') : (ch.agentName ?? null);
  return ch.folderName ?? null;
}

function renderNameLabels(
  ctx: CanvasRenderingContext2D,
  characters: Character[],
  offsetX: number,
  offsetY: number,
  zoom: number,
): void {
  const fontSize = Math.max(8, Math.round(NAME_LABEL_FONT_SIZE_BASE * zoom));
  ctx.save();
  ctx.font = `${fontSize}px "FS Pixel Sans", monospace`;
  ctx.textAlign = 'center';
  ctx.textBaseline = 'bottom';
  ctx.lineJoin = 'round';
  ctx.lineWidth = NAME_LABEL_STROKE_WIDTH;
  ctx.strokeStyle = NAME_LABEL_STROKE_COLOR;
  ctx.fillStyle = NAME_LABEL_COLOR;

  for (const ch of characters) {
    if (ch.matrixEffect === 'despawn') continue;
    const label = getCharacterLabel(ch);
    if (!label) continue;

    const sittingOff =
      ch.state === CharacterState.TYPE || ch.state === CharacterState.SLEEP
        ? CHARACTER_SITTING_OFFSET_PX
        : 0;
    const x = Math.round(offsetX + ch.x * zoom);
    const y = Math.round(offsetY + (ch.y + sittingOff - NAME_LABEL_VERTICAL_OFFSET_PX) * zoom);

    ctx.strokeText(label, x, y);
    ctx.fillText(label, x, y);
  }

  ctx.restore();
}
```

- [ ] **Step 3: Add `showNameLabels` param to `renderFrame` and call `renderNameLabels`**

In the `renderFrame` signature (around line 651), add the new optional param after `layoutRows`:

```ts
export function renderFrame(
  ctx: CanvasRenderingContext2D,
  canvasWidth: number,
  canvasHeight: number,
  tileMap: TileTypeVal[][],
  furniture: FurnitureInstance[],
  characters: Character[],
  zoom: number,
  panX: number,
  panY: number,
  selection?: SelectionRenderState,
  editor?: EditorRenderState,
  tileColors?: Array<ColorValue | null>,
  layoutCols?: number,
  layoutRows?: number,
  showNameLabels?: boolean,
): { offsetX: number; offsetY: number } {
```

Then, immediately after the `renderScene(...)` call (after line 704) and before `renderBubbles`, insert:

```ts
// Name labels (always below bubbles)
if (showNameLabels) {
  renderNameLabels(ctx, characters, offsetX, offsetY, zoom);
}
```

- [ ] **Step 4: Verify TypeScript compiles**

```bash
cd webview-ui && npx tsc --noEmit
```

Expected: no errors.

- [ ] **Step 5: Commit**

```bash
git add webview-ui/src/office/engine/renderer.ts
git commit -m "feat: add renderNameLabels canvas text above agent heads"
```

---

### Task 3: Thread `showNameLabels` through `OfficeCanvas`

**Files:**

- Modify: `webview-ui/src/office/components/OfficeCanvas.tsx`

- [ ] **Step 1: Add `showNameLabels` to `OfficeCanvasProps`**

Find the `interface OfficeCanvasProps` block and add:

```ts
showNameLabels: boolean;
```

- [ ] **Step 2: Destructure the new prop**

Find the destructuring of `OfficeCanvasProps` params in the component function and add `showNameLabels` to it.

- [ ] **Step 3: Pass `showNameLabels` to `renderFrame`**

In the `renderFrame(...)` call (around line 255), add `showNameLabels` as the last argument:

```ts
const { offsetX, offsetY } = renderFrame(
  ctx,
  w,
  h,
  officeState.tileMap,
  officeState.furniture,
  officeState.getCharacters(),
  zoom,
  panRef.current.x,
  panRef.current.y,
  selectionRender,
  editorRender,
  officeState.getLayout().tileColors,
  officeState.getLayout().cols,
  officeState.getLayout().rows,
  showNameLabels,
);
```

- [ ] **Step 4: Verify TypeScript compiles**

```bash
cd webview-ui && npx tsc --noEmit
```

Expected: no errors.

- [ ] **Step 5: Commit**

```bash
git add webview-ui/src/office/components/OfficeCanvas.tsx
git commit -m "feat: thread showNameLabels prop into OfficeCanvas renderFrame call"
```

---

### Task 4: Remove `alwaysShowOverlay` from `ToolOverlay`

**Files:**

- Modify: `webview-ui/src/office/components/ToolOverlay.tsx`

- [ ] **Step 1: Remove `alwaysShowOverlay` from `ToolOverlayProps` interface**

Delete this line from `ToolOverlayProps`:

```ts
alwaysShowOverlay: boolean;
```

- [ ] **Step 2: Remove it from the destructured props**

Remove `alwaysShowOverlay` from the function parameter destructuring.

- [ ] **Step 3: Replace the show-guard line**

Find (around line 120):

```ts
if (!alwaysShowOverlay && !isSelected && !isHovered) return null;
```

Replace with:

```ts
if (!isSelected && !isHovered) return null;
```

- [ ] **Step 4: Replace the opacity line**

Find (around line 173):

```ts
              opacity: alwaysShowOverlay && !isSelected && !isHovered ? (isSub ? 0.5 : 0.75) : 1,
```

Replace with:

```ts
              opacity: 1,
```

- [ ] **Step 5: Verify TypeScript compiles**

```bash
cd webview-ui && npx tsc --noEmit
```

Expected: no errors (will surface unused prop errors in App.tsx — that's fine, fix in next task).

- [ ] **Step 6: Commit**

```bash
git add webview-ui/src/office/components/ToolOverlay.tsx
git commit -m "refactor: ToolOverlay only shows on hover/select, remove alwaysShowOverlay"
```

---

### Task 5: Wire everything in `App.tsx`

**Files:**

- Modify: `webview-ui/src/App.tsx`

- [ ] **Step 1: Pass `showNameLabels` to `OfficeCanvas`**

Find the `<OfficeCanvas ... />` JSX (around line 247) and add the prop:

```tsx
showNameLabels = { alwaysShowOverlay };
```

- [ ] **Step 2: Remove `alwaysShowOverlay` from `ToolOverlay`**

Find the `<ToolOverlay ... />` JSX (around line 244) and remove:

```tsx
alwaysShowOverlay = { alwaysShowOverlay };
```

- [ ] **Step 3: Verify TypeScript compiles with no errors**

```bash
cd webview-ui && npx tsc --noEmit
```

Expected: no errors.

- [ ] **Step 4: Commit**

```bash
git add webview-ui/src/App.tsx
git commit -m "feat: wire alwaysShowOverlay toggle to canvas name labels"
```

---

### Task 6: Manual visual test

- [ ] **Step 1: Build the extension**

```bash
npm run build
```

Expected: build completes with no TypeScript errors.

- [ ] **Step 2: Open Extension Dev Host (F5) and verify**

- With "Always Show Labels" OFF in Settings: no name labels visible, overlay only appears on hover/select.
- Toggle "Always Show Labels" ON: blue name labels (`folderName`) appear above every agent's head, no dark background panel.
- Hover an agent: full ToolOverlay appears on top of the name label.
- Sub-agents: no name label (they are not annotated).
- Zoom in/out: labels scale with zoom.
- Label is centered above the character's head, just below where the bubble sprite would appear.

- [ ] **Step 3: Tune `NAME_LABEL_VERTICAL_OFFSET_PX` if label position is off**

If the label overlaps the head or is too far above, adjust the constant in `webview-ui/src/constants.ts`. Good range: 16–24px.

---

### Task 7: Final commit

- [ ] **Step 1: Commit any tuning changes**

```bash
git add webview-ui/src/constants.ts
git commit -m "fix: tune name label vertical offset"
```

(Skip if no tuning was needed.)
