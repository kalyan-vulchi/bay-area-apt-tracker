# Bay Area Apartment Tracker

A single-file web app for tracking 1-bedroom apartments during a South Bay
apartment hunt. Scores each listing against a fixed set of must-haves, plots
price history, maps them, and surfaces the best candidates.

- **Live:** https://kalyan-vulchi.github.io/bay-area-apt-tracker/
- **Repo:** https://github.com/kalyan-vulchi/bay-area-apt-tracker
- **Everything lives in one file:** `index.html` (HTML + CSS + JS inline).
- **Hosting:** GitHub Pages off `main`. Any push to `main` auto-deploys.

> This README is the cold-start context. If you're picking this up in a new
> session, read it top to bottom before touching anything.

---

## 1. The hunt (search criteria)

- **1 BR / 1 BA**, South Bay (Sunnyvale / Santa Clara / Cupertino / San Jose).
- **Budget ≤ $3,300/mo**, measured on **total price = base rent + required parking** (see §4).
- **Commute ≤ 30 min** to the office at **1375 Crossman Ave, Sunnyvale** (`37.4141032, -122.0103658`).
- **Must-haves:** in-unit W/D, AC (any type counts, incl. window).
- **Preferred:** hardwood / wood-style floors.
- **Parking** is not a hard requirement, but a *required* parking fee is added into the total price.

A listing **qualifies** when it has W/D **and** AC **and** total ≤ $3,300 **and** commute ≤ 30.

---

## 2. Architecture

- **Single `index.html`.** Tailwind (CDN), Leaflet (map, CDN), Chart.js (trends, CDN). No build step, no framework.
- **Data lives in the browser** via `localStorage` key `kalyan_bay_apts_v2`. The user's browser is the source of truth for their edits.
- **Seed data is in the code:** `SHORTLIST` (the tracked listings) and `CORPUS_LIST` (a "web search" bucket, currently just Avalon). `PRICE_HISTORY_SEED` seeds the trends chart from git snapshots.
- **Views (tabs):** Summary, List, Split (list+map), Map, Trends.

### Merge / sync model (how code ⇄ browser stay in sync)
On every load, `importList()` and `importPrices()` reconcile the code seed with the browser's stored data:

- **`IMPORT_FIELDS`** — force-synced from code → browser on **every load** (code always wins):
  `address, commute, commute_off, dist_mi, has_wd, has_ac, ac_type, has_parking, has_hardwood, notes, scrapable, impression, parking_fee`.
- **`PRICE_FIELDS`** — **version-gated**; applied only when code's `PRICE_VERSION` is **greater** than the browser's stored version:
  `rent, sqft, available, plans, promo, url`.
- **`PRICE_VERSION`** (currently **53**) — an integer. **Bump it by 1 whenever you change any price/promo/url/plans in the code**, or browsers won't pick the change up.
- **`status`** and **`unavailable`** are **browser-owned**: seeded on new entries, never force-synced, so in-browser toggles persist. `status` only ever advances along a funnel (`not_visited → toured …`), never regresses.
- **Scrapable listings** (see §6) additionally **force-sync their price fields from code every load** — so their prices always mirror the code.

### New listings
`importList()`'s else-branch seeds a brand-new apartment (one not already in `localStorage`) with an explicit object literal. **If you add a new field to the data model, add it to that literal too**, or fresh browsers won't get it (existing browsers pick it up via `IMPORT_FIELDS`).

---

## 3. Data model (per apartment)

```
name, address, url, lat, lng,
rent            // BASE monthly rent (integer). Trends tracks this.
parking_fee     // required first-spot parking cost; 0 when included (see §4)
sqft, available // available = earliest move-in date "YYYY-MM-DD" or null
commute, commute_off, dist_mi   // peak / off-peak minutes, miles (see §7)
has_wd, has_ac, ac_type, has_parking, has_hardwood   // booleans + ac_type string
status          // 'not_visited' | 'toured'  (browser-owned)
impression      // tour rating 0-10 (only meaningful once toured)
notes           // freeform; tour findings live here
plans           // array of {label, sqft, rent, available} floor plans
promo           // active promotion string or null
scrapable       // true for the 6 cron-managed listings (see §6)
unavailable     // true = visual grey-out only (see §5). browser-owned.
price_history    // [{date, rent}] — drives the Trends chart
```

`ac_type` values: `central` (15 pts), `split_both` (12), `split_living` (10), `window` (8), `unknown`/absent → 8.

---

