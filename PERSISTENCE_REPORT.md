# Incremental Everything Persistence Architecture Report

## 1) Executive summary

This plugin uses a **hybrid persistence model** with three layers:

1. **Rem powerup slots (source of truth for per-rem SRS state)**
   - Critical incremental state (next repetition date, priority, repetition history, creation date) is written to powerup properties on each Rem.
   - This makes the data durable and naturally tied to RemNote entities.

2. **Synced storage (durable key-value state across sessions/devices)**
   - Used for data that must survive app restarts and often should sync between devices (page position, page ranges, page history, shield history, sorting options, long-lived indexes).
   - Keys are namespaced and many are parameterized (`incremental_current_page_<incRemId>_<pdfRemId>`).

3. **Session storage (ephemeral runtime cache / workflow state)**
   - Used for fast UI/reactive state and orchestration flags (queue state, pending jobs, temporary popup context, caches).
   - Rebuilt/refreshed from durable state when needed.

The combination gives:
- Durable correctness (slots + synced)
- Fast UI (session caches)
- Migration safety (legacy format detection + KB-aware transformations)

---

## 2) Where persistence is initialized

On plugin activation, the plugin registers settings/powerups/widgets, then initializes runtime behavior and caches. In light mode, it intentionally seeds card-priority cache to an empty session array instead of heavy preload. See `onActivate`.  

Key behavior:
- Registration pipeline runs first.
- Device/performance mode can change startup persistence behavior.
- Session cache bootstrapping is conditional.

---

## 3) Data domains and how each is persisted

## A. Incremental Rem SRS state (core durable model)

### Stored in Rem powerup slots
For each incremental rem, the plugin persists:
- `nextRepDate` (as Daily Document reference)
- `priority`
- `repHist` (JSON string array of review events)
- `originalIncDate` (creation date reference)

This state is written when:
- A rem is first made incremental (`initIncrementalRem`)
- A review occurs (`updateSRSDataForRem`)

### Structured payloads
History entries include rich event metadata:
- `date`, `scheduled`, `interval`
- `reviewTimeSeconds`
- `wasEarly`, `daysEarlyOrLate`
- `eventType` (`rep`, `rescheduledInQueue`, `manualDateReset`, `dismissed`, etc.)
- `priority` snapshot at event time

This is validated through Zod schemas (`IncrementalRep`, `IncrementalRem`) when reconstructing structured objects from raw slot data.

### Read path
`getIncrementalRemFromRem` reads slot data, resolves Daily Document references, parses JSON history, and returns validated `IncrementalRem` objects.

---

## B. Incremental Rem cache (fast session mirror)

The plugin mirrors all incremental rems into a session cache key (`all-incremental-rem`) to avoid repeated expensive traversal.

- `loadIncrementalRemCache` scans all rems tagged with incremental powerup, converts each via `getIncrementalRemFromRem`, stores result in session storage.
- `updateIncrementalRemCache` and `removeIncrementalRemCache` do incremental writes for single-rem updates.

This is a textbook **source-of-truth + in-memory/session projection** architecture.

---

## C. PDF progress model (durable, namespaced, bounded)

For each `(incrementalRemId, pdfRemId)` pair, the plugin creates namespaced synced keys:
- `incremental_current_page_<incRemId>_<pdfRemId>`
- `incremental_page_range_<incRemId>_<pdfRemId>`
- `incremental_page_history_<incRemId>_<pdfRemId>`

### Page history structure
Each history entry is structured:
- `page: number`
- `timestamp: number`
- optional `sessionDuration`
- optional `highlightId`

### Durability/quality controls
- Backward compatibility for legacy page history formats (number-only entries).
- Trims to last 100 entries to prevent storage bloat.
- Filters noisy durations (e.g., <=2s) and rejects implausibly long values.
- Preserves highlight bookmark intent by inheriting prior same-page `highlightId` when needed.

---

## D. Cross-feature durable indexing

The plugin maintains a synced index mapping PDF -> known rem IDs:
- Key pattern: `known_pdf_rems_<pdfRemId>`
- Accessors: `getKnownPdfRemsKey`, `registerRemsAsPdfKnown`

This allows recovery/discovery across sessions when local scope searches or session cache are insufficient.

---

## E. Shield history (KB-aware nested model + migration)

