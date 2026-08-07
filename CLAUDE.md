# PCARDB Loan Automation — Project Guide for Claude Code
yhid rthgghghtrg
> **Keep this file current.** After every significant change (feature, schema,
> template, deployment), update the **Changelog** and any affected section
> below. This file is the context bridge between sessions.

## What this is

Digitizes loan applications for Gokak Taluka PCARD Bank (Karnataka). Operator
fills a React form → FastAPI + SQLite stores it → the system generates the
bank's standard Kannada print packet as a PDF (**21 pages for Tractor, 23 for
LAND_DEV**). **Tractor and LAND_DEV schemes are built**; the remaining 3
schemes (BULLOCK, SHEEP_40/20/10) are gated on bank sign-off of the Tractor
pilot and will replicate the same pattern. LAND_DEV went through its first
owner-driven correction round on 2026-08-06 (new crop chart + Appraisal
LD.xlsx, verified page-by-page vs the reference PDF); the bank's own review
round is still to come.

**Repo:** https://github.com/ayushpatravali/pcardb (owner's personal GitHub;
a fork on a second account feeds the Railway demo — "Sync fork" after pushes).

## Architecture (current — the old Excel/win32com engine is GONE)

```
React form (frontend/src/pages/NewApplication.jsx)
  → POST/PUT /applications (routers/applications.py, exact SchemeType routing)
  → SQLite row (models.py; JSON-string columns for arrays)
  → GET /pdf/download/{id} or POST /applications/{id}/generate
  → services/render_service.py: build_context() fills the scheme's Jinja2
    HTML pages (21 Tractor / 23 LAND_DEV)
  → WeasyPrint → PDF (~1s). Missing required fields → 422 with
    {missing: [keys], fields: [{key, label_kn, label_en}]} — the print page
    shows the labels with an Edit button (application stays saved).
```

- **Templates:** `backend/templates/pages/*.html` (+ `pages/tractor/t1–t7`),
  assembled by `templates/packet.html` in the order defined in
  `backend/schemas/tractor.py` `PAGES`. CSS in `templates/css/base.css`.
  Labels are Unicode Kannada transcribed from the bank's reference PDFs
  (`legacy_assets/pdfss/`, NOT in git — contains real customer PII).
- **Field specs:** `backend/schemas/` — tier per field: `collected` (form) /
  `computed` (server-derived) / `constant` / `handwritten` (prints blank).
- **Computed server-side:** annual_income = Σ crop incomes from `current_crop`
  JSON; per-parcel land valuation = `land_valuation_per_acre` × extent;
  totals; Kannada amount-in-words (`utils/kannada_numbers.py`).
- **JSON-string columns on Application:** `co_applicants`, `land_parcels`
  (incl. per-parcel `valuation`), `current_crop`, `previous_loans`.
  Numeric fields inside them are coerced to float at render (form sends strings).
- **Auth:** JWT, 7-day default (`TOKEN_EXPIRE_MINUTES` overrides); the SPA
  verifies the stored token via GET /users/me on load (dead session →
  login BEFORE typing); roles manager/field_officer; enums stored by VALUE
  (lowercase); signup is DISABLED unless `ALLOW_SIGNUP=1` (local scripts set it).
- **Form drafts:** new-application form autosaves every change to
  localStorage (`pcardb_draft_<scheme>`), restores on reopen, clears on
  successful save — typed work survives dead sessions/crashes. 401 at save
  keeps the form and instructs re-login in a new tab.

## Run

| Task | macOS/Linux | Windows |
|---|---|---|
| one-time setup | `make setup` | `setup.bat` |
| localhost | `make run` | `run.bat` |
| + ngrok tunnel | `make run-ngrok` | `run_ngrok.bat` |

Default logins `manager/manager123`, `officer/officer123` (env-overridable:
`MANAGER_PASSWORD`, `OFFICER_PASSWORD`). macOS needs
`DYLD_FALLBACK_LIBRARY_PATH=/opt/homebrew/lib` (brew pango) — scripts set it.
Never run WeasyPrint through `nohup`/`env` on macOS (SIP strips DYLD vars).

**Demo hosting (Railway, free):** Dockerfile at repo root; single container
serves API + built frontend (`/api/*` routes + SPA fallback in `main.py`).
Env vars on the service: `MANAGER_PASSWORD`, `OFFICER_PASSWORD`, `DEMO_SEED=1`
(auto-seeds the editable Vasant Malli sample each boot). `GET /healthz` shows
env delivery as booleans. **Persistent data:** attach a Railway Volume mounted
at `/data` and set `DATABASE_URL=sqlite:////data/database.db` (four slashes) —
without it the disk is ephemeral and typed applications are LOST on redeploy
(bit the owner on 2026-08-03). Volume DB schema persists across deploys, so
future schema changes need ALTER statements there too (or delete the volume DB
to recreate).
Final deployment target: fully local on the bank manager's machine, bank LAN only.