## 4. Total price = base rent + parking

`totalRent(a) = a.rent + parking_fee`. **This total drives display, sorting,
budget filters, scoring/qualification, averages, and the avg-by-group chart.**
The **Trends chart still plots base rent** (parking is flat, so base rent is the
meaningful trend).

Cards show the total with a `"$rent + $fee parking"` breakdown line; map popups
show the total with `(incl. $X pkg)`.

**Current parking fees** (first required spot; 0 = free/included):

| Listing | parking_fee |
|---|---|
| River Terrace | $95 |
| City Gate at Cupertino | $50 |
| Domicilio | $50 (cheaper uncovered; covered is $125) |
| Estancia at Santa Clara | $25 |
| Mission Pointe by Windsor | $10 (open first-come; reserved $40) |
| everything else | $0 |

Rule going forward: set `parking_fee` to the **cheapest required** amount when the
first spot costs money; `0` when the first/assigned spot is free/included.
`parking_fee` is an `IMPORT_FIELD` (force-synced), so no `PRICE_VERSION` bump needed.

---

## 5. Available / Unavailable toggle

Each card has a green **"✓ Available"** / grey **"✕ Unavailable"** toggle
(`toggleAvailable()`), field `unavailable`.

**Unavailable is purely a visual grey-out** (dimmed card + "Unavailable" badge).
Score, qualification, and price are **still computed from the latest known price**
— availability does **not** change the score. (Earlier iterations blanked the
price to N/A and dropped it from scoring; that was reverted per the user's call:
"calculate the score based on latest price; greying out is the cleanest visual.")

- Trends chart **still shows** an unavailable listing's history for the dates it
  was available — drawn as a **dashed line** labeled `"(unavail.)"` that **stops
  at its last real data point** (no flat projection to today). It stays in the
  checkbox list.
- `unavailable` is browser-owned (persists across reloads); sync it code-ward
  from the export like `status`.

---

## 6. The daily cron + the 6 "scrapable" listings

Six listings are `scrapable:true` and auto-priced by a **daily cloud routine**:
**Cezanne, Apricot Pit, The Podium, River Terrace, Mission Pointe by Windsor,
Avalon Silicon Valley.**

- The routine runs **daily at ~5 AM Pacific (12:00 UTC)**, WebFetches each
  property's **official** page, updates `rent`/`available`/`promo` for those 6 in
  `index.html`, and commits straight to `main` (auto-deploys). Commits are titled
  `Auto price refresh YYYY-MM-DD`.
- It reads the **lowest listed 1BR asking rent** from the official site only —
  **never aggregators** (Apartments.com/Zillow/etc. are stale/wrong).
- Because these force-sync from code, the browser always mirrors the code value.
- Routine ID `trig_01U3rYJaezFd8KtUg9JLaCe1` (managed at
  `https://claude.ai/code/routines/trig_01U3rYJaezFd8KtUg9JLaCe1`). It holds a
  GitHub PAT **in the routine config, not in this repo** — revoke it when the hunt
  is done. Manage/trigger it with the `RemoteTrigger` tool.
- **Keep the cron ON** — the user wants it as a fallback for days they don't
  export. It has stalled before (cloud-env network allowlist blocking the property
  domains, Jun 10–11 & Jul 10–13); if it fires but stops committing for 2+ days,
  the cause is the env network, not the PAT or the pages (verify by WebFetching
  the pages yourself and testing the PAT).
- No standing overrides currently — all 6 match their official lowest 1BR. (A
  brief Mission Pointe "$3,507" was a misread and was corrected to $3,327.) If the
  user ever wants a specific unit's price to survive the cron, remove that listing
  from the cron's prompt scope rather than fighting it daily.

---

## 7. Commute methodology

`commute` (peak) = **worst of AM & PM** =
`max(AM arrive-office-Thu-9am, PM leave-office-Thu-4:45pm)`, pulled from the
**TomTom Routing API** (`calculateRoute`, `traffic=true`, arriveAt/departAt a
future Thursday). `commute_off` = free-flow (no-traffic) time. Do **not**
hand-estimate from straight-line distance. `commute*`/`dist_mi` are `IMPORT_FIELDS`
(no `PRICE_VERSION` bump). The user provides a TomTom API key per session — **do
not commit it**.

---

## 8. Daily sync workflow (the core recurring task)

The user exports from the app's **"Sync Data"** button (a JSON named
`apartments-data-YYYY-MM-DD.json` in `~/Downloads`) and shares the path. Then:

