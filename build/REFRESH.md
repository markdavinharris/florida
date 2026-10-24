# Florida Early Vote 2026 dashboard — daily refresh runbook

Published artifact (update in place, never create a new one): **https://claude.ai/artifact/PVQVhDAyQZkUJs1sriArY5** — title "Florida Early Vote 2026".
First published 2026-10-24 with data through 2026-10-22, with the gated "Projected final electorate" panel (PROJECTION.md).
Public site: https://florida.coldspark.com, repo `markdavinharris/florida`, refreshed by its own GitHub Action at
:37 past each hour 7 AM–11 PM ET (`build/AUTOMATION.md`, `claude/early-vote-sites-publish.md`). Claude refresh
tasks must NOT push to the repo. The public page carries no projection panel (the Action refreshes only the data block,
so a committed projection would go stale).

All kit files are in this Project under `claude/florida-ev-dashboard/` and in the repo under `build/`.

## Steps (about 15 minutes)
1. Make a working folder `florida/` in the scratchpad with `data/` and `bq/`. Fetch every file in
   `claude/florida-ev-dashboard/` with `project_read` and write it under its own name (`counties.json` goes in `data/`).
2. Run the queries in SPEC.md with q.py (ASOF = the day number of the last complete day, 12 on 2026-10-24 for data through
   2026-10-22; if unsure, run S 2026 first and take the smallest d whose returns + in-person exceed 500). Save, copy to `bq/`,
   convert with the helpers. Never retype numbers.
3. `python3 build.py` → `dashboard.html` and `index.html`; it prints the as-of date and the projection line. If proj.json is for
   a different day the projection is hidden (by design).
4. Spot-check: `SELECT COUNTIF(Rejected=0 AND ReturnDate BETWEEN '2026-01-01' AND '<asof>') + COUNTIF(EarlyVoted BETWEEN '2026-01-01' AND '<asof>') FROM abev_2026 WHERE State='FL'`
   must equal the "Early votes cast" tile. For a county, compare on `REGEXP_REPLACE(UPPER(Jurisname),' COUNTY$','')`, not on an exact Jurisname.
5. `python3 shot.py index.html out.png` and look at it. Publish with the Artifact tool: `url: https://claude.ai/artifact/PVQVhDAyQZkUJs1sriArY5`,
   `file_path: <dashboard.html>`, `label: "Data through <date>"`, `files: {"site/index.html": {"from": "<abs index.html>", "contentType": "text/html"}}`.
   Read the artifact first if asked. No `icon`. The 67 `precincts/*.json` already on the artifact are kept.
6. If template.html or app.js changed, write them back to the Project. For a design change on the public site, build a copy
   WITHOUT proj.json (`mkdir pub/data`, copy `data/*.json` except proj.json, `python3 build.py pub/data pub/dashboard.html`) and
   commit `pub/index.html` as the repo's `index.html` (add_repo push), then dispatch the workflow with force.
7. Report: as-of date, statewide early votes and vs 2024 / 2022 at the same point, net R−D, projection line, anything odd.

## Data notes
- Party: registered party (`PartyRollUp`: `Republican`, `Democrat`; `Unaffiliated/DTS` and `Independent/Other` go to "NPA / other").
- Early voting must run Oct 24 – Oct 31, 2026 (d=10 to d=3); counties may start on d=15 and most large ones did (Oct 19, 2026; Oct 21, 2024; Oct 24, 2022).
- Rows with a blank DT_Regid have a NULL Jurisname (35,062 in 2026, 163,012 in 2024, 97,519 in 2022): kept statewide as
  UNKNOWN, dropped from the county file, so build.py's county-sum WARNING for every year is expected (that difference only).
- 2026-10-22 (d=12, first build 2026-10-24): 1,982,511 early votes, 1,266,286 outstanding mail ballots (Miami-Dade 187,007 early
  votes); 3,280,017 ABEV rows, 67,454 rejected, 35,125 not matched to a voter.
- Requests made more than 60 days out are dated 2026-09-04 (the cap); most FL requests sit on that date.
- County spellings: `MIAMI DADE`→`MIAMI-DADE` (voter file), `DE SOTO`→`DESOTO` (feed and voter file). St. Johns / St. Lucie have no period anywhere (ST JOHNS, ST LUCIE).. build.py asserts exactly 67 counties.
- Precinct maps: NYT 2024 precinct map (`FL-precincts-with-results.geojson.gz`) matched to `voters.PrecinctCode` by top-8 census blocks (Query B2). 2026-10-24: 5,707 of 5,710 codes placed, 94 weak, 240 shapes shared. 67 files, 2.6 MB.
- Projection on the first build: early −1.1 → projected +15.0 (range +5.8 to +17.9) as of 2026-10-22 (d=12). Weakest backtest of any state: train 2022→predict 2024 model 5.8 vs raw-early −1.1 (raw was closer), 2024→2022 −8.2 vs −11.9. Florida's mail vote is large and heavily Democratic early, Republicans vote late in person; treat the range as the message, not the point.
