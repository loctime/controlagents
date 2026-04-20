# Zone-Based Agent Placement Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Main agents are assigned to `agentZone: 'main'` seats (green cushioned chairs), sub-agents are assigned to `agentZone: 'subagent'` seats (all other chairs), and sub-agents sit and type from a real seat instead of spawning at the nearest walkable tile.

**Architecture:** `agentZone` flows from `manifest.json` → `CatalogEntry` → `FurnitureCatalogEntry` → `Seat` at layout parse time. `findFreeSeat(zone?)` filters by zone with two fallback levels. `addSubagent()` is refactored to assign a seat (like `addAgent()`).

**Tech Stack:** TypeScript, Vite, React, Canvas 2D. No new dependencies.

---

## File Map

| File                                                               | Change                                                                                                        |
| ------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------- |
| `shared/assets/manifestUtils.ts`                                   | Add `agentZone?` to `FurnitureManifest`, `InheritedProps`, `FurnitureAsset`                                   |
| `shared/assets/types.ts`                                           | Add `agentZone?` to `CatalogEntry`                                                                            |
| `shared/assets/build.ts`                                           | Propagate `agentZone` in `buildFurnitureCatalog()`                                                            |
| `webview-ui/src/office/layout/furnitureCatalog.ts`                 | Add `agentZone?` to `LoadedAssetData.catalog` + `FurnitureCatalogEntry`; propagate in `buildDynamicCatalog()` |
| `webview-ui/src/office/types.ts`                                   | Add `agentZone?` to `Seat`                                                                                    |
| `webview-ui/src/office/layout/layoutSerializer.ts`                 | Set `seat.agentZone` from `getCatalogEntry()` in `layoutToSeats()`                                            |
| `webview-ui/src/office/engine/officeState.ts`                      | `findFreeSeat(zone?)`, update `addAgent()`, rewrite `addSubagent()`                                           |
| `webview-ui/public/assets/furniture/CUSHIONED_CHAIR/manifest.json` | Add `"agentZone": "main"`                                                                                     |
| `webview-ui/test/dev-assets.test.ts`                               | Add test: CUSHIONED_CHAIR catalog entries have `agentZone: 'main'`                                            |

---

### Task 1: Add `agentZone` to the shared type pipeline

**Files:**

- Modify: `shared/assets/manifestUtils.ts`
- Modify: `shared/assets/types.ts`
- Modify: `shared/assets/build.ts`

- [ ] **Step 1: Add `agentZone` to `FurnitureManifest` and `InheritedProps` and `FurnitureAsset` in `shared/assets/manifestUtils.ts`**

  In `FurnitureManifest` (after `backgroundTiles: number;`):

  ```typescript
  agentZone?: 'main' | 'subagent';
  ```

  In `InheritedProps` (after `backgroundTiles: number;`):

  ```typescript
  agentZone?: 'main' | 'subagent';
  ```

  In `FurnitureAsset` (after `backgroundTiles?: number;`):

  ```typescript
  agentZone?: 'main' | 'subagent';
  ```

  In `flattenManifest()`, inside the `node.type === 'asset'` branch, after the `backgroundTiles` spread in the returned object:

  ```typescript
  ...(inherited.agentZone ? { agentZone: inherited.agentZone } : {}),
  ```

  In the group branch, `childProps` already inherits all `InheritedProps` fields via `{ ...inherited }`, so `agentZone` propagates automatically.

- [ ] **Step 2: Add `agentZone` to `CatalogEntry` in `shared/assets/types.ts`**

  After `animationGroup?: string;`:

  ```typescript
  agentZone?: 'main' | 'subagent';
  ```

- [ ] **Step 3: Propagate `agentZone` in `buildFurnitureCatalog()` in `shared/assets/build.ts`**

  In the `manifest.type === 'asset'` branch, inside `catalog.push({...})`, after `backgroundTiles: manifest.backgroundTiles,`:

  ```typescript
  ...(manifest.agentZone ? { agentZone: manifest.agentZone } : {}),
  ```

  In the group branch, update `inherited` to include `agentZone`:

  ```typescript
  const inherited: InheritedProps = {
    groupId: manifest.id,
    name: manifest.name,
    category: manifest.category,
    canPlaceOnWalls: manifest.canPlaceOnWalls,
    canPlaceOnSurfaces: manifest.canPlaceOnSurfaces,
    backgroundTiles: manifest.backgroundTiles,
    ...(manifest.agentZone ? { agentZone: manifest.agentZone } : {}),
    ...(manifest.rotationScheme ? { rotationScheme: manifest.rotationScheme } : {}),
  };
  ```

  Then in the `for (const asset of assets)` loop inside the group branch, the `catalog.push` already spreads the asset object which now includes `agentZone` from `flattenManifest`:

  ```typescript
  catalog.push({
    ...asset,
    furniturePath: `furniture/${folderName}/${asset.file}`,
  });
  ```

  No change needed there — `agentZone` is already on `FurnitureAsset` and spreads through.