## Testing

From `backend/` with venv `/Users/ayush/project/.venv-mac`:
- `python tools/render_test.py [--scheme LAND_DEV] [--pages a1,b3]` — render
  fixture packet; must print `pages: 21 OK` (Tractor, default) or `pages: 23
  OK` (`--scheme LAND_DEV`)
- `python tools/render_highlight.py <id>` — packet with UI-input values highlighted yellow (bank review)
- `python tools/compare_pdfs.py <pdf>` — page-count + side-by-side images vs reference
- `python tools/seed_reference_apps.py` — seed Vasant Malli sample
- e2e pattern: `with TestClient(main.app) as c:` (context manager, or startup/seeding won't run)

## Hard rules

1. **PII never in git**: `legacy_assets/`, `*.xlsx`, `*.pdf`, `*.db` are
   gitignored. Real applicant data moves by USB/AirDrop only.
2. **Form ↔ backend contract**: backend adapts to what the form sends; field
   names in `ApplicationCreate` mirror the form payload exactly.
3. **Kannada input must be Unicode.** Nudi legacy (ASCII) input is detected
   and rejected server-side (`reject_nudi_ascii`). Bank machines: Nudi 6 in
   Unicode mode.
4. **Templates use StrictUndefined** — every referenced key must exist in the
   context (see `p.setdefault("valuation", None)` pattern).
5. **Local schema changes**: `ALTER TABLE` the dev `database.db` in place
   (see git history for one-liners); Railway recreates its DB on redeploy.
6. **Page-count fidelity**: any page edit must keep `render_test.py` at 21
   pages for Tractor / 23 for LAND_DEV (`--scheme LAND_DEV`) — fixed-height
   pages, tighten spacing if content grows.
7. Git identity/remote is repo-local only; never touch global git config
   (owner has separate org GitHub in other VS Code windows).

## Changelog

- **2026-08-07** — LAND_DEV correction round 2, from the owner's annotated
  `land_dev corrections.pdf` (repo root, NOT in git — 18 highlights + 12
  comments, all mapped and fixed; backend/templates only, frontend untouched):
  (1) **"+ 0" insurance suffix removed** at all 5 remaining sites (a1 request
  line + scheme-table row, b1 ×2, b2 ×2, valuation/committee page) via the
  existing `plus_insurance` filter; Tractor still prints "+ 1,00,000"
  (verified 11 text sites, 21 pages). (2) **ಅಂಚೆ ≠ hobli**: a1/ld1/inspection
  printed `app.hobli` under the ಅಂಚೆ label — now the village prints there
  (right-aligned on a1) and the hobli value fills inspection's empty ಹೋಬಳಿ
  slot. (3) a1 item 2: leftover "ಅ) ಸಣ್ಣ / ದೊಡ್ಡ ರೈತರು" option text removed.
  (4) **Farmer-type repayment terms** (replaces the 08-06 N−2 reading):
  small farmer 7 yrs / big farmer 6 yrs; yearly kantus = years−1 (small 6,
  big 5) after the 12-month grace; repayment % of incremental income = 50
  small / 75 big (new computed `repayment_pct`, b4 12 ಉ + ld4 21 bind it;
  Tractor stays 75). LAND_DEV ignores the form's loan-duration select for
  these prints. (5) **Loan cap**: create/update reject LAND_DEV loans above
  the dev-works estimate (bilingual 400); b3's cost row prints blank when a
  stored (legacy) loan still exceeds cost — no more negative ಸ್ವಂತ ಕೊಡುಗೆ.
  (6) **Water source**: a2 item-5 land table + ld3 ಖುಷ್ಕಿ/ನೀರಾವರಿ column now
  print the land's water source; season/irrigated crop-row values with no
  Kannada (operator typed "e"/"f") print blank. `rate_per_acre`/`amount`
  added to numeric JSON coercion (string amounts crashed ld2's %.2f).
  (7) Bank name: ld9 estimate header got the missing name_line2; ld5's two
  signature lines unified to "ಪಿಕಾರ್ಡ ಬ್ಯಾಂಕ ನಿ ಗೋಕಾಕ"; pp's ಮೇಲ್ವಿಚಾರಕರು
  centred over its bank line. Both packets 21/23 OK; API e2e (cap 400 →
  create 200 → edit-cap 400 → PDF 200) passes. Skipped per user: the
  "remove this Blank page" note on ld3's empty lower half (likely an owner
  mistake — would change the 23-page count). Follow-up same day: form now
  matches — LAND_DEV's duration select is replaced by a locked farmer-type
  display (small 7 / big 6; payload always sends the derived value, so old
  rows self-correct on save) and the live-summary kantu uses
  loan/(years−1) with the 12-month initial period noted.
- **2026-08-06** — Full LAND_DEV verification vs both new Excel files + the
  reference PDF (owner: "make sure you are completely right"). Machine-
  checked all 31 CROP_ECON/CROP_CHART entries against the chart xlsx (exact
  match), compared every generated ld page against the reference page-by-
  page. Fixes: (1) **ld4 item 15 is 80%** of valuation (ref p14 + Ap4
  formula), NOT the 50% used on b4 — the bank's own packet uses BOTH
  percentages (p10=50 economic capacity, p14=80 appraisal); new computed
  keys appraisal_loan_eligibility/appraisal_net_loan_eligibility.
  (2) **Moratorium implemented** (closes the documented gap — confirmed by
  ref p10+p14 AND Ap4): LAND_DEV repayment = 12-month initial period +
  (N−2) equal YEARLY kantus, installment_kantu = loan/(N−2) (7yr/15L → 5
  kantus of 3,00,000); b4 ಎ/ಏ lines scheme-branched, ld4 items 19/20
  filled; Tractor untouched (loan/N ವಾರ್ಷಿಕ). (3) b4 sanction line "+ 0"
  → blank via plus_insurance. (4) ld1 header = state bank (ಬೆಂಗಳೂರು-18)
  per ref p11/Ap1. (5) ld2 works table gained the ದರ column + ಸಂಸ್ಥೆಯ
  ಹೆಸರು prints "ರೈತರಿಂದ ಮಾಡಲಾಗುವದು" (Ap2 constant); dev_work_items rows
  get setdefault guards. Pages 10/12/14 re-verified visually against the
  reference; both packets 21/23 OK; API e2e passes. Crops dropdown: ALL 31
  chart crops stay available for pre/post-dev (owner). Known acceptable
  deviations from the ref sample: extent multiplies as acres+guntas/40
  (24.5) not the ref's literal "24.2"; my 6-row works table = ref's 7 rows
  (two zero-rate rows merged; totals identical).
- **2026-08-06** — LAND_DEV corrections from owner's new Excel files
  (`CROP INCOME CHART for land dev.xlsx` + `Appraisal LD.xlsx`, both at repo
  root, NOT in git): (1) **ಕ್ಷೇತ್ರ ಆಯವ್ಯಯ ತಃಖ್ತೆ (ld3/Ap3) now fully computed**
  — new `CROP_ECON` chart in the form ([yield/acre, market rate,
  expenditure/acre] for all 31 crops; net = y×r−c matches the existing
  CROP_CHART values exactly); pre/post-dev crop rows store
  cost_per_acre/total_cost/yield_per_acre/total_yield/rate/total_income and
  annual_income = income−cost (chain mirrors the reference sheet: totals from
  ROUNDED total yield). Previously these columns printed blank for typed
  applications ("something in the database not going properly"). ld3 also
  gained the reference's 4-item other-costs box (ಭೂಕಂದಾಯ/ಬೆಳೆ ಸಾಲದ ಬಡ್ಡಿ/
  ವಿದ್ಯುಚ್ಛಕ್ತಿ/ಇತರೆ) in col 13 with section totals in col 14. Legacy rows
  without econ keys still render (setdefault None). (2) **Dev-work rates
  locked** to bank constants [5336, 61714, 7807, 0, 3429, 1714] (owner +
  LD1 sheet) — compact read-only table in the form (🔒, small font, total
  row); operator cannot edit; old stored rates corrected on next save.
  (3) **Print-page dead end fixed**: PDF now fetched via the API client;
  422 shows an amber "Application #N is SAVED — nothing lost" panel with
  the missing fields as Kannada+English LABELS (backend 422 now includes
  `fields: [{key,label_kn,label_en}]`) and an "Edit & Fill" button to the
  edit form (was: raw status-code JSON in a new tab, operator stranded).
  **Flag for bank**: reference Ap3 multiplies 24ac20g as literal "24.2";
  form uses correct 24.5 (matches valuation + dev-work pages). Pending:
  owner will send a restricted crop list + "XYZ prices" discussion.
