# Root Financial — Advisory Headcount Planner

## What this is
A single-file web app for tracking and planning the advisory team headcount at Root Financial Partners. It shows current team structure across pods (Banyan, Aspen, Oak, Canopy, Vista), supports a projected view with change events, and surfaces hiring signals automatically.

**Live URL:** https://dtthor.github.io/root-headcount/  
**GitHub repo:** https://github.com/dtthor/root-headcount  
**Supabase project:** https://tesptjkkoaiahfddxmhi.supabase.co  

---

## Files
- `headcount-planner.html` — development copy (edit this one)
- `index.html` — deployment copy, always kept in sync with `headcount-planner.html` via `cp`
- `CLAUDE.md` — this file

**After every change:** `cp headcount-planner.html index.html` then push to GitHub.

---

## Tech stack
- **Frontend:** Single-file HTML/CSS/JS (no build step, no framework)
- **Backend:** Supabase Postgres — single JSON blob in `headcount_state` table, row `id='main'`, column `data`
- **Auth:** Supabase Google OAuth, restricted to `@rootfinancialpartners.com` accounts
- **Realtime:** Supabase Realtime subscription for live multi-tab sync
- **Hosting:** GitHub Pages (auto-deploys from `main` branch)

---

## Architecture

### State
```js
state = {
  people: [...],        // all people (base, unmodified by events)
  events: [...],        // change events (promotions, hires, departures, team moves)
  photoOverrides: {},   // { personId: base64JPEG } — custom photos
  lastSaved: null       // ISO timestamp, used for realtime conflict detection
}
```

### Key functions
- `getVisiblePeople()` — applies events up to the current cutoff date; returns the "as of now" or "as of projected date" view of people
- `computeNeededNext(allPeople)` — returns hiring signal triggers (see Upcoming tab)
- `migrateState(s)` — data migrations run on every load
- `loadState()` — loads from Supabase, falls back to localStorage, then INITIAL_PEOPLE
- `saveState()` — saves to Supabase and localStorage
- `render()` — full re-render of current tab
- `subscribeToRealtime()` / `unsubscribeRealtime()` — manages the Supabase realtime channel

### Auth flow
- `applyAuthUser(user)` — UI updates only (user pill, show app). Safe inside `onAuthStateChange`.
- `loadAndRender()` — loads data + renders. **Never call from inside `onAuthStateChange`** (causes Supabase internal deadlock).
- `getSession()` is the reliable trigger for initial data load on page open.
- `SIGNED_IN` event (re-login) defers `loadAndRender()` via `setTimeout(..., 0)`.

---

## Roles & structure
| Role | Description |
|------|-------------|
| SA   | Senior Advisor — pod leader |
| LFA  | Lead Financial Advisor |
| FA   | Financial Advisor |
| AFA2 | Associate Financial Advisor 2 |
| AFA1 | Associate Financial Advisor 1 |
| CSC  | Client Service Coordinator |
| CSA  | Client Service Associate |

**Support relationships:** stored as `supportsIds` array on each person. SA/LFA/FA have empty `supportsIds` (they're the ones supported).

---

## Features

### Views
- **Current** — team as of today
- **Projected** — team as of a chosen future date (applies upcoming events)

### Tabs
- **Overview / Upcoming** — Overview shows pod stats; Upcoming shows hiring signals
- **Teams** — pod-by-pod breakdown with support relationships
- **Changes** — timeline of all planned change events

### Change event types
- `promotion` — role change (with criteria tracking per role)
- `new_hire` — incoming person (stored on event, not in people array until start date)
- `departure` — person leaving
- `team_move` — person changing pods

### Upcoming (hiring signal triggers)
1. No CSCs on the team
2. CSC/CSA supporting more than 4 pods
3. AFA leaving a pod (promotion out, team move, departure) leaving an LFA without AFA support
4. LFA with ≥$50M AUM and no AFA support

### Promotion criteria
Each role transition has tracked criteria checkboxes stored on the person object under `criteria`.  
**AFA2 → FA** has 6 criteria: CFP exam, 12+ months, Mock Sequoia Assessment, SA sign-off, Client commitment, Investment Capstone.

### Photos
- Stored as base64 JPEG in `state.photoOverrides[personId]`
- Center-cropped to 128×128 at 0.88 quality via canvas
- `pendingPhoto` pattern: photo changes are buffered locally and only written to state on Save

---

## Supabase setup notes
- RLS is **disabled** on `headcount_state` — auth is handled at the app level
- Google OAuth is enabled in the Supabase dashboard
- Redirect URLs must include `https://dtthor.github.io/root-headcount/`
- Google Cloud Console OAuth client must have `https://tesptjkkoaiahfddxmhi.supabase.co/auth/v1/callback` as an authorized redirect URI

---

## Known patterns / gotchas
- **Never call `loadState()` inside `onAuthStateChange` callback** — causes a Supabase internal deadlock where the API call never resolves
- **`openPersonModal`** reads from `getVisiblePeople()` not `state.people`, so it shows post-event state (e.g. Cassie after her promotion)
- **Incoming people** (new_hire events with future dates) have full edit parity with regular people — criteria and AUM stored on the event object itself
- **`DOMContentLoaded render()` was removed** — render only fires after auth + data load completes
- The `index.html` deployment copy must always be kept in sync manually

---

## Pending / possible future work
- Remove debug `[auth]` and `[loadState]` console.log statements once auth is stable
- Consider adding role for the Growth team (currently separate from advisory)
- AUM thresholds: LFA triggers at $50M