- [ ] **Step 4: Type-check**

  Run: `npx tsc --noEmit -p shared/tsconfig.json` (or `npm run build` from root — it type-checks everything).

  Expected: no errors.

- [ ] **Step 5: Commit**

  ```bash
  git add shared/assets/manifestUtils.ts shared/assets/types.ts shared/assets/build.ts
  git commit -m "feat: add agentZone to shared asset type pipeline"
  ```

---

### Task 2: Propagate `agentZone` through the webview catalog to `Seat`

**Files:**

- Modify: `webview-ui/src/office/layout/furnitureCatalog.ts`
- Modify: `webview-ui/src/office/types.ts`
- Modify: `webview-ui/src/office/layout/layoutSerializer.ts`

- [ ] **Step 1: Add `agentZone?` to `LoadedAssetData.catalog` in `furnitureCatalog.ts`**

  Inside the `catalog: Array<{...}>` type in `LoadedAssetData`, after `frame?: number;`:

  ```typescript
  agentZone?: 'main' | 'subagent';
  ```

- [ ] **Step 2: Add `agentZone?` to `FurnitureCatalogEntry` in `webview-ui/src/office/types.ts`**

  `FurnitureCatalogEntry` is the interface starting at line 92. After `mirrorSide?: boolean;`:

  ```typescript
  /** Which agent type can occupy seats of this furniture type. Absent = any agent. */
  agentZone?: 'main' | 'subagent';
  ```

- [ ] **Step 3: Propagate `agentZone` in `buildDynamicCatalog()` in `furnitureCatalog.ts`**

  Inside `buildDynamicCatalog()`, in the `.map((asset) => {...})` that produces `allEntries`, the returned object already has spread conditionals. After the `...(asset.mirrorSide ? { mirrorSide: true } : {})` line, add:

  ```typescript
  ...(asset.agentZone ? { agentZone: asset.agentZone } : {}),
  ```

  The virtual `:left` entries created in the `for (const asset of assets.catalog)` loop use `...sideEntry` — `agentZone` will spread automatically from the side entry. No extra change needed.

- [ ] **Step 4: Add `agentZone?` to `Seat` in `webview-ui/src/office/types.ts`**

  Inside `interface Seat` (line 50), after `assigned: boolean;`:

  ```typescript
  /** Which agent type this seat is reserved for. Absent = any agent. */
  agentZone?: 'main' | 'subagent';
  ```

- [ ] **Step 5: Set `seat.agentZone` in `layoutToSeats()` in `layoutSerializer.ts`**

  `layoutToSeats()` starts at line 165. Inside the inner loop that creates seats, the `seats.set(seatUid, {...})` call currently sets `uid`, `seatCol`, `seatRow`, `facingDir`, `assigned`. Add `agentZone`:

  ```typescript
  seats.set(seatUid, {
    uid: seatUid,
    seatCol: tileCol,
    seatRow: tileRow,
    facingDir,
    assigned: false,
    ...(entry.agentZone ? { agentZone: entry.agentZone } : {}),
  });
  ```

  `entry` is already in scope as `getCatalogEntry(item.type)` — `agentZone` is now on `FurnitureCatalogEntry` so it's accessible.

- [ ] **Step 6: Type-check**

  Run from repo root: `npm run build`

  Expected: builds without TypeScript errors.

- [ ] **Step 7: Commit**

  ```bash
  git add webview-ui/src/office/layout/furnitureCatalog.ts webview-ui/src/office/types.ts webview-ui/src/office/layout/layoutSerializer.ts
  git commit -m "feat: propagate agentZone through webview catalog to Seat"
  ```

---

### Task 3: Zone-aware seat assignment in `officeState`

**Files:**

- Modify: `webview-ui/src/office/engine/officeState.ts`

