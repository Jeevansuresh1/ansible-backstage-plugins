# Collections Page — Per-Page Pagination Optimization

## Problem

The Collections page previously loaded **ALL** collection entities (~2939 entities, ~58MB) into a client-side `PaginatedEntityCache` on every page visit. The heavy field `spec.collection_readme_html` (~20KB per entity) was included even though `CollectionCard` never uses it. The cache expired every 5 minutes, re-triggering the full fetch. On top of that, `EntityListProvider` (a Backstage context wrapper) performed a **second** background bulk fetch of all matching entities — doubling the problem.

## Solution

Two-mode per-page fetching that loads only the current page's data from the Backstage catalog API.

### Mode A — `showLatestOnly` OFF (direct per-page)

True server-side pagination. **ONE** `queryEntities` call per page (~6KB):

```
catalogApi.queryEntities({
  filter: { kind: 'Component', 'spec.type': 'ansible-collection', ...activeFilters },
  fields: LIGHTWEIGHT_FIELDS,    // 12 fields, excludes readme
  limit: 12,                     // PAGE_SIZE
  offset: (currentPage - 1) * 12,
  orderFields: [{ field: 'metadata.name', order: 'asc' }],
  fullTextFilter: searchQuery ? { term: searchQuery } : undefined,
})
```

`totalItems` from the response gives the exact count for pagination.

### Mode B — `showLatestOnly` ON (lightweight index + per-page fetch)

Dedup requires knowing all collection names+versions to find the latest. But we don't need full entity data for that.

**Step 1 — Lightweight dedup index** (~1-2MB, cached 5 min):

```
catalogApi.queryEntities({
  filter: { kind: 'Component', 'spec.type': 'ansible-collection', ...activeFilters },
  fields: ['metadata.name', 'metadata.annotations', 'spec.collection_full_name', 'spec.collection_version'],
  limit: 5000,
  orderFields: [{ field: 'metadata.name', order: 'asc' }],
})
```

Only 4 tiny fields per entity. From this:
- Groups by `fullName::sourceId`, keeps the `metadata.name` of the highest version per group
- Result: ordered list of unique collection entity names
- `length` → exact unique count → exact page count
- Cached in-memory with 5-min TTL, keyed by `sourceFilter::tagFilter::searchQuery`

**Step 2 — Fetch 12 entities for current page** (~6KB):

```
catalogApi.queryEntities({
  filter: { kind: 'Component', 'spec.type': 'ansible-collection', 'metadata.name': pageNames },
  fields: LIGHTWEIGHT_FIELDS,
  limit: 12,
})
```

Fetches exactly the 12 entities needed. Array values in the filter mean OR.

### Performance comparison

| Metric | Before | After (Mode A) | After (Mode B, first load) | After (Mode B, cached index) |
|--------|--------|----------------|---------------------------|------------------------------|
| Data per page load | ~58MB | ~6KB | ~1-2MB + ~6KB | ~6KB |
| API calls per page | 7+ sequential | 1 | 2 | 1 |
| Page navigation | instant (client-side) | 1 call (~6KB) | 1 call (~6KB) | 1 call (~6KB) |

### Filter dropdowns via `getEntityFacets`

Filter options (sources, tags) are populated using `getEntityFacets` instead of loading all entities:

```
// Tags
catalogApi.getEntityFacets({
  filter: { kind: 'Component', 'spec.type': 'ansible-collection' },
  facets: ['metadata.tags'],
})

// PAH sources
catalogApi.getEntityFacets({
  filter: { ...BASE_FILTER, 'metadata.annotations.ansible.io/collection-source': 'pah' },
  facets: ['metadata.annotations.ansible.io/collection-source-repository'],
})

// SCM sources
catalogApi.getEntityFacets({
  filter: { ...BASE_FILTER, 'metadata.annotations.ansible.io/collection-source': 'scm' },
  facets: ['metadata.annotations.ansible.io/scm-host-name'],
})
```

Returns just facet values + counts (no entity data). A separate `queryEntities` with `limit: 1` fetches `totalItems` for the unfiltered total count (`totalUnfilteredCount`).

---

## Files Changed

### New file: `collectionsInvalidation.ts`

Simple callback module (~17 lines) replacing direct `collectionsCache` import. The hook registers its refresh function on mount; `syncPollingService` calls `invalidateCollections()` when sync completes.

```
setCollectionsInvalidateCallback(cb)   — hook registers its refresh
clearCollectionsInvalidateCallback()   — hook cleans up on unmount
invalidateCollections()                — triggers refresh from sync service
```

### Rewritten: `usePaginatedCollections.ts`

Complete rewrite of the hook internals. Same exported interface (`UsePaginatedCollectionsResult`).

**Key internals:**
- `LIGHTWEIGHT_FIELDS` — 12 fields for card rendering (excludes `spec.collection_readme_html`)
- `DEDUP_INDEX_FIELDS` — 4 fields for the Mode B dedup index
- `INDEX_CACHE_TTL_MS` — 5 minutes
- `buildEntityFilter()` — constructs server-side filter from source/tag selections
- `buildRepoFilter()` — constructs filter for repository detail page view
- `dedupIndexEntities()` — groups by `fullName::sourceId`, keeps highest version per group
- `fetchFacets()` — populates filter dropdowns + unfiltered total count
- `fetchSyncStatus()` — fetches sync status from backend
- `fetchPage(page)` — the main fetch function, handles all 3 paths:
  - Repository detail: fetch all matching, client-side paginate (small subset)
  - Mode A: single `queryEntities` with offset/limit
  - Mode B: dedup index (cached) + page fetch by name array
- `fetchGenRef` — generation counter preventing stale async responses from overwriting newer state
- Invalidation callback registered on mount, cleared on unmount