- **2026-08-06** — Dead-session data loss fixed (tester lost a fully typed
  application to a 401 at save). Root cause: "logged in" was just a token
  string existing in localStorage — never validated — and the application
  form makes NO api call until Save, so an expired token surfaced only after
  all the typing, and the form threw the work away. Four layers: (1) token
  default 480min → 7 days (TOKEN_EXPIRE_MINUTES still overrides);
  (2) AuthContext now verifies the stored token via GET /users/me on load —
  dead sessions bounce to login BEFORE typing (network errors keep the
  session); (3) form drafts autosave to localStorage per scheme on every
  change, restore when the blank form reopens, clear on successful save;
  (4) 401 at save → bilingual alert "log in in a new tab, come back, press
  Save — nothing lost", stays on the page (works for edits too; the retried
  request reads the fresh token). Needs Railway redeploy to reach testers.
- **2026-08-06** — LAND_DEV form: generic Crop Details table hidden (owner —
  duplicate of the pre/post-development crop tables; only irrigation sources
  stay in Agriculture Details). Payload sends `current_crop: []` for LAND_DEV
  (clears stale duplicates on edit); render_service falls back
  `parsed.crops → pre_dev_crops` so the two shared pages that print the
  currently-growing crop (ssm item 3, inspection crop column) still fill.
  Verified on the fixture packet (both pages show the first pre-dev crop;
  23 pages OK). Pending owner input: a restricted crop list for the
  pre/post-dev dropdowns ("I'll give you the list later").