Shield history is written to synced storage using nested objects keyed by KB ID (and optionally by scope/document), then by date.

Patterns:
- KB-level: `history[kbId][date] = entry`
- Scoped: `history[kbId][scopeKey][date] = entry`

Important persistence traits:
- Legacy format detection.
- Primary-KB gated migrations from legacy root-date keys into KB-aware shape.
- Non-primary KB avoids destructive migration assumptions.

This is a robust example of **schema evolution with conditional migration**.

---

## F. Settings (durable configuration)

Settings are registered with typed defaults and later read through `plugin.settings.getSetting(...)`.

Examples include scheduler behavior, default priorities, UI visibility toggles, performance mode, and platform safety preferences.

Settings persistence is used to influence startup behavior (e.g., CSS registration, preload mode).

---

## G. Session orchestration state (ephemeral but structured)

Session storage holds workflow/control state such as:
- queue lifecycle flags (`incremental-queue-active`, etc.)
- seen-item tracking
- popup contexts
- pending operation queues/signals (`pendingPrioritySave`, refresh keys)
- plugin operation flags for guarding reactive listeners (`plugin_operation_active`, `plugin_updating_srs_data`)

Helper `resetQueueSession` shows explicit lifecycle cleanup strategy.

---

## 4) Replication guide for your own RemNote plugin

Use this exact architecture if you want reliable persistence:

1. **Define your data domains first**
   - Entity-bound truth (store on Rem powerup slots)
   - Cross-entity durable state (store in `setSynced`)
   - Runtime/UI state (store in `setSession`)

2. **Centralize key constants**
   - Keep key IDs in one `consts.ts` file.
   - Use namespaced key factories for composite identity (`feature_<entityA>_<entityB>`).

3. **Use typed schemas for persisted payloads**
   - Add Zod (or equivalent) parsing for all JSON payloads read from slots/storage.
   - Reject/repair invalid payloads early.

4. **Implement read/write adapters per domain**
   - E.g., `getThing`, `setThing`, `addThingHistoryEntry`.
   - Hide key formatting and migration logic inside adapters.

5. **Maintain a session cache projection**
   - Build a cache from durable state at startup or on demand.
   - Keep it in sync with small incremental updates.
   - Add explicit “reload trigger” keys for reactive refresh.

6. **Bound untrusted growth**
   - Cap arrays (like history length).
   - Validate timings/ranges.
   - Deduplicate IDs using sets before writes.

7. **Add migration-aware loaders**
   - Detect legacy shapes.
   - Migrate only in safe contexts.
   - Keep migration idempotent.

8. **Guard against reactive race conditions**
   - Use session flags to mark plugin-initiated writes.
   - Debounce or queue writes where rapid updates occur.

9. **Separate UX state from durable truth**
   - Popups and transient interactions should read/write session keys.
   - Final outcomes should commit to slots/synced.

10. **Document key contracts**
   - For each key: owner, data type, lifecycle, retention policy, and migration version.

---

## 5) Minimal implementation blueprint

```ts
// 1) keys.ts
export const thingCacheKey = 'thing-cache'; // session
export const getThingHistoryKey = (thingId: string) => `thing_history_${thingId}`; // synced

// 2) schema.ts
const ThingEvent = z.object({ ts: z.number(), action: z.string() });
const Thing = z.object({ id: z.string(), score: z.number(), history: z.array(ThingEvent) });

// 3) durable writes (slot/synced)
await rem.setPowerupProperty('myPowerup', 'score', [score.toString()]);
await plugin.storage.setSynced(getThingHistoryKey(thingId), trimmedHistory);

// 4) session projection
await plugin.storage.setSession(thingCacheKey, projectedThings);

// 5) lifecycle reset
await plugin.storage.setSession('my_feature_active', false);
```

---

## 6) Why this plugin’s persistence is strong

- It uses **entity-native persistence** for scheduling truth (powerup slots).
- It uses **durable key-value storage** for orthogonal feature state and indexes.
- It uses **session caches** for performance and responsiveness.
- It includes **schema validation**, **format migration**, **deduplication**, and **bloat limits**.
- It includes **operational flags** to avoid false event triggers during plugin-originated writes.

This is a solid production-grade pattern for RemNote plugin persistence.