### Modified: `CollectionsListPage.tsx`

**Removed:**
- `EntityListProvider` wrapper — was causing a ~58MB background bulk fetch
- `CollectionsTypeFilter` component — seeded `EntityListProvider` with kind/type filters
- `UserListPicker` from `@backstage/plugin-catalog-react` — required `EntityListProvider`
- `useEntityList` hook — only consumed `filters.user?.value === 'starred'`

**Added:**
- Custom starred toggle using MUI `List`/`ListItem` — replaces `UserListPicker`
- Local `userFilter` state (`'all' | 'starred'`) — replaces `useEntityList().filters.user`
- Debug snackbar — shows "Page X fetched from catalog — Y of Z total" after each fetch
- Updated `collectionsTitleCountSuffix` — shows `(filtered of total)` when counts differ:
  - `(44)` — no filters active, counts match
  - `(44 of 235)` — filters/dedup reduce the count
  - `(0 of 235)` — search/filters match nothing

### Modified: `CollectionsListPage.test.tsx`

**Removed:**
- `EntityListProvider` wrapper from render helpers
- `EntityKindFilter`, `EntityTypeFilter` imports
- `UserListPicker` mock
- `collectionsCache.clear()` calls
- `CollectionsTypeFilter` describe block (3 tests)
- `useEntityList` spy-based starred filter tests

**Added/Updated:**
- `getEntityFacets` mock returning PAH source facets
- Never-resolving `getEntityFacets` mock in loading test (prevents hang)
- New starred filter tests using `fireEvent.click(screen.getByTestId('starred-filter'))`
- Search test updated: mock returns empty results when `fullTextFilter.term` is present
- `CatalogFilterLayout` describe block

### Modified: `syncPollingService.ts`

- Changed `import { collectionsCache } from '../CollectionsCatalog/collectionsCache'` → `import { invalidateCollections } from '../CollectionsCatalog/collectionsInvalidation'`
- Changed `collectionsCache.invalidateFetchedData()` → `invalidateCollections()`

### Modified: `syncPollingService.test.ts`

- Updated mock from `collectionsCache` to `collectionsInvalidation`
- Updated assertion from `collectionsCache.invalidateFetchedData` to `invalidateCollections`

### Modified: `index.ts` (barrel exports)

- Removed `collectionsCache` and `CollectionsCacheState` exports
- Added `invalidateCollections`, `setCollectionsInvalidateCallback`, `clearCollectionsInvalidateCallback` exports

### Dead code (not yet deleted): `collectionsCache.ts`, `collectionsCache.test.ts`

Still on disk but no production code imports them. Safe to delete.

---

## How the data flows

```
User loads /self-service/collections
  │
  ├── fetchFacets()  [runs once on mount]
  │   ├── getEntityFacets → tags for dropdown
  │   ├── getEntityFacets → PAH sources for dropdown
  │   ├── getEntityFacets → SCM sources for dropdown
  │   └── queryEntities(limit:1) → totalUnfilteredCount (for "X of Y" display)
  │
  ├── fetchSyncStatus()  [runs once on mount]
  │   └── GET /api/catalog/ansible/sync/status → sync timestamps
  │
  └── fetchPage(1)  [runs on mount + filter/search/page changes]
      │
      ├── [showLatestOnly OFF — Mode A]
      │   └── queryEntities(offset, limit:12, fields:12) → page entities + totalItems
      │
      └── [showLatestOnly ON — Mode B]
          ├── queryEntities(limit:5000, fields:4) → dedup index (cached 5min)
          │   └── dedupIndexEntities() → ordered list of unique entity names
          └── queryEntities(names[page], limit:12, fields:12) → page entities

User clicks next/prev page
  └── fetchPage(newPage) → same flow, index is cached → just 1 API call (~6KB)

Sync completes (via syncPollingService)
  └── invalidateCollections() → clears index cache, re-fetches current page + facets
```

## Key design decisions

1. **Why not just Mode A for everything?** Mode A can't deduplicate across versions — `offset/limit` returns raw catalog rows, so page 1 might show v1.0.0 and v2.0.0 of the same collection. Dedup requires knowing ALL versions to pick the latest.

2. **Why cache the index, not the page data?** Page data is tiny (~6KB) and fetches are fast. The index (~1-2MB) is the expensive call — caching it makes page navigation instant.

3. **Why `getEntityFacets` for dropdowns?** Previously, filter options were derived from ALL loaded entities. With per-page fetching, we only have 12 entities — not enough to populate dropdowns. `getEntityFacets` returns all unique values for a field without loading entities.

4. **Why remove `EntityListProvider`?** It auto-fetches ALL matching entities into a React context. Even with our per-page hook, `EntityListProvider` was making a separate ~58MB fetch in the background. The ONLY data consumed from it was `filters.user?.value === 'starred'` — a simple boolean toggle replaced with local state.

5. **Why `fetchGenRef`?** Prevents race conditions. If the user rapidly clicks next/prev, multiple async fetches fire. Each fetch increments the counter. When a fetch completes, it checks if its generation matches the current one — if not, it discards its results (a newer fetch is in progress or already completed).

6. **Why `totalUnfilteredCount` separate from `totalCount`?** Distinguishes "no collections exist at all" (show empty state) from "filters match nothing" (show "No collections match your search or filters" + keep filters visible). Also powers the `(44 of 235)` title display.

---

## Test status

- `CollectionsListPage.test.tsx`: **24/24 passing**
- `usePaginatedCollections.test.tsx`: needs verification
- `syncPollingService.test.ts`: needs verification
- TypeScript (`yarn tsc`): **passes**
- Lint: **passes**