- **2026-08-05** — Post-build audit fixes: (1) removed a dead duplicate
  "Land Development Metrics" form card (pre-rebuild leftover — 5 fields
  writing to payload keys nothing reads, plus a second land_type dropdown
  colliding with the real one). (2) Live summary bar's installment updated
  to the YEARLY formula (loan/duration, tagged ವಾರ್ಷಿಕ) — it still showed
  the half-yearly loan/(2×duration) the 2026-08-04 bank review retired, so
  the on-screen kantu contradicted the printed packet. (3) Verified the full
  LAND_DEV lifecycle by test: create → read → edit → re-read → PDF (edited
  values persist, no duplicate details row); empty crop lists correctly 422
  with a field list at PDF time.
- **2026-08-04** — LAND_DEV scheme built end-to-end (23-page packet, mirrors
  Tractor's architecture): `schemas/land_dev.py` spec; `LandDevDetails`
  model rewritten (land_type + JSON columns `pre_dev_crops`/`post_dev_crops`/
  `dev_work_items` + `total_dev_cost`, replacing the pre-rebuild stub);
  9 new page templates (`templates/pages/land_dev/ld1-9.html`) transcribed
  from the Kallangouda V Patil reference PDF (23 pages: `legacy_assets/
  pdfss/Kallangouda Patil Land Development Scheme.pdf`); real pre/post-
  development crop-income computation (not Tractor's flat 30% heuristic) —
  `computed.incremental_income` = post-dev crop income − pre-dev crop income,
  feeding the *already-built* shared financial-capacity page (10) with no
  template changes needed there. Two cross-scheme generalizations found by
  reading the reference PDF: loan-eligibility percentage is scheme-specific
  (`LOAN_ELIGIBILITY_PCT`: Tractor 80%, Land Dev 50% — was hardcoded 80% for
  everyone) and `installment_kantu` now keys off the generic `app.loan_amount`
  instead of Tractor-only `details.total_loan_amount`. Shared page 9
  (`b3.html`) made scheme-aware — it hardcoded tractor make/HP/dealer and a
  Tractor/Trailer cost table; Land Dev's reference shows that item blank and
  a single "ಭೂ ಅಭಿವೃದ್ಧಿ ಯೋಜನೆ" cost row instead. Frontend: land-type dropdown,
  two crop tables (pre/post-dev, reusing the Tractor crop-row component +
  crop-rate chart), 6-row development-cost table with per-acre rate inputs.
  `render_test.py` gained a `--scheme` flag + Land Dev fixture (23 pages);
  seeded a Kallangouda-pattern sample alongside Vasant Malli. Verified against
  the reference PDF page-by-page and via a live API create→render round trip.
  Design/plan docs: `docs/superpowers/specs/2026-08-04-land-dev-design.md`,
  `docs/superpowers/plans/2026-08-04-land-dev-implementation.md`.
- **2026-07-31** — Engine rebuild: retired generator.py/excel_service/
  pdf_service/win32com + excel router + nudi_converter. New schemas package,
  render_service (Jinja2+WeasyPrint), 21 Tractor page templates transcribed
  from reference PDFs, models cleanup (enum values, exact scheme routing,
  dropped duplicate columns), verification tools, Makefile, README.
  Cross-checked labels via knconverter; 6 spelling fixes.
- **2026-07-31** — Demo hosting: Dockerfile (node build + python/pango),
  /api route aliases + SPA serving, env-seeded users, signup off by default
  (`ALLOW_SIGNUP` opt-in), `/healthz`, `DEMO_SEED` auto-sample. Railway live
  at pcardbbank-gokak.up.railway.app (via fork on second account).
- **2026-07-31** — Fixes found by e2e: create-response losing id (ORM refresh),
  form's string-typed numbers crashing PDF arithmetic (float coercion).
- **2026-08-02** — Land valuation (Tractor): `land_valuation_per_acre` input,
  locked per-parcel value column + total in form; page 19 (ಅನುಬಂಧ–2) value
  column/total/per-acre certification line filled.
- **2026-08-02** — Application date override (blank = today), irrigation HP
  dropdown 1–20 step 0.5 + "HP only for motorised sources" note, page-1 photo
  box → passport 35×45mm labeled ಅರ್ಜಿದಾರರ ಭಾವಚಿತ್ರ.
- **2026-08-03** — Formula map: all 489 cell formulas from the Vasant Malli
  Tractor workbook extracted to `legacy_assets/formula_maps/tractor_formulas.json`
  (gitignored) — reference for every computed figure in the packet.
- **2026-08-04** — Bank-review round (tester feedback batch): (1) numeric
  inputs now accept any decimal — `step="any"` default in InputField + raw
  land-table inputs (akaar 14.91 was being rounded to 14 by browser number
  validation; old rows have the truncated value STORED — re-edit to fix).
  (2) b1 item 1 + b2 item 14 print bank.place ಗೋಕಾಕ (were operator-typed
  branch, showed ಕುಲಗೋಡ); b2 line is now fully static text
  "ಪ್ರಾ.ಸ.ಕೃ.ಗ್ರಾ.ಬ್ಯಾಂಕ್ ನಿ ಗೋಕಾಕ". (3) b1 item 5 label ಸದಸ್ಯ ಸಂಖ್ಯೆ →
  ನೀರಾವರಿ ಕ್ಷೇತ್ರ. Reminder: owner must Sync fork + redeploy for testers.
- **2026-08-04** — Number-input wheel bug: raw land-table inputs (acres/
  guntas/akaar) + land_valuation_per_acre now blur on mouse wheel and have
  min="0" (InputField already did both). Cause of the −4.09 akaar: operator
  typed 14.91, then scrolled the page with the field still focused — Chrome
  steps a focused number input by 1 per wheel tick (19 ticks down = −4.09).
  Stored bad values must be re-edited; the mechanism is now blocked.
- **2026-08-04** — t6 (page 16) bank review: (1) 10.3 + 10.4 print blank
  (was app.taluk ಗೋಕಾಕ — bank fills by hand); (2) 10.9 typo ಆಸ್ತಿ → ಆಸಕ್ತಿ;
  (3) bottom ವಾಸ್ತವ್ಯ now prints bank.place ಗೋಕಾಕ (was app.village ಕುಲಗೋಡ).
- **2026-08-04** — t5 (page 15) bank review: (1) 8.7 REVERTED to workbook's
  "+ 1,00,960" (TRAILER_HIRE_INCOME constant; owner had called it stale on
  08-03 and switched to t4's net 2,00,000 — bank wants 1,00,960; t4's own
  hire chain unchanged; ssm2 item 13 follows via repayment_capacity).
  (2) 8.10 interest rate 12.50 → 12.00. (3) 9.1 table: hardcoded "ಯಾವದೇ ಸಾಲ
  ಪಡೆದಿರುವದಿಲ್ಲ" removed + full column borders restored; row now binds
  previous_loans (purpose/total/outstanding/installment/repaid_status) when
  the applicant has old loans, blank otherwise.
- **2026-08-04** — Certification wording (bank review): t3 (p13) + pp (p20)
  now say "ಘಟಪ್ರಭಾ / ಮಾರ್ಕಂಡೇಯ ಅಚ್ಚುಕಟ್ಟು"; pp's broken "ಇದ್ದು ___ ಮತ್ತು ___
  ಯಿಂದ" line rewritten to t3's clean form "ಇದ್ದು, ಇವುಗಳಿಂದ
  ನೀರಾವರಿಯಾಗುತ್ತದೆ ಅಂತಾ ದೃಢೀಕರಿಸಿದೆ."
- **2026-08-04** — t1 (page 11) bank review: (1) 2.1 tractor make now prints
  beside its label (was centered far right); (2) removed the loan+insurance
  print on the ಅಶ್ವ ಶಕ್ತಿ line (12 insurance sites remain — the t1 totals-table
  one stays); (3) 2.3 "(ಟನ್‌ಗಳು)" moved inline after the capacity value (was
  orphaned at the left margin).
- **2026-08-04** — b4 (page 10) bank review: (1) section 11 ಆ/ಇ now filled —
  post_dev_income (annual + 30%) and incremental_income (30%), were blank.
  (2) 12 ಅ) right label ಸದಸ್ಯ ಸಂಖ್ಯೆ → ದಾಖಲೆಯ ಪುಟ ಸಂಖ್ಯೆ. (3) ಏ) installment
  is now YEARLY — installment_kantu = total_loan/years (was ÷2N half-yearly;
  11.2L/7 = 1,60,000), prints "(ವಾರ್ಷಿಕ)" after the amount; period line now
  "N ವರ್ಷ (N ಕಂತು)". Matches t5 8.9 which was already annual — no other
  installment sites needed the ವಾರ್ಷಿಕ tag.
