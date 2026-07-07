# Weekly Mo2 Deal Status Report — Runbook

This runbook is executed by a scheduled Claude Code routine every **Wednesday
9:00 AM America/Chicago**. It produces the "Mo2 Properties — Deal Status
Report" PDF, matching the approved reference report
(`reports/template/Mo2_Deal_Status_Report_20260624_reference.pdf`).

Audience of the report: **Kenny & Michael Motew**. Prepared by: **Grant
Sapkin**. Source of truth: **monday.com CRM**.

---

## 1. Report window

- The report covers the **7 days ending at run time** (previous Wednesday → this Wednesday), in America/Chicago.
- The report is titled "Week of {Month D, YYYY}" using the **run date**.

## 2. Data sources (verified IDs — re-verify with `get_board_info` if any call errors)

**Board:** `Deals` — board ID **9292248624**, workspace `CRM` (11207281),
monday.com account `mo2-hq.monday.com`.

**Deal (item) columns:**

| Column | ID | Notes |
|---|---|---|
| Stage | `deal_stage` | status labels: `Screen + Underwrite` (id 5), `LOI` (id 3), `Pre-DD + PSA` (id 15), `DD + Financing` (id 0), `Closed Won` (id 1), `HOLD` (id 4), `Pass` (id 2) |
| Track | `color_mm2w4pjz` | status labels: `Mo2` (id 1), `GS` (id 2), `Both` (id 3) |
| Deal Value | `deal_value` | number, $ |
| Owner/Broker | `text_mkz4wvf4` | counterparty name shown in the report |
| Owner | `deal_owner` | people |
| Verdict Reasoning | `long_text_mm2wqztn` | context only |

**Tasks (subitems, board 9292249036) columns:**

| Column | ID | Notes |
|---|---|---|
| Status | `status` | `Working on it` (0), `Done` (1), `Stuck` (2), `Waiting on Others` (3), `In Progress` (4) |
| Date | `date0` | due/target date |
| Owner Type | `dropdown_mm3rpgcq` | Grant / Kenny / Michael / Architect / Attorney / Broker / GC / Seller / Lender / Other |
| Decision Needed | `boolean_mm3r9f1z` | checkbox — flags tasks needing Kenny/Michael input |

## 3. Gather data (monday.com MCP tools)

1. **Mo2-track deals** — `get_board_items_page` on board `9292248624` with
   `includeColumns: true`, `includeSubItems: true`, and filter:

   ```json
   [{"columnId": "color_mm2w4pjz", "compareValue": [1, 3], "operator": "any_of"}]
   ```

   ⚠️ Status filters take **label IDs** (Mo2 = 1, Both = 3), not index values
   and not label text. Paginate with `nextCursor` until `has_more` is false.

2. **In-scope deals for the report:**
   - Stage in `DD + Financing`, `Pre-DD + PSA`, `LOI`, `Screen + Underwrite`, `HOLD` → active pipeline, always included.
   - Stage `Closed Won` or `Pass` → include **only if the stage changed during the report window** (report it as "Closed" / "Passed" with a short wrap-up), otherwise exclude.
   - Order sections by deal maturity: DD + Financing → Pre-DD + PSA → LOI → Screen + Underwrite → HOLD, then by Deal Value descending within a stage.

3. **Week's updates** — `get_updates` with `objectType: "Board"`,
   `objectId: "9292248624"`, `includeItemUpdates: true`, `fromDate`/`toDate` =
   report window. Paginate (`limit: 25–100`, increment `page`). Responses can
   be very large (synced emails land here) — if a result is persisted to a
   file, mine it with `jq`/python. Keep only updates whose `item_id` belongs
   to an in-scope deal.

4. **Week's field changes** — `get_board_activity` on board `9292248624` with
   `fromDate` = window start and `itemIds` = the in-scope deal IDs. This
   catches Stage transitions, new/completed tasks, and column edits that
   don't appear as updates.

## 4. Compose the report (match the reference PDF exactly)

**Header:** `MO2 PROPERTIES` / `Deal Status Report` /
`Week of {date} • Prepared by Grant Sapkin • For Kenny & Michael Motew`.

**Per deal section:**

- **Deal name** (item name, e.g. "222 S Morgan").
- **Stage line:** `Stage: {Stage}{ -> milestone/date if one is imminent} | Value: ${X.X}M | Counterparty: {Owner/Broker}`. Format value as `$13.8M` / `~$1.05M` (use `~` when the value is an estimate); omit `Value` if empty.
- **Headline:** 1–2 sentences synthesizing the week's most important development and the current gating item. Written from the week's updates/emails/activity — this is the analysis layer, not a data dump.
- **Task table** (columns `Task | Status | Notes`): open subitems (not Done). Status shown with a colored dot + label (● In Progress, ● Working on it, ● Stuck, ● Waiting on Others). Notes: concise synthesis of the latest state from updates/emails, with attribution and dates where useful (e.g. "per Aaron 6/22").
- **Recently completed:** `✓ item` list — subitems marked Done during the window plus notable milestones from updates/activity (executed docs, approvals, funds, key dates confirmed).

**Decisions Needed (conditional):** if any in-scope task has the
`Decision Needed` checkbox checked, add a final section listing
`{Deal} — {Task}: what's being decided / what Kenny & Michael need to weigh in on`.
Skip the section entirely if nothing is flagged.

**Footer (every page):** `Mo2 Properties • Confidential`, `Source: monday.com CRM`, report date.

**Tone/quality bar:** terse, specific, decision-oriented — names, dates,
dollar figures. Never invent facts; if the CRM is silent on a deal all week,
say "No movement this week." Do not include GS-track deals.

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
   correct, every in-scope deal present, no placeholder tokens left.

## 6. Deliver & archive

1. **Send the PDF to Grant** via `SendUserFile` with `status: "proactive"` and
   a 2–3 sentence caption summarizing the week (deals moved, closings, items
   needing decisions).
2. **Archive:** commit the PDF and the filled HTML to
   `reports/archive/{YYYY-MM-DD}/` on branch
   `claude/weekly-mo2-deal-routine-c0sinm` and push
   (`git push -u origin claude/weekly-mo2-deal-routine-c0sinm`, retry with
   backoff on network errors).

## 7. Failure handling

- If monday.com MCP tools are unavailable, retry once; if still failing, tell
  the user the run failed and why — do not send an empty or fabricated report.
- If a column/label ID has drifted, call `get_board_info` on `9292248624`,
  update this runbook with the corrected IDs, commit, and continue the run.
