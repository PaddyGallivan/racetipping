# Horse Race Tipping — RESUME-HERE

**Live:** https://horseracetipping.com  
**Worker:** `racetipping-api` (Cloudflare)  
**D1:** `racetipping-db` — UUID `42d14cda-3b37-46fe-8f2c-e6d1aea7d0c3`  
**Admin PIN:** `1016` (stored in `organisations.settings.adminPin` for slug `family`)

## Stack
- Cloudflare Worker (`racetipping-api`) serving all routes
- D1 database (`racetipping-db`) — SQLite
- Single-page punter UI served from Worker (no separate frontend)
- Admin panel at `/family` (PIN-gated)

## Org: Gallivan Family
- Slug: `family`
- Admin PIN: `1016`
- Admin email: `pgallivan@outlook.com`
- Race day: "Gallivan Cup — May 2026" (id=2, is_active=1)
- Venues: HAW (Hawkesbury, 10 races, venue_id=3) + EAG (Eagle Farm, 9 races, venue_id=4)
- Horses loaded for all 19 races ✅

## Known Issues / History
- **validateSession bug fixed 2026-05-03**: Worker was doing `SELECT id FROM org_sessions` — no `id` column exists. Fixed to `SELECT token`. Always include D1 binding in deploy metadata or it drops silently.
- **TAB API blocked from CF edge**: `doScrapeResultsForRD` (which calls TAB from within the Worker) gets blocked by CF-to-CF WAF. Use external scrape → POST /results pattern instead.
- **TAB results expire**: TAB beta API drops completed meeting data the next day — scrape must run same day.

## How to enter results manually
```bash
# 1. Get admin token
TOKEN=$(curl -s -X POST https://horseracetipping.com/api/family/auth/login \
  -H "Content-Type: application/json" -d '{"pin":"1016"}' | python3 -c "import sys,json; print(json.load(sys.stdin)['token'])")

# 2. POST result for each race
curl -X POST https://horseracetipping.com/api/family/results \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"race_id":21,"pos1_barrier":1,"pos1_win_odds":5.0,"pos1_place_odds":2.0,"pos2_barrier":2,"pos2_place_odds":1.8,"pos3_barrier":3,"pos3_place_odds":1.5,"pos4_barrier":4}'
```

Race IDs: HAW R1-R10 = ids 21-30, EAG R1-R9 = ids 31-39.

## Deploy
Worker is deployed via CF API multipart PUT. **Must include D1 binding in metadata:**
```json
{"main_module":"worker.js","bindings":[{"name":"DB","type":"d1","id":"42d14cda-3b37-46fe-8f2c-e6d1aea7d0c3"}]}
```

## Next actions
- Stripe: set up product page for pub signups (STRIPE_SECRET_KEY already in Worker env)
- Register `LuckDragonAsgard/racetipping` when token has org scope
- Set up result scraping via external cron (not Worker-internal) for future race days
