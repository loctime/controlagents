# Zone-Based Agent Placement

**Date:** 2026-04-20
**Status:** Approved

## Summary

Main agents and sub-agents are assigned to seats in distinct zones, defined by furniture type. Green cushioned chairs are reserved for main agents; all other chairs are for sub-agents. Sub-agents gain proper seat assignment (sit, type, wander from their seat) instead of spawning at the nearest walkable tile.

## Data Model

### `agentZone` field

Added to every layer of the catalog pipeline:

- `FurnitureManifest` (`shared/assets/manifestUtils.ts`) — `agentZone?: 'main' | 'subagent'`
- `InheritedProps` (`shared/assets/manifestUtils.ts`) — propagated to child members
- `CatalogEntry` (`shared/assets/types.ts`) — `agentZone?: 'main' | 'subagent'`
- `FurnitureCatalogEntry` (`webview-ui/src/office/layout/furnitureCatalog.ts`) — `agentZone?: 'main' | 'subagent'`
- `Seat` (`webview-ui/src/office/types.ts`) — `agentZone?: 'main' | 'subagent'`

Absence of `agentZone` means "any agent can use this seat" (backward compatible).

### Manifest change

`webview-ui/public/assets/furniture/CUSHIONED_CHAIR/manifest.json` gets `"agentZone": "main"`.

## Data Flow

```
manifest.json (CUSHIONED_CHAIR)  →  agentZone: "main"
        ↓
buildFurnitureCatalog() propagates field through CatalogEntry
        ↓
FurnitureCatalogEntry.agentZone available at runtime
        ↓
layoutToSeats() reads entry.agentZone via getCatalogEntry() → Seat.agentZone
        ↓
findFreeSeat('main')     → reserved for main agents
findFreeSeat('subagent') → reserved for sub-agents (with real seat)
```

No signature change to `layoutToSeats()` — it already calls `getCatalogEntry()` internally.

## Seat Assignment Logic

### `findFreeSeat(zone?: 'main' | 'subagent')`

1. If `zone` defined: collect free seats where `seat.agentZone === zone`
2. If none available: collect free seats where `seat.agentZone` is undefined (unzoned)
3. If still none: collect all free seats (full fallback — current behavior)
4. Within the collected set, prefer PC-facing seats (existing preference logic)

### Main agents (`addAgent`)

Call `findFreeSeat('main')`. Behavior otherwise unchanged.

### Sub-agents (`addSubagent`)

Replace closest-walkable-tile logic with seat assignment:

1. Call `findFreeSeat('subagent')`
2. If a seat is found: assign it, snap character to seat, run full FSM (idle/walk/type) from that seat
3. If no seat found: fallback to closest walkable tile (current behavior — no regression)

Sub-agents that get a real seat wander and return to it between tasks, exactly like main agents.

## Files Changed

| File                                                               | Change                                                                         |
| ------------------------------------------------------------------ | ------------------------------------------------------------------------------ |
| `shared/assets/manifestUtils.ts`                                   | Add `agentZone?` to `FurnitureManifest`, `InheritedProps`, `FurnitureAsset`    |
| `shared/assets/types.ts`                                           | Add `agentZone?` to `CatalogEntry`                                             |
| `shared/assets/build.ts`                                           | Propagate `agentZone` in `buildFurnitureCatalog()`                             |
| `webview-ui/src/office/layout/furnitureCatalog.ts`                 | Add `agentZone?` to `FurnitureCatalogEntry`                                    |
| `webview-ui/src/office/types.ts`                                   | Add `agentZone?` to `Seat`                                                     |
| `webview-ui/src/office/layout/layoutSerializer.ts`                 | Set `seat.agentZone` from catalog entry in `layoutToSeats()`                   |
| `webview-ui/src/office/engine/officeState.ts`                      | `findFreeSeat(zone?)`, update `addAgent()`, rewrite `addSubagent()` seat logic |
| `webview-ui/public/assets/furniture/CUSHIONED_CHAIR/manifest.json` | Add `"agentZone": "main"`                                                      |

## Fallbacks / Edge Cases

- No `main` seats available → main agent uses any unzoned seat
- No `subagent` seats available → sub-agent uses any unzoned seat → then any seat → then closest walkable tile
- Existing layouts with no `agentZone` chairs work unchanged (all seats are unzoned = available to all)
- Sub-agent seat is freed on `removeSubagent()` / `removeAllSubagents()` (existing seat-free logic already handles this)