- **2026-08-04** — b3 item 10: label ಒಪ್ಪಟ್ಟಿದೆ → ಒಪ್ಪಬಹುದೇ; removed stray
  `details.tractor_make` binding there (printed ಜಾನ್ ಡೀಯರ್) — now a blank
  line. Make still prints at its correct site, b3 item 5.
- **2026-08-04** — b2 ಶಾಖಾ ಕಛೇರಿ section (page 8) prints fully blank per bank
  review: box-5 pahani/MR/kraya page numbers (was survey_no + 0s), section-6
  asset table (was total_extent + 0), both ಇಲ್ಲ answer boxes, ಉ) ಹೌದು box, and
  all item-7 zeros — bank staff fill this section by hand. Item 8 and item 14
  (below/above the reviewed area) unchanged.
- **2026-08-04** — Removed 3 stray loan+insurance prints flagged by bank
  review: a3 beside ದೃಢೀಕರಣ heading, t2 heading line 5, valuation between the
  Sd/ signatures (the one in valuation's ಮಂಜೂರ column stays). 13 insurance
  sites remain.
- **2026-08-03** — Dates: dashboard table pinned to en-GB (was viewer-locale;
  US testers saw mm/dd on the same URL). Native date pickers REPLACED with
  DD/MM/YYYY masked text inputs (browser widget follows the viewer's OS
  region — cannot be styled). dmyToIso/isoToDmy convert at payload/hydration;
  age auto-calc parses the masked format. PDF was always dd/mm/yyyy.
- **2026-08-03** — Hire-income chain computed from constants (HIRE_RATE=400/hr
  × 600 hrs): t4 row = 2,40,000 → −20,000 → 2,20,000 → −20,000 → net 2,00,000;
  t5 8.7 = 75% incremental + 2,00,000 (replaces workbook's stale +1,00,960);
  ssm2 item 13 follows. Owner confirmed the workbook figures were mistakes.
- **2026-08-03** — Insurance "+1,00,000" now at EVERY loan-amount print site
  (16 total): a1 request line + scheme table, a3 declaration, b1 demand+loan
  lines, b2 ×2, b3 cost table, b4 sanction, t1 ×2, t2, t5 8.8, t7, valuation
  ×2. Single constant INSURANCE_AMOUNT in render_service.
- **2026-08-03** — PDF batch (owner): t4 rate ₹200→₹400; t5 8.5 shows old-loan
  outstanding, 8.6 = eligibility − outstanding, 8.7 prints "r75 + 1,00,960"
  (fixed TRAILER_HIRE_INCOME) with total; ssm2 item 13 = that 8.7 total
  (`computed.repayment_capacity`) replacing stale 59920 (workbook linked the
  SHEEP sheet's net income); a1 scheme table gets "+ 1,00,000" insurance row;
  ssm boxed title centered. **`num` filter now prints Indian digit grouping**
  (11,30,000) and strips operator-typed commas before parsing (also in
  `_to_float` and annual-income derivation).
- **2026-08-03** — Palette: primary green rebased on John Deere green #367C2B
  (owner request) — whole scale regenerated in tailwind.config.js; chart hexes
  in Home.jsx updated to match. Accent gold unchanged.
- **2026-08-03** — Caste list from CROP INCOME CHART.xlsx: dropdown now has the
  bank's 12 castes verbatim (knconverter-decoded, keeps their ಪರಿಶಿಷ್ಠ
  spelling): ಪರಿಶಿಷ್ಠ ಜಾತಿ/ಪಂಗಡ, ಅಲ್ಪ ಸಂಖ್ಯಾತರು, ಇತರೆ ಸಾಮಾನ್ಯ (default),
  ದಿಗಂಬರ ಜೈನ, ಮುಸ್ಲಿಂ, ಹಿಂದೂ ರಡ್ಡಿ/ಮಾಳಿ/ಬಣಜಿಗ/ಕುರಬರ/ಲಿಂಗವಂತ/ಉಪ್ಪಾರ + free-text
  "other" option (chart note). Stored value prints verbatim on the packet
  (verified: ಹಿಂದೂ ಕುರಬರ on page 1). Legacy General/SC/ST values map across;
  unknown legacy castes open as "other" with the text preserved.
- **2026-08-03** — Crop chart from bank's CROP INCOME CHART.xlsx: all 31 crops
  with authoritative per-acre rates (e.g. ಕಬ್ಬು 83000, ಭತ್ತ 17400, ದ್ರಾಕ್ಷಿ
  (ಬೀಜ ರಹಿತ) 172500). Stored crop value is now the exact Kannada chart name
  (no English intermediary → no drift); legacy English values map via
  CROP_LABEL_MAP for old rows (Soybean/Tomato/Chilli/Other have no chart entry:
  kept as typed, stored income preserved). Verified 31/31 names+rates match the
  xlsx byte-for-byte. Chart's embedded feature notes (caste list, female-loanee
  dashboard, free-text caste, hobli/taluk dropdowns) recorded in
  NEXT_SESSION_TASKS item 7 — not yet implemented.
- **2026-08-03** — Kannada-only mode: ಕನ್ನಡಕ್ಕೆ ಬದಲಿಸಿ now makes the whole form
  pure Kannada (labels, section titles, dropdown/checkbox options, notes,
  buttons, placeholders, sidebar) via `src/lib/kannada.js` knLabel/knOption —
  display-only transforms; stored option VALUES unchanged (backend contract +
  PDF kn_display intact). English mode keeps the bilingual mix. Language
  choice persisted in localStorage (used to reset on refresh). This closes
  NEXT_SESSION_TASKS item 6 (was parked for pre-go-live; owner pulled it in).
- **2026-08-03** — UI v2: shadcn/ui components authored in src/components/ui
  (button/card/input/badge/separator, cva + cn util, @ alias). Login brand
  panel gains bank+farmland SVG backdrop. Dashboard: per-scheme bar chart
  (recharts, counts + sanctioned value) and loan-size band distribution from
  fetchApplications. Form: sticky live-summary bar (extent, loan, installment
  ≈ loan/(2×duration), updates per keystroke), shadcn Buttons for add-row and
  submit, InputField wraps shadcn Input.
- **2026-08-03** — UI overhaul (no flow/field changes): forest-green + harvest-
  gold design system (tailwind.config `primary`/`accent`/`surface`, self-hosted
  Noto Sans Kannada in frontend/public/fonts). Redesigned Login (bank brand
  panel, Kannada identity), SignUp (+ ವಲಯ region input feeding User.region),
  Layout (dark green sidebar), Home (stat cards + recharts status donut),
  SelectScheme; NewApplication restyled via SectionHeader/InputField/SelectField
  (new `variant` prop: dark/highlight for the totals panels). Deps: motion,
  recharts, class-variance-authority. Loan-duration select moved after
  caste/farmer/borrower-type block (owner request).
- **2026-08-03** — Form fix: crop income + land totals/valuation now recompute
  on every keystroke. RHF `watch()` returns the same array ref while typing
  inside field-array rows, so the effects only fired on row add/remove — now
  keyed on JSON-serialized row contents.
- **2026-08-03** — Display normalization (render_service): (1) acre.gunta
  extent notation — parcels/crops get `p.extent`/`c.extent` ("8.19" = 8 ac 19 g)
  and `computed.total_extent`; all templates that printed bare acres now use
  them (page 19 keeps its separate acre/gunta columns). (2) Kannada-only
  output: `kn_display()` strips English halves of bilingual form values —
  "General / ಸಾಮಾನ್ಯ"→ಸಾಮಾನ್ಯ, "ಕಾಲುವೆ (Canal)"→ಕಾಲುವೆ (keeps "(7 HP)"),
  English crop values map via CROP_KN (Sugarcane→ಕಬ್ಬು…), farmer_type token
  mapping. New computed keys: caste_kn, irrigation_kn, total_extent.
- **2026-08-03** — Page 10 section 12 (ಆರ್ಥಿಕ ಸಕ್ಷಮತೆ) fully computed using the
  workbook's B4/T5 formula chain (mapped in `docs/formula_map_pages_9_10.md`):
  security = land_valuation_total; loan eligibility = 80% of it; net = − old
  outstanding; repayment eligibility = 75% of (30% × annual income); net = −
  old installment (page 9 ಉ feeds page 10 ಊ); ಏ = per half-yearly installment
  = total loan ÷ (duration × 2), e.g. 11.2L over 7 yrs = 14 kantu of 80000.
  Also: b4 sanction line now prints "+ 100000" (workbook B4!F42 hardcodes it —
  5th insurance site); b3 item-5 dealer/ಸಮರ್ಪಕವಾಗಿದೆ values left-aligned.
- **2026-08-03** — Loan duration + insurance: `loan_duration_years` field (form
  select 1–15, default 7) drives b4 repayment period ("N ವರ್ಷ (2N ಕಂತು)") and t5
  kantu (= total_loan/duration) + 8.10 row. Fixed `INSURANCE_AMOUNT=100000`
  (render_service constant) prints as "+ 100000" (never the word insurance) at
  the 4 final-loan-amount sites: b3 cost table, t1 totals table, t5 8.8, t7
  recommendation line (fills the existing "+ ___ ವಿಮೆ" blank).
- **2026-08-03** — Gokak fixes + per-user region: a3 ಸ್ಥಳ now prints `bank.place`
  constant (ಗೋಕಾಕ), was operator-typed branch_name. New `User.region` column
  (Kannada, default ಗೋಕಾಕ; settable via /register) prints at a4 ವಲಯ via
  `computed.region_kn`. `app.bank_name` prints nowhere in the packet. Also fixed
  a stray Korean "주소" in the form's Kannada notice (→ ವಿಳಾಸ).
- **2026-08-02** — Previous-loan block (borrower type = Old): 8 inputs stored
  as `previous_loans` JSON; PDF page 9 section 9) reordered (amounts first,
  attachment items last), ಲಗತ್ತಿಸಿದೆ → ಲಗತ್ತಿಸಿರಿ, values bound.