1. `git pull --rebase origin main` (the cron may have pushed overnight).
2. **Diff** the export against the code seed (parse `SHORTLIST`+`CORPUS_LIST`; see §10).
3. **Apply every rent change** — including the 6 scrapable listings (policy as of
   Jul 2026: the user's export is authoritative for all prices; previously
   scrapable were skipped, which caused their edits to revert).
4. Also apply `url` / `promo` / `parking_fee` / `unavailable` changes when present.
5. **Bump `PRICE_VERSION`** by 1.
6. **Guard against bad data:** a single-day move ≥ ~$400, or a price *below* a
   listing's known floor, is suspect → verify against the **official site** via
   WebFetch; if you can't confirm, **hold it and ask the user** rather than
   propagate it. (Several such holds happened; some were transient bad reads that
   self-corrected next day.)
7. Commit (`Sync <date> export: N price changes; PV <n>`), pull --rebase, push.

**Data rules (do not violate):**
- Only use prices from the **property's own site** or what the **user says** — never aggregators.
- Store the **listed asking rent**, never the effective-after-promo rent.
- Set `has_*` amenities **only from a tour or a direct leasing-office confirmation**; `null` when unknown; trust the user's tour notes over online info.

---

## 9. Scoring (`score(apt)`)

Points (max 100): W/D **+30**; AC by type (**central 15 / split_both 12 /
split_living 10 / window 8**, unknown 8); rent graduated ~20→10 up to a $3,350
soft cap then 0; commute graduated 15 (≤10 min) → 0 (30 min); hardwood **+10**;
tour impression **+ up to 10**. `qualifies` = no missing must-have (W/D, AC,
total ≤ $3,300, commute ≤ 30). `isNearMiss()` = fails exactly one must-have (with
budget within +$200) and the rest confirmed.

---

## 10. Dev / verification notes

- **No Node in this environment.** Use `/Users/kalyan/opt/anaconda3/bin/python3`.
- The code is JS, not JSON. To diff/parse the seed arrays, extract `SHORTLIST` /
  `CORPUS_LIST` by bracket-matching and convert JS-object-literal → JSON (quote
  bare keys, single→double quotes handling `\'`, strip trailing commas). This
  repo's session scripts use exactly that helper.
- **Verify UI changes for real** with headless Chrome (there is no test suite):
  `"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" --headless=new
  --disable-gpu --no-sandbox --dump-dom --virtual-time-budget=9000 file://<path>`
  Inject a `window.load` + `setTimeout(...)` probe that calls the render functions
  directly (`renderTrends()`, `renderSummary()`, `applyFilters()`) and writes a
  result to `document.title` — `setView()` renders async (50 ms), so reading chart
  state right after it is a race. `--screenshot=out.png` for layout checks.
- After any edit, sanity-check bracket balance:
  `python3 -c "h=open('index.html').read();assert all(h.count(a)==h.count(b) for a,b in [('{','}'),('[',']'),('(',')')])"`.

---

## 11. Current state (as of late July 2026)

- **42 listings** (41 shortlist + Avalon corpus). **`PRICE_VERSION` 53.**
- Toured many; recent tours: Park Central, Brookdale, Sage at Cupertino (Sage's
  tour corrected W/D→true and AC→window, making it qualify).
- Qualifying set is small and shifts daily with prices; near-budget listings
  (e.g. Montclaire, Verdant, Magnolia, Oak & Umber, Mansion Grove) oscillate over/
  under $3,300 frequently.
- Features shipped: total-price-with-parking, available/unavailable grey-out,
  Trends average line + avg-rent-by-group chart, bottom-aligned card actions,
  TomTom worst-of-AM/PM commutes.

---

## 12. Quick reference

| I want to… | Do this |
|---|---|
| Push new prices to all browsers | change `rent`/`promo`/`url` in code **and bump `PRICE_VERSION`** |
| Push amenity/notes/commute/parking change | just edit the field (it's an `IMPORT_FIELD`, force-synced) |
| Add a new listing | add to `SHORTLIST`; also add any new field to the seed literal in `importList()` |
| Add a parking fee | set `parking_fee` (cheapest required first-spot cost) |
| Mark a listing unavailable | it's a browser toggle; sync the `unavailable` flag from the export |
| Re-price the 6 scrapable listings | let the cron do it, or apply the user's export values (bump `PRICE_VERSION`) |
| Fix a commute | pull TomTom worst-of-AM/PM (§7) |
