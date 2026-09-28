# Open-sourcing the static airspace layer of Open Drone Space

## Context

`open-drone-space` is a private repo (GitHub, deployed through Lovable, Supabase backend). It
has two data cadences (docs/ARCHITECTURE.md):

- **Occasional** — airspace, aerodromes, No Drone Zones, statutory zones, terrain: built
  offline by `scripts/ingest/*.mjs` from `data/raw/`, committed to `public/data/`.
- **Daily** — HungaroControl Airspace Use Plan + ROMATSA NOTAMs → Postgres; NOTAMs carry
  pilots' mobile numbers (`notam_contacts`).

Goal: a separate **public** repo that holds the occasional layer end to end — raw-input
fetchers, ingest scripts, the published datasets, and an embeddable map — with no backend.
Daily data stays in the private app.

### Assessment of the idea (corrections)

1. **The split is right.** The occasional layer is already backend-free (static files behind a
   CDN), deterministic and reviewable. GitHub (repo + Releases + Pages + jsDelivr) is all the
   "storage" it needs; the whole published set is about 1.1 MB. You don't need a database or Git LFS.
2. **It isn't "fetched once".** It changes occasionally: the AIRAC cycle (every 28 days) for
   OFM/eAIP, new CAA register editions, and decree amendments. Only the terrain grid is truly
   one-off. So the public repo needs a refresh process as well as a place to keep the data.
3. **The static data also contains phone numbers.** `public/data/tables/aerodromes.json`
   (from the CAA aerodrome register) has phones in all 131 rows and e-mails in 100. About 54
   operators look like natural persons. The data is already public, but the GDPR still
   applies to republishing it. And a public repo, its forks, npm and Zenodo can never be
   corrected or erased, unlike the app's tables. The public build drops `operator`,
   `operatorAddress`, `phones` and `emails`. The embed can link to the official register
   instead. The zone GeoJSON is already clean: it only carries `register_id`.
4. **The OFM licence is unresolved.** The manifest says "free for non-commercial use".
   openflightmaps.org/about says the opposite: commercial use is allowed, attribution is
   required, and users must report errors and give their end users a way to report errors.
   The full licence text wasn't found (the package `readme.pdf` only points to the website).
   This decides whether `airspaces`/`aerodromes` can be published, so it's Phase 0.
5. **The embed must stay honest.** A map that shows only static layers looks like "all clear"
   on days with active TRAs or eseti légterek. The freshness invariant has to carry over: the
   embed says in a fixed banner that daily activations are not shown, and links to
   opendrone.space for today's status.

## Dataset verdicts

| Dataset | Cadence | Source / licence | Verdict |
|---|---|---|---|
| `no-drone-zones` | decree amendment | 26/2007, hand transcription; law is outside copyright (Szjt. 1. § (4)) | ✅ highest-value original work |
| `aerodrome-zones` (2 km / 750 m / 3 km) | register edition | derived from register points + 4/1998 | ✅ names/coords only |
| `protected-zones` | rare | 4/1998 + Parliamentary Guard notice | ✅ |
| `rmz-tmz` | AIRAC | eAIP ENR 2.2 transcription (12 features) | ✅ low risk; HungaroControl reuse terms not verified, attribute |
| `terrain-grid` | once | Copernicus GLO-90, redistribution allowed with notice | ✅ carry the notice |
| `airspaces`, `aerodromes`, drop zones | AIRAC | OFM OFMX | ⚠️ publish once Phase 0 confirms; attribution + error-report path |
| `tables/aerodromes` | register edition | CAA register | ⚠️ publish **without** operator/address/phones/emails |
| `tables/restricted-areas` | rare | aviation authority list (organisations only) | ✅ |
| `tables/nature-areas` | AIRAC | from OFM airspaces | ✅ (follows OFM verdict) |
| `tables/fee-accounts` | rare | 14/2015 FM r. | ➖ procedural, not map; stays private |
| OFM chart tiles | AIRAC | separate raster, ~230 MB | ❌ not needed |
| Remark translations (`src/lib/i18n/source-text.json`) | with data | produced via Lovable AI gateway | ✅ ship as committed data; the LLM step stays private |
| Daily AUP, NOTAMs, forms, procedures, rules | — | — | ❌ out of scope (your choice) |

