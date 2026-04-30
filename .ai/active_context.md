# Screen Docent — Active Context

> **Last Updated:** 2026-04-04

---

## Current State: Phase 5.5 — Discovery Pipeline Tuned ✅

The system is **stable and running locally**. The multi-museum discovery pipeline has been overhauled with per-API search parameter optimisation, progressive fallback strategies, and quality filtering.

| Milestone | Status |
|-----------|--------|
| Autonomous RAG Curator (Wikipedia + Gemini enrichment) | ✅ Complete |
| Modular Semantic Art Scout (6 active museum APIs) | ✅ Complete |
| Query Classifier (dictionary + Gemini Flash hybrid) | ✅ Complete |
| Result Ranker (scoring + deduplication) | ✅ Complete |
| Discovery Queue with approve/reject workflow | ✅ Complete |
| Paginated "Load More" search sessions | ✅ Complete |
| Admin utilities: Factory Reset, Clear Pending | ✅ Complete |
| Tier-2 API Key management (Europeana) | ✅ Complete |
| TTFB bottleneck resolved (4 Uvicorn workers + StaticFiles + Cache-Control) | ✅ Complete |
| Docker zero-touch deployment to MS-01 | ✅ Complete |
| Tiered cache strategy (immutable media / no-cache API / short-lived code assets) | ✅ Complete |

---

## Recently Completed: Discovery Scout API Tuning (2026-04-04)

Optimised search parameters for all 6 active museum APIs:

| Museum | API Strategy | Van Gogh Results |
|--------|-------------|-----------------|
| **Chicago Art Institute** | `q=` with artist boosting | ~5 paintings |
| **Met Museum** | Canonical name + fallback without `artistOrCulture` | ~10 works |
| **Cleveland Museum** | `artists=` dedicated param | ~5 paintings |
| **Rijksmuseum** | `creator=` + `imageAvailable=true` | 11 artworks |
| **SMK** | `keys=` canonical name | 0 (collection limitation) |
| **Europeana** | Progressive `who:` → text fallback | 16 works |

Also implemented:
- **Background task error logging** — crashes now surface instead of silent swallowing
- **API no-cache headers** — `/api/*` always returns fresh data
- **Thumbnail cache-buster** — `?_cb={id}` prevents stale images in discovery grid

---

## Active Goal: Phase 6 — The Autonomous Director

Phase 6 introduces an autonomous curation intelligence that observes how artwork is displayed and evolves the rotation based on learned preferences. The Director will:

- Track per-artwork **telemetry** (how long each piece is displayed, how often it's skipped, user interactions with the placard).
- Compute an **affinity score** per artwork that influences shuffle weighting.
- Autonomously promote high-affinity pieces and demote low-engagement ones within playlists.

---

## Recently Completed: Aggressive Cache-Busting for UI (2026-04-29)

Resolved a persistent issue where thumbnails in the UI showed incorrect images after a factory reset.
- Added explicit cache-busters (`?f=filename.jpg`) to all Review Queue thumbnail requests.
- Updated `app.py` middleware to explicitly return `no-cache` for HTML files so users never get stuck on cached JS.

---

## Next Immediate Steps

> [!IMPORTANT]
> The Phase 6 Database expansion, Bag Shuffle statefulness, and Frontend Telemetry wiring are fully complete.

### Step 1: Director Agent Optimization
The foundational telemetry loop is active. The backend is receiving `total_display_time`, `skip_count`, and updating the `affinity_score`. 
The `affinity_score` is already actively weighting the "Bag Shuffle" probabilities for playlist selection. 
We need to monitor and potentially refine the (V1) math in the heartbeat endpoint to ensure affinity doesn't scale unbounded or penalize items unfairly.

---

## Open Questions

1. **Premium APIs:** When to integrate Harvard Art Museums and Smithsonian? They require separate API keys and rate limiting strategies.
2. **Affinity Decay:** Should the affinity score decay over time so that old favourites don't permanently dominate the rotation?
3. **Skip Signal Strength:** Is a manual skip (prev/next button) a stronger negative signal than simply letting the timer cycle naturally?
4. **Per-Display Affinity:** Should affinity scores be global or per-display?
