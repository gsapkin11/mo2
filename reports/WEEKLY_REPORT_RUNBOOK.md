# Weekly Mo2 Deal Status Report — Runbook

This runbook is executed by a scheduled Claude Code routine every **Wednesday
9:00 AM America/Chicago**. It produces the "Mo2 Properties — Deal Status
Report" PDF, matching the approved reference report
(`reports/template/Mo2_Deal_Status_Report_20260624_reference.pdf`).

Audience of the report: **Kenny & Michael Motew**. Prepared by: **Grant
Sapkin**.

> **Source confidentiality (required).** The report pulls from Grant's internal
> deal-tracking system, but the **delivered report must never name that system**
> (no "WildroseOS", no "monday.com", no CRM/tool brand anywhere in the PDF, the
> footer, or the caption). The report reads as Grant's own prepared status
> report. This runbook and repo are internal-only; the confidentiality rule is
> about the rendered PDF and the message that delivers it.

---

## 1. Report window

- The report covers the **7 days ending at run time** (previous Wednesday → this Wednesday), in America/Chicago.
- The report is titled "Week of {Month D, YYYY}" using the **run date**.

## 2. Data source (the deals endpoint)

All deal data comes from one read-only JSON endpoint. **Do not read or write
monday.com** — the CRM tools are not used by this report and may be
unauthenticated in the routine's session.

```bash
curl -sSL --retry 1 --max-time 30 \
  "https://wildrose-northside-folio.lovable.app/api/public/deals-report?token=a69c68a5aa567a1c481b4050a2ee117df94bd46bc4691db2&track=Mo2,Both&days=7"
```

Returns:

```json
{
  "generated_at": "…",
  "deals": [{
    "address": "1342 W Randolph St",
    "track": "Mo2",                    // "Mo2" | "GS" | "Both"
    "stage": { "name": "Screened", "kind": "open" },  // kind: open | won | lost
    "price": 15313616,                 // number, USD (may be null)
    "units": 23,                       // may be null
    "market": "Fulton Market",         // may be null
    "verdict": "Pursue",               // Pursue | Pass | …
    "thesis": "…",                     // the analytical narrative (ACTIVE / CLOSED WON / lessons)
    "notes": "Broker: … · Email … · Group: …",
    "next_step": "…",                  // the current open action (may be null)
    "next_step_due": "2026-07-30",     // may be null
    "broker_name": "…",                // may be null — fall back to parsing `notes`
    "created_at": "…", "updated_at": "…",
    "recently_updated": true           // true if the deal changed inside the `days` window
  }]
}
```

**Field → report mapping:**

| Report field | Source |
|---|---|
| Deal name | `address` |
| Stage | `stage.name` |
| Value | `price` → `$X.XM` (`~$X.XM` if clearly an estimate); omit if null |
| Units | `units` (append to stage line, e.g. `23 units`) |
| Counterparty | `broker_name`; if null, parse the `Broker: NAME` prefix out of `notes` |
| Headline | synthesize 1–2 sentences from `thesis` + `notes` + `next_step` — the week's key development and the current gating item |
| Open task / next step | `next_step` (+ `next_step_due`) |
| Recently completed | milestone lines mined from `thesis` (e.g. `CLOSED WON (…)` entries) |
| This week's activity | deals with `recently_updated: true` |

The endpoint is already filtered to **Mo2 + Both** tracks. Never include a deal
with `track: "GS"` even if one slips through.

## 3. Select what's in scope

1. **Active pipeline** (`stage.kind === "open"`): always included.
   - `recently_updated: true` → full deal section (headline, next step, completed items).
   - `recently_updated: false` → one line in the "No Movement This Week" quiet strip — never a full section.
2. **Closed / Passed** (`stage.kind === "won"` / `"lost"`): include **only if
   `recently_updated` is true** (it moved during the window) — a short "Closed" /
   "Passed" wrap-up section that week; otherwise exclude.
3. **Order** by imminence/importance (imminent closings first), then by
   `price` descending. Use the `.stagetag` banner for states like
   `CLOSING FRIDAY 7/10`, `CLOSED 6/30`, `NEW`.

If **every** in-scope deal has `recently_updated: false`, the report is still
sent: all deals go in the quiet strip under a "No movement across the pipeline
this week" note (per the template's conventions) — do not send an empty report,
and do not fabricate movement.

## 4. Compose the report (match the reference PDF exactly)