## Next / pending

- **Deployment strategy PLANNED (2026-08-03)** — see `docs/DEPLOYMENT_PLAN.md`
  (per-branch `branch_config.json` instead of git branches; signed license
  w/ expiry as payment control, renewed by owner over SSH; daily no-PII
  heartbeat; standing access = Tailscale + Windows OpenSSH + RDP, owner is
  tailnet admin; remote one-file installer after the Gokak pilot; LAN
  hostname, no domain) and `docs/STARTUP_GUIDE.md`
  (proprietorship + Udyam + AMC template). Build order: branch-config refactor
  first (fold into hobli/taluk dropdown task), license+heartbeat only at first
  paid deal.
- **From CROP INCOME CHART.xlsx notes (item 7, partially done):** still pending
  — female-loanees dashboard card (count + caste categories 1–4), hobli
  dropdown (ಅರಬಾಂವಿ, ಕೌಜಲಗಿ, ಗೋಕಾಕ) + taluk dropdown (ಮೂಡಲಗಿ, ಗೋಕಾಕ)
  — make these read from branch config per the deployment plan,
  and "also add after this Last Name next line" (needs owner clarification).
  Note: form has no gender-independent "female loanee" flag beyond `gender`.
- **Owner actions (URGENT):** Sync fork + redeploy Railway — the demo lacks
  the landdevdetails auto-migration (LAND_DEV apps 500 there), the
  dead-session fixes (tester lost a typed application to this on
  2026-08-06), and all LAND_DEV correction work. Verify the deployed hash
  is ≥ 7bfae56 after redeploy. Also: revoke the two GitHub PATs pasted in
  chat; tell the tester their lost application must be retyped AFTER the
  redeploy.
