# Mo2 Reporting Automation

Automation assets for Mo2 Properties recurring reports, generated from the
monday.com CRM.

## Contents

| Path | Purpose |
|------|---------|
| `reports/WEEKLY_REPORT_RUNBOOK.md` | Step-by-step runbook the scheduled routine follows every Wednesday to produce the Mo2 Deal Status Report |
| `reports/template/report-template.html` | HTML/print template used to render the report to PDF (matches the approved June 24, 2026 format) |
| `reports/template/Mo2_Deal_Status_Report_20260624_reference.pdf` | The approved reference report supplied by Grant — the visual/content standard every weekly report must match |
| `reports/archive/` | Generated weekly reports, one folder per report date |

## Schedule

A Claude Code routine runs every **Wednesday at 9:00 AM Chicago time**, scans
all updates on Mo2-track deals in the monday.com CRM from the past week, and
delivers the report as a PDF.