**Header:** `MO2 PROPERTIES` / `Deal Status Report — {Month D, YYYY}` (date in
the title, per Grant 7/8/26) /
`Week of {date} • Prepared by Grant Sapkin • For Kenny & Michael Motew`.

**Per deal section:**

- **Deal name** (`address`, e.g. "222 S Morgan").
- **Stage line:** `Stage: {Stage}{ -> milestone/date if one is imminent} | Value: ${X.X}M | {units} units | Counterparty: {broker}`. Omit any part whose source is null.
- **Headline:** 1–2 sentences synthesizing the week's most important development and the current gating item, written from `thesis` / `notes` / `next_step`. This is the analysis layer, not a data dump.
- **Task table** (columns `Task | Status | Notes`): the endpoint has no per-task
  subitems, so use the deal's `next_step` as a single open-task row —
  `Task = "Next step"`, status dot `● In Progress`, `Notes = next_step (due {next_step_due})`.
  If `next_step` is null, omit the table and let the headline carry the state.
- **Recently completed:** `✓ item` list — milestones mined from `thesis`
  (executed docs, closings, approvals, funds, confirmed key dates). Omit if none.

**Decisions Needed (conditional):** the endpoint carries no structured
"decision needed" flag. Only add this section if a `thesis`/`notes`/`next_step`
explicitly calls for a Kenny/Michael decision; otherwise omit the section
entirely.

**Footer (every page):** `Mo2 Properties • Confidential` and the report date
only. **No data-source/CRM/tool name** — see the confidentiality rule above.

**Tone/quality bar:** terse, specific, decision-oriented — names, dates,
dollar figures. Never invent facts. Do not include GS-track deals.
Spell out legal/deal abbreviations in full (e.g. "Amended & Restated LLC
agreement", not "A&R LLC agreement") — Grant's request 7/8/26.

**Length limit (hard requirement from Grant, 7/8/26): the report must fit
~2 pages.** To stay under:

- Full deal sections ONLY for deals with meaningful activity this week (`recently_updated`).
- Quiet deals go in the one-line-per-deal "No Movement This Week"
  (`.quiet` block) — never a full section.
- Max ~4–5 task rows per deal; notes ≤ 2 short sentences; merge minor items.
- Order sections by imminence/importance (imminent closings first).
- Closed deals: full section the week they close and while material post-close
  items remain, then drop to the quiet strip / off entirely.
- Passed deals: omit unless the pass itself is the week's news (one section
  max, headline only).

**Grant's corrections override the endpoint.** If Grant supplies deal updates in
chat before or after a run, they are the source of truth for that report —
regenerate and re-archive. New deals he names that aren't in the system yet get
a section from his notes, marked "not yet tracked."

## 5. Render to PDF

1. Fill `reports/template/report-template.html` (copy it, replace the
   placeholder tokens, repeat the deal-section block per deal).
2. Render with the pre-installed Chromium:

   ```bash
   /opt/pw-browsers/chromium --headless --no-sandbox --disable-gpu \
     --no-pdf-header-footer --print-to-pdf={out}.pdf {filled}.html
   ```

3. Name the file `Mo2_Deal_Status_Report_{YYYYMMDD}.pdf` (run date).
4. Sanity-check the PDF (open/read it) before delivering: header date
   correct, every in-scope deal present, no placeholder tokens left, and **no
   internal system/CRM name anywhere in the rendered report**.

## 6. Deliver & archive

1. **Send the PDF to Grant** via `SendUserFile` with `status: "proactive"` and
   a 2–3 sentence caption summarizing the week (deals moved, closings, items
   needing decisions). The caption must not name the internal data source
   either.
2. **Archive:** commit the PDF and the filled HTML to
   `reports/archive/{YYYY-MM-DD}/` on branch
   `claude/weekly-mo2-deal-routine-c0sinm` and push
   (`git push -u origin claude/weekly-mo2-deal-routine-c0sinm`, retry with
   backoff on network errors).

## 7. Failure handling

- If the deals endpoint errors, times out, returns non-JSON, or returns
  `401`/an empty `deals` array unexpectedly: retry once (`--retry 1` plus one
  manual re-run). If it still fails, **tell Grant the run failed and why** — do
  not send an empty or fabricated report.
- Do **not** fall back to monday.com or any other source. The endpoint above is
  the only data source; a source outage is a reported failure, not a reason to
  switch tools.
- If the JSON shape has drifted (a mapped field is missing), report the specific
  field that changed to Grant, update this runbook's mapping table if the change
  is stable, commit, and continue the run.