## Plan

### Phase 0 — Legal groundwork (before any public commit)
- Get the full **OFM Data User License**: check the download page of the LH package, or e-mail
  OFMA and ask directly whether derived GeoJSON can be redistributed in a public repo. Fix the
  licence string in `scripts/ingest/parse-ofmx.mjs` / manifest in the private repo.
- **Licences (decided: permissive + credit to the author):**
  - Code: **Apache-2.0** with a `NOTICE` file ("Open Drone Space — © Bálint Decsi,
    github.com/balintdecsi/…"). Apache §4(d) requires redistributors to carry the NOTICE along.
  - Our own data (NDZ transcription, derived zones, protected sites, tables): **CC-BY-4.0**.
    Reusers must credit the creator where the data is shown, e.g. in the map attribution.
  - The embed's attribution control credits "Open Drone Space" by default. Reusers of the
    Apache-licensed code may remove it; reusers of the CC-BY data must credit the source anyway.
  - Third-party-derived data keeps its own licence: OFM → OFM licence; terrain → the Copernicus
    notice. Map it with REUSE (`REUSE.toml`, SPDX ids per path).
- Personal data policy: no contact details for persons in the public repo, not even public
  ones. Git history, forks, npm and Zenodo releases are immutable, so an entry can never be
  corrected or erased. The contacts stay in the private app, which can update them.

### Phase 1 — New public repo, fresh history
- Create it with an initial commit. Don't copy the private repo's history.
- Name `open-drone-space-data`. Layout:
  ```
  scripts/fetch/     download raw inputs → data/raw/ (allowlist + sha256 per source)
  scripts/ingest/    moved from private: parse-ofmx, build-ndz, parse-aerodrome-register,
                     build-aerodrome-zones, build-rmz-tmz, build-protected, build-terrain,
                     build-tables (minus fee-accounts), publish, lib/ (incl. dropzones.mjs)
  scripts/ingest/data/   protected-sites.json, restricted-area-operators.json
  data/              published output: *.geojson, tables/, manifest.json, datapackage.json, i18n/source-text.json
  src/               shared TS: types.ts, catalog.ts (categories/colours), height-filter.ts, geo utils
  embed/             the embeddable map (Phase 3)
  tests/             data-validation tests
  docs/              DATA_SOURCES.md (static sections), MANIFESTO excerpt, CONTRIBUTING
  ```
- `parse-aerodrome-register.mjs` gets split: an exported parser returns full rows, and the
  public build writes only `registerId, name, icao, kind, kindEn, section, settlement, lat, lon`.
  The private app imports the same parser to build its contacts table.
- `data/raw/` stays git-ignored. The fetch script has an **allowlist** of public sources, so
  no non-public reference material can ever enter the repo.
- Tests to port: `dropzones`, `terrain`, `height-filter`, the data half of
  `uncontrolled-ceiling`, and `tests/support/data.ts`. Add JSON Schema validation of feature
  properties and geometry validity (closed rings, no self-intersection).
- Add a `datapackage.json` (Frictionless Data) generated from the existing manifest, which
  already carries source, licence, checksum and dates.
- Add a `.github/ISSUE_TEMPLATE/data-error.yml` "Report a data error" form. It satisfies OFM's
  error-report duty and is the contributor entry point.

### Phase 2 — Automation (GitHub Actions)
- **CI on PR:** lint, `bun test`, rebuild from committed inputs, and post a feature-count and
  geometry diff summary as a PR comment.
- **Scheduled refresh** (weekly, plus the day after each AIRAC effective date): fetch sources,
  compare checksums, and when something changed, rebuild and open a PR
  (`peter-evans/create-pull-request`). A human still reviews the legislative geometry. The job
  never pushes to main.
- **Manual-until-verified sources:** check whether the OFM package and the CAA register can be
  downloaded without a login or a click-through licence. If not, the workflow flags "new
  edition likely" and a maintainer drops the file in by hand.
- **Release on tag** (`vYYYY.MM.N`, or the AIRAC id): GitHub Release with the data as assets,
  npm publish with provenance (`@balintdecsi/hu-airspace`: data + TS types + catalog), and a Pages
  deploy of the embed and a browsable data index. jsDelivr then serves both npm and GitHub
  files for free.
- **Zenodo** integration: each release gets a DOI, so the dataset is citable.

### Phase 3 — Embeddable map
- Vite library build using Leaflet, the same library as the app. Reuse `catalog.ts` colours
  and the height filter.
- Two ways to embed:
  - `<iframe src="https://<pages>/embed/?layers=ndz,ctr,aerodrome-2km&lang=hu&lat=…&lon=…&z=…">`
  - `<script src="https://cdn.jsdelivr.net/npm/@<scope>/hu-airspace/embed.js">` +
    `OpenDroneSpace.mount(el, {...})`
- Required in the UI: OFM/Copernicus attribution, "not for navigation", the fixed
  **"Daily activations and eseti légterek are not shown — check opendrone.space"** banner, a
  "Report an error" link to the issue form, and the data version/date from the manifest.
- Bilingual hu/en strings, kept small inside the embed.

### Scope of v1 (decided)
v1 = Phases 0–3, in the personal-account repo `balintdecsi/open-drone-space-data`. The private app keeps
its own pipeline unchanged. To limit drift until Phase 4:
- The public repo becomes the place where static-ingest fixes are made first.
- A cheap bridge (optional v1.1): `public/data` in the app stays byte-compatible with the
  public `data/`, so a tiny `sync-data` script (copy the files of a tagged release) can replace
  the private ingest for these datasets with no code-sharing work.

### Phase 4 (later) — Private app consumes the public package
- Add the npm package as a dependency. A small `sync-data` script copies the pinned release
  into `public/data/`, so the app keeps working when the DB is down and Lovable builds as
  before. Or fetch it from jsDelivr at a pinned version.
- Delete the moved scripts from the private `scripts/ingest/`. Keep a private
  `build-contacts.mjs` that uses the public register parser, plus `fee-accounts`, the
  translation step, tiles, and the daily AUP/NOTAM code.
- Point `src/lib/airspace/catalog.ts` / `types.ts` at the package (re-export) so the two
  repos can't drift apart.
- Update docs: README data table, DATA_SOURCES.md, ARCHITECTURE.md "two cadences" (now:
  upstream repo vs private DB), and the `/sources` page link to the public repo.
- Dependabot/Renovate on the private repo bumps the data package → PR → Lovable deploy.

## Tools

- **Use:** GitHub Actions, Pages, Releases, npm (+ provenance), jsDelivr, REUSE/SPDX,
  Frictionless `datapackage.json`, Zenodo, issue forms, Dependabot. All free for public repos.
- **Skip Travis CI.** Its student-pack offer is free *private* builds; Actions is free and
  unlimited on public repos and already where the code lives.
- **Skip Zyte / Scrapy Cloud.** A handful of static government documents on a monthly cadence,
  with no anti-bot, doesn't need a crawling platform. The hard part is parsing, not fetching,
  and it would add a Python stack to a JS pipeline.
- **Student pack, marginal:** Sentry (private app errors), Simple Analytics (privacy-friendly
  embed usage stats, fits the manifesto), BrowserStack/LambdaTest (checking the embed on
  mobile Safari). Nothing in the pack is needed for the FOSS repo.
- **Worth learning for this project:** semantic/calendar versioning of datasets, GitHub
  Actions scheduled workflows + PR bots, npm library builds (Vite library mode), open-data
  licensing (ODbL vs CC-BY vs custom). Later, **ED-269** GeoZone output
  (`src/lib/airspace/ed269.types.ts` exists): Hungary publishes no ED-269 feed, so an open
  one would be a real contribution.

## Verification

- Public repo: `bun test` passes against `data/`. A fresh clone plus `scripts/fetch` +
  `ingest:all` reproduces `data/` with identical checksums (manual sources pre-placed).
- `grep -rE '\+36|@[a-z0-9-]+\.(hu|com)' data/` finds nothing. `reuse lint` is clean.
- The embed loads from a plain HTML file served by `python -m http.server`, both as an
  iframe and a script. The banner, attribution and error link are visible (check in a headless
  browser at phone width).
- Private app after Phase 4: `bunx tsc --noEmit`, `bun test`, `bun run build`, and
  `git diff --stat public/data` shows no unexpected change. The map renders the same layers.