- [ ] **Step 1: Add `zone` parameter to `findFreeSeat()`**

  Current signature (line 178): `private findFreeSeat(): string | null`

  New signature:

  ```typescript
  private findFreeSeat(zone?: 'main' | 'subagent'): string | null {
  ```

  The body collects `pcSeats` and `otherSeats` by iterating `this.seats`. Add zone filtering at the top of the iteration, **before** the `if (seat.assigned) continue` check — actually insert it **after**:

  ```typescript
  for (const [uid, seat] of this.seats) {
    if (seat.assigned) continue;
    // Zone filter: if zone requested, skip seats reserved for the other zone
    if (zone && seat.agentZone && seat.agentZone !== zone) continue;
    // ... rest of existing logic (facesPC check, push to pcSeats/otherSeats)
  ```

  This means:
  - `zone='main'` skips seats with `agentZone='subagent'` — keeps `main` and unzoned seats.
  - `zone='subagent'` skips seats with `agentZone='main'` — keeps `subagent` and unzoned seats.
  - `zone=undefined` keeps all seats (current behavior).

  **Fallback when no zone-matched seats exist:** If the first pass (with zone filter) returns `null`, call again without zone. Add a wrapper after the existing logic:

  Replace the current single `findFreeSeat()` implementation with this pattern:

  ```typescript
  private findFreeSeat(zone?: 'main' | 'subagent'): string | null {
    return this.findFreeSeatsFiltered(zone) ?? this.findFreeSeatsFiltered(undefined);
  }

  private findFreeSeatsFiltered(zone: 'main' | 'subagent' | undefined): string | null {
    // Build set of tiles occupied by electronics (PCs, monitors, etc.)
    const electronicsTiles = new Set<string>();
    for (const item of this.layout.furniture) {
      const entry = getCatalogEntry(item.type);
      if (!entry || entry.category !== 'electronics') continue;
      for (let dr = 0; dr < entry.footprintH; dr++) {
        for (let dc = 0; dc < entry.footprintW; dc++) {
          electronicsTiles.add(`${item.col + dc},${item.row + dr}`);
        }
      }
    }

    const pcSeats: string[] = [];
    const otherSeats: string[] = [];
    for (const [uid, seat] of this.seats) {
      if (seat.assigned) continue;
      if (zone && seat.agentZone && seat.agentZone !== zone) continue;

      let facesPC = false;
      const dCol =
        seat.facingDir === Direction.RIGHT ? 1 : seat.facingDir === Direction.LEFT ? -1 : 0;
      const dRow =
        seat.facingDir === Direction.DOWN ? 1 : seat.facingDir === Direction.UP ? -1 : 0;
      for (let d = 1; d <= AUTO_ON_FACING_DEPTH && !facesPC; d++) {
        const tileCol = seat.seatCol + dCol * d;
        const tileRow = seat.seatRow + dRow * d;
        if (electronicsTiles.has(`${tileCol},${tileRow}`)) {
          facesPC = true;
          break;
        }
        if (dCol !== 0) {
          if (
            electronicsTiles.has(`${tileCol},${tileRow - 1}`) ||
            electronicsTiles.has(`${tileCol},${tileRow + 1}`)
          ) {
            facesPC = true;
            break;
          }
        } else {
          if (
            electronicsTiles.has(`${tileCol - 1},${tileRow}`) ||
            electronicsTiles.has(`${tileCol + 1},${tileRow}`)
          ) {
            facesPC = true;
            break;
          }
        }
      }
      (facesPC ? pcSeats : otherSeats).push(uid);
    }

    if (pcSeats.length > 0) return pcSeats[Math.floor(Math.random() * pcSeats.length)];
    if (otherSeats.length > 0) return otherSeats[Math.floor(Math.random() * otherSeats.length)];
    return null;
  }
  ```

- [ ] **Step 2: Update `addAgent()` to pass `'main'` zone**

  In `addAgent()` (line 264), find the line:

  ```typescript
  seatId = this.findFreeSeat();
  ```

  Change to:

  ```typescript
  seatId = this.findFreeSeat('main');
  ```

- [ ] **Step 3: Rewrite `addSubagent()` to assign a real seat**

  Current `addSubagent()` (line 437) builds a spawn point from the closest walkable tile and calls `createCharacter(id, palette, null, null, hueShift)` (no seat).

  Replace the spawn logic (everything from `// Find the closest walkable tile...` down to `ch.isSubagent = true;`) with seat-based assignment, falling back to the old closest-tile logic:

  ```typescript
  // Try to assign a subagent seat
  const seatId = this.findFreeSeat('subagent');
  let ch: Character;

  if (seatId) {
    const seat = this.seats.get(seatId)!;
    seat.assigned = true;
    ch = createCharacter(id, palette, seatId, seat, hueShift);
  } else {
    // Fallback: spawn at closest free walkable tile to parent
    const parentCol = parentCh ? parentCh.tileCol : 0;
    const parentRow = parentCh ? parentCh.tileRow : 0;
    const dist = (c: number, r: number) => Math.abs(c - parentCol) + Math.abs(r - parentRow);
    const occupiedTiles = new Set<string>();
    for (const [, other] of this.characters) {
      occupiedTiles.add(`${other.tileCol},${other.tileRow}`);
    }
    let spawn = { col: parentCol, row: parentRow };
    if (this.walkableTiles.length > 0) {
      let closest = this.walkableTiles[0];
      let closestDist = Infinity;
      for (const tile of this.walkableTiles) {
        if (occupiedTiles.has(`${tile.col},${tile.row}`)) continue;
        const d = dist(tile.col, tile.row);
        if (d < closestDist) {
          closest = tile;
          closestDist = d;
        }
      }
      spawn = closest;
    }
    ch = createCharacter(id, palette, null, null, hueShift);
    ch.x = spawn.col * TILE_SIZE + TILE_SIZE / 2;
    ch.y = spawn.row * TILE_SIZE + TILE_SIZE / 2;
    ch.tileCol = spawn.col;
    ch.tileRow = spawn.row;
  }
  ```

  Keep the lines after (setting `ch.isSubagent`, `ch.parentAgentId`, `ch.matrixEffect`, etc.) unchanged.

  Also update `removeSubagent()`: sub-agents with seats already free their seat via the existing `if (ch.seatId)` check in `removeSubagent()` — verify that code path exists and works (it does, at line ~505).