- **From owner, parked:** "XYZ prices" discussion (owner will explain);
  a restricted crop list for the pre/post-dev dropdowns may come later
  (for now ALL 31 chart crops show, per owner 2026-08-07); decision needed
  if the bank objects to extent = acres+guntas/40 (24.5) vs their ಆಯವ್ಯಯ
  sheet's literal acre.gunta decimal (24.2).
- (done 2026-08-06) LAND_DEV moratorium: 12-month initial period + (N−2)
  yearly kantus, kantu = loan/(N−2) — confirmed by reference p10+p14 and
  Ap4, implemented in render_service + b4 + ld4.
- Bank sign-off on printed Tractor + LAND_DEV packets → then replicate
  templates+specs to SHEEP_40/20/10 → BULLOCK (shared pages already built;
  their UI forms don't exist yet and will copy the Tractor/LAND_DEV pattern).
- Optional: Railway volume for persistent demo data; Windows-VM validation of
  the GTK runtime before bank install; frontend dead-code sweep (PDFOverlay,
  tractor_map.js, TractorApplicationForm.jsx) — needs owner approval.
- (done 2026-08-03) KN/EN language switch for the form; PDF ದಿ removal was NOT
  done in backend BANK constants on purpose (matches bank's reference PDFs) —
  frontend-only per owner. If the bank wants it off the print, edit
  `schemas/common_fields.py` BANK name_line1.