- [ ] **Step 4: Type-check and build**

  Run: `npm run build`

  Expected: no TypeScript errors, build completes.

- [ ] **Step 5: Commit**

  ```bash
  git add webview-ui/src/office/engine/officeState.ts
  git commit -m "feat: zone-aware findFreeSeat and subagent seat assignment"
  ```

---

### Task 4: Tag CUSHIONED_CHAIR manifest and add catalog test

**Files:**

- Modify: `webview-ui/public/assets/furniture/CUSHIONED_CHAIR/manifest.json`
- Modify: `webview-ui/test/dev-assets.test.ts`

- [ ] **Step 1: Add `agentZone` to CUSHIONED_CHAIR manifest**

  Edit `webview-ui/public/assets/furniture/CUSHIONED_CHAIR/manifest.json`.
  Add `"agentZone": "main"` at the top level (alongside `"category"`, `"canPlaceOnWalls"`, etc.):

  ```json
  {
    "id": "CUSHIONED_CHAIR",
    "name": "Cushioned Chair",
    "category": "chairs",
    "type": "group",
    "groupType": "rotation",
    "rotationScheme": "3-way-mirror",
    "canPlaceOnWalls": false,
    "canPlaceOnSurfaces": false,
    "backgroundTiles": 0,
    "agentZone": "main",
    "members": [
      ...
    ]
  }
  ```

- [ ] **Step 2: Write a failing test in `webview-ui/test/dev-assets.test.ts`**

  The test file uses Node's built-in `test` runner and starts a Vite dev server to fetch the catalog JSON. Add a new test after the existing ones:

  ```typescript
  test('CUSHIONED_CHAIR catalog entries have agentZone: main', async () => {
    const server = await startDevServer('/', 5183);
    const url = serverUrl(server);
    try {
      const res = await fetch(assetUrl(url, '/', 'furniture-catalog.json'));
      const catalog = (await res.json()) as CatalogEntry[];
      const cushionedChairs = catalog.filter((e) => e.id.startsWith('CUSHIONED_CHAIR'));
      assert.ok(cushionedChairs.length > 0, 'Expected CUSHIONED_CHAIR entries in catalog');
      for (const entry of cushionedChairs) {
        assert.equal(
          entry.agentZone,
          'main',
          `Expected agentZone 'main' on ${entry.id}, got ${entry.agentZone}`,
        );
      }
    } finally {
      await server.close();
    }
  });
  ```

- [ ] **Step 3: Run test to verify it fails (agentZone not yet on CatalogEntry)**

  Run: `npm run test:webview`

  Expected: test fails with `Expected agentZone 'main' on CUSHIONED_CHAIR_FRONT, got undefined` (or similar).

  (If the test passes already, it means agentZone was already present — double-check Task 1 was applied correctly.)

- [ ] **Step 4: Run test again after all tasks are complete**

  All prior tasks (1–3) must be done before this passes.

  Run: `npm run test:webview`

  Expected: all tests pass including the new CUSHIONED_CHAIR test.

- [ ] **Step 5: Full build to confirm no regressions**

  Run: `npm run build`

  Expected: build succeeds, no TypeScript errors, no lint errors.

- [ ] **Step 6: Commit**

  ```bash
  git add webview-ui/public/assets/furniture/CUSHIONED_CHAIR/manifest.json webview-ui/test/dev-assets.test.ts
  git commit -m "feat: tag CUSHIONED_CHAIR as main-agent zone and add catalog test"
  ```

---

## Manual Verification

After building, open the Extension Dev Host (F5) and:

1. Place green cushioned chairs on one side of the layout and wooden chairs on the other.
2. Create 2–3 main agents — they should all take seats on the cushioned chair side.
3. Trigger sub-agents (run a Task tool in Claude Code) — sub-agents should appear and sit in the wooden chair seats.
4. If no cushioned chairs are available, main agents fall back to any unzoned seat (verify no crash).
5. Sub-agent seats are freed when the sub-agent despawns (verify the seat becomes available again).
