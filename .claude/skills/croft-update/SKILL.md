---
name: croft-update
description: Draft the next weekly Croft / Oxford Street Education bankruptcy newsletter. Pulls new docket activity from CourtListener (auto-buying missing PDFs from PACER), summarizes each new filing in plain language, and writes a YYYYMMDD.md draft in the repo root for Jake to edit before sending.
---

# Croft Bankruptcy Newsletter — drafting skill

You are drafting the next issue of Jake Walker's weekly plain-language newsletter on the Oxford Street Education / Croft School Chapter 7 bankruptcy. Your output is a single markdown file in the repo root, written in Jake's established voice (see `20260611.md` for the canonical example). Jake will edit your draft before sending.

This file is the full operating manual. Follow it step by step.

## Case constants

- CourtListener docket ID: **73445594**
- CourtListener docket URL: https://www.courtlistener.com/docket/73445594/oxford-street-education-llc/
- Court: **mab** (U.S. Bankruptcy Court, District of Massachusetts)
- Docket number: **26-11334**
- PACER case ID: **525527**
- PACER case identifier used in storage URLs: `gov.uscourts.mab.525527`
- Document URL template:
  `https://storage.courtlistener.com/recap/gov.uscourts.mab.525527/gov.uscourts.mab.525527.{ENTRY_NUMBER}.0.pdf`
  (For attachments, the third numeric segment is the attachment number — e.g. `.8.1.pdf` for attachment 1 of entry 8.)
- Petition date: **June 5, 2026**
- Chapter: **7**
- Judge: **Janet E. Bostwick**
- Interim Trustee: **Harold B. Murphy**
- 341 Meeting of Creditors: **July 14, 2026, 9:00 AM** (Zoom)
- Proof of Claim deadline: **August 14, 2026**
- Government Proof of Claim deadline: **December 2, 2026**

Surface the 341 meeting, the Proof of Claim deadline, and any newly noticed hearings in the "Key dates & deadlines" section of every issue until each date has passed. Surface the Government POC deadline only when ≤30 days away or relevant.

## Repo files you will read/write

Relative to the repo root (`/Users/jake/Projects/croftbankruptcyemails`):

- `20260611.md` — voice/format anchor. Re-read before drafting.
- `YYYYMMDD.md` — issue files. Newest = highest date. You write a new one.
- `.croft-state.json` — machine state. You read and update it.
- `.croft-conflicts.json` — conflict watch rules. You read it.
- `GLOSSARY.md` — terms already defined in prior issues. You read it; you may append new terms after drafting.
- `DISCLOSURE.md` — standing footer text. Append verbatim to every draft.
- `CONFLICTS.md` — human-readable mirror of `.croft-conflicts.json`. No need to read at draft time.

Do **not** `git add` or `git commit`. Stop after writing the file.

## Step-by-step procedure

### 1. Resolve cutoff and issue number

Read `.croft-state.json`. Set:

- `issue_number = last_issue_number + 1`
- `today` = today's date in `YYYY-MM-DD`
- `output_filename = <YYYYMMDD>.md` (today)

If `.croft-state.json` is missing, bootstrap: list issue files matching `^\d{8}\.md$` in the repo root, parse the highest `# Croft Bankruptcy Update #N` heading among them, set `last_issue_number = N` and `last_issue_date = filename date`. Set `last_max_docket_entry_id = 0` and `last_max_date_modified = "1970-01-01T00:00:00Z"` so the first run pulls everything.

If `output_filename` already exists, write `<YYYYMMDD>-v2.md` instead and warn Jake in your final summary.

### 2. Pull new docket activity

Call `mcp__claude_ai_CourtListener__call_endpoint` with:

```
endpoint_id: "docket-entries"
query: {
  docket: 73445594,
  date_modified__gte: <last_max_date_modified minus 24 hours, ISO-8601 UTC>,
  order_by: "recap_sequence_number",
  fields: ["id", "entry_number", "date_filed", "date_modified",
           "recap_sequence_number", "description"]
}
num_results: 100
```

Do **not** request the `recap_documents` field — it can return ~500 KB of nested data per call and blow the token budget. Fetch document detail separately in step 3.

Paginate with `get_more_results` until exhausted. Then locally filter to entries where:

- `id` is not already in `covered_entry_ids` AND
- the entry is not in `deferred_entry_ids` flagged as "skip permanently" (currently we never set this, but reserve the flag).

Belt-and-suspenders pass: also call once with `date_filed__gte: <last_issue_date - 7d>` to catch backdated filings whose `date_modified` precedes the watermark. Union the results with the first call. Mark any entry whose `date_filed` is older than `last_issue_date` as "filed [date] (appeared this week)" in the draft.

If the union is empty, jump to step 6 ("zero new filings" path).

### 3. Fetch document detail per entry

For each new entry, call:

```
endpoint_id: "recap-documents"
query: {
  docket_entry: <entry_id>,
  fields: ["id", "document_number", "attachment_number", "is_available",
           "is_sealed", "page_count", "description", "pacer_doc_id",
           "document_type", "ocr_status"]
}
num_results: 50
```

The nested `recap_documents` returned with a docket entry can be a count (integer) rather than a list — always re-query here. An entry can have zero `recap_documents` (text-only minute entry); handle this in step 5.

### 4. Auto-buy missing PACER documents

Collect every `recap_documents` row across the new entries where `is_available` is false AND `is_sealed` is false. Drop any whose docket-entry `description` matches `^(Certificate of Service|Notice of Appearance|Receipt|BNC Certificate)` — these are routine and not worth purchasing.

Safety caps:

- If the resulting buy list has **more than 10 documents**, stop and ask Jake which to purchase before continuing. Quote the descriptions and page counts.
- If any individual document has `page_count > 50` (or page_count is null), stop and ask Jake before buying that one. Cost rule of thumb: $0.10/page, capped at $3.00.
- If a document has `document_type == 2` (attachment) and the parent doc is sealed, skip it.

For each approved purchase, **re-GET that `recap-documents` row right before buying** to confirm `is_available` is still false (the bulk RECAP archive may have caught up since the bulk fetch). If now available, skip the purchase.

Submit the buy via `call_endpoint` with `endpoint_id: "recap-fetch"`:

```
endpoint_id: "recap-fetch"
query: {
  request_type: 2,
  recap_document: <doc_id>
}
```

`recap-fetch` is async. The response includes a fetch-job `id` and `status`. Status semantics:
- `1` queued, `4` in progress, `2` success, `3`/`5`/`6`/`7` various failures.

Poll by re-calling `recap-fetch` with `query: {id: <job_id>, fields: ["id", "status", "date_completed", "message"]}` every ~8 seconds for up to 3 minutes. On `status: 2`, re-GET the `recap-documents` row and confirm `is_available: true`. On failure, log the message in your final summary, leave the entry uncovered, and continue with other entries.

Note: It is possible the MCP `call_endpoint` tool only supports GETs. If a POST-style submission fails with a method-not-allowed error, stop and ask Jake to run the fetch manually via the CourtListener web UI, then re-run the skill once the docs are available.

Maintain a running cost estimate; print it to the conversation before each purchase ("Purchasing entry 18 (4 pages, est. $0.40) ...").

### 5. Read and summarize each new filing

For each new entry that now has at least one available document:

- Start with `mcp__claude_ai_CourtListener__search_document` against targeted phrases — at minimum: `"Friends of JP"`, `"JP Educational Collective"`, `"Croft Bond"`, `"tuition"`, `"trustee"`, `"motion"`, `"order"`, `"hearing"`, plus any case-specific terms surfacing in the description.
- Only call `read_document` for the full text when the search hits don't tell you enough, or when the filing is short (≤10 pages) and warrants a full read.
- Capture: who filed, what they asked for or reported, any specific dates or dollar amounts, any new hearing date, any party other than the debtor.

For text-only minute entries (no `recap_documents`), summarize from the entry `description` field alone. These are often hearing notices and orders that matter — do not skip them.

Group entries by topic. Multiple filings about the same motion or proceeding belong in one `### h3` subsection.

### 6. Draft `YYYYMMDD.md`

Use `20260611.md` as the structural anchor. Required sections in order:

1. `# Croft Bankruptcy Update #<N> — <Month Day, YYYY>` (today, e.g. "June 18, 2026")
2. **Opening paragraph.** Use Jake's first-person framing — open with "My goal for this newsletter is to provide a regular plain-language summary of what has happened in the Oxford Street Education/Croft School bankruptcy, and to flag key deadlines and dates." Then add one sentence naming the date range covered this issue (e.g. "This issue covers filings from June 11 through June 17, 2026."). Do **not** rewrite the mission sentence into third person or a more formal register.
3. `## First, the reassuring part` — **only if** the docket actually supports it. If the week has no real good news, omit the section entirely. Never manufacture comfort.
4. `## Key dates & deadlines` — bulleted list with bold dates. Always include the 341 meeting and the Proof of Claim deadline until each date has passed. Add new hearings, objection deadlines, or status conferences as they appear. Use the format `* **Month Day, YYYY, time — Event name**`. **Do not include the government Proof of Claim deadline (Dec 2, 2026) unless it is ≤30 days away OR a filing this week specifically references it.** Likewise, do not include Zoom meeting IDs, passcodes, or call-in numbers in this section — those are in the official Notice of Bankruptcy and don't need to be republished.
5. `## What got filed this week` — one `### h3` per topical cluster, not per entry. Multiple related filings collapse into a single subsection with multiple inline PDF links. Routine items go into a final "Other filings" bullet list, matching the style of issue #1. **Include a routine item in "Other filings" only if it (a) directly affects families, (b) sets or moves a deadline, (c) introduces a new party that readers should recognize, or (d) discloses something material (e.g. a conflict, a fee, a vote).** Drop pure procedural items (BNC certificates of mailing, certificates of service, declarations re: electronic filing, fee receipts, routine matrix verifications) unless one of (a)–(d) applies.
6. `## Key Links` — public docket link (constant), the initial petition (`...mab.525527.1.0.pdf`), and the Notice of Bankruptcy (`...mab.525527.7.0.pdf`). Add foundational filings only as they emerge (e.g. a confirmed reorganization plan).
7. Append the contents of `DISCLOSURE.md` verbatim as the final block, preceded by a `---` rule.

**Audience-orienting parentheticals.** When introducing an entity, group, or term the average family reader might not recognize, include a brief parenthetical clarifying who/what it is — even when it isn't strictly necessary to the legal point. Match the style of issue #1's "(This was the separate parent group in JP focused on the future of the school.)". One per new entity per issue is enough.

#### "Quiet week" path (zero new filings)

Write the issue anyway. Structure:

- `# Croft Bankruptcy Update #<N> — <Month Day, YYYY>`
- One-paragraph "Quiet week" intro stating the date range and that nothing new was filed.
- `## Key dates & deadlines` — same as normal.
- `## Key Links` — same as normal.
- DISCLOSURE.md footer.

Skip "First, the reassuring part" and "What got filed this week". Print "Quiet week — no new filings between [last_issue_date] and [today]" to the conversation. Let Jake decide whether to send.

#### Massive dump (>20 new entries)

Group aggressively. Surface the 5–8 most consequential topical clusters as `### h3` sections. Push the rest into the closing "Other filings" bullet list. If anything is truly low-priority, add its `id` to `deferred_entry_ids` in state and note in your final summary that you deferred N items.

### 7. Inject conflict disclosures

After drafting (but before writing the file), scan **only the drafted prose of each `### h3` subsection** (not the broader case context) against the `match` terms in `.croft-conflicts.json`. A rule fires for a subsection only if a match term **literally appears in that subsection's prose** (case-insensitive, word-boundary). The source-text snippets you pulled from filings inform the rule only if you actually wrote a phrase containing a match term into the prose.

When a rule fires, insert the rule's `disclosure` line as the first paragraph under that subsection's heading. Insert at most **one** disclosure per `### h3`; if multiple rules match, prefer the most specific (e.g. `friends-of-jp` over `prepaid-tuition`).

**Do not inject preemptively.** Do not add a disclosure because the broader case touches a conflict, or because the trustee's actions could plausibly affect a conflicted interest. The trigger is literal text in your draft of that subsection — nothing else. If you are unsure whether a disclosure applies, leave it out and rely on the standing footer from `DISCLOSURE.md`.

If the opening paragraph (above the first `### h3`) itself contains a match term, promote the disclosure to its own block immediately under the opening paragraph. Otherwise do not promote.

Log the count: "Inserted N conflict disclosures."

### 8. Drafting voice rules

These are hard rules — enforce them on your own output before writing the file:

- **Paraphrase only.** No quoted span longer than 6 consecutive words from any filing.
- **Procedural verbs only.** Filed, asked, granted, denied, scheduled, disclosed, ordered, set, withdrew, served. Banned: "likely means", "appears to", "suggests", "indicates", "could imply", "may signal".
- **Prefer everyday phrasing over lawyerly phrasing when the meaning is identical.** Match the conversational register of issue #1. Concrete substitutions:
  - "an initial okay" not "granted on a preliminary basis"
  - "signed off fully" or "the court agreed" not "granted the motion in full"
  - "asked the court for permission" not "filed an emergency motion seeking limited operating authority"
  - "answered questions under oath" not "testified"
  - "agreed to pay" not "stipulated to pay"
  - First names + last name on first mention of a person, then just "the trustee" / "the judge" / etc. — don't keep repeating full name + title.
- **Define legal terms in bold on first use per issue.** Before defining, check `GLOSSARY.md`. If the term is already there, reuse the established phrasing verbatim; don't re-explain. If you introduce a new term, append it to `GLOSSARY.md` after writing the issue.
- **No commentary, no prediction, no characterization of motive or fault.**
- **Dates spelled out.** "June 12" or "June 12, 2026" (use the bare-month form when the year is obvious from context, matching issue #1's style) — never "6/12/26" or "June 12th". For times, use "9:00 a.m." (lowercase, periods).
- **Link every factual claim** to the underlying PDF using the document URL template. If you're citing a hearing rather than a filed document, link to the order that scheduled or memorialized it.
- **Sentences ≤ 25 words. No semicolons. Em-dashes are fine.**
- **One `### h3` per topic**, not per docket entry.
- **Numbers and dates verbatim from filings.** When summarizing dollar amounts, write `$75,000` not "approximately seventy-five thousand dollars".
- **Lead with reassurance only when the record supports it.** Otherwise omit "First, the reassuring part".
- **Never give legal advice.** If a reader question would normally need legal advice (e.g. "how do I file a Proof of Claim"), point to public resources (the official notice, the U.S. Trustee page) rather than instructing.

### 9. Update state and stop

After writing the issue file:

1. Update `.croft-state.json`:
   - `last_issue_number = <new N>`
   - `last_issue_date = <today YYYY-MM-DD>`
   - `last_issue_filename = <output_filename>`
   - `last_max_docket_entry_id = max(id across all entries seen this run, including via belt-and-suspenders pass)`
   - `last_max_date_modified = max(date_modified across all entries seen this run)`
   - `covered_entry_ids`: append every new entry id from this run
   - `covered_recap_document_ids`: append every recap_document id used in the draft
   - Leave `deferred_entry_ids` alone unless you explicitly added to it
2. If you defined any new legal terms in the issue, append them to `GLOSSARY.md` with the same definition you used in the draft and a `*(first defined in Issue #<N>, <date>)*` tag.

Print a short summary to the conversation:

- Issue number drafted
- Date range covered
- Number of new entries covered, and which ids
- PACER purchases made and total estimated cost
- Number of conflict disclosures inserted
- Anything deferred or failed

**Do not output the draft body to the conversation. The file is the deliverable.** Tell Jake the filename and remind him to review before committing.

## Failure modes — explicit handling

- **Sealed entry (`is_sealed: true`):** one-line mention in the draft ("A sealed filing was entered on [date]; the contents are not public."). No link. No purchase.
- **Multi-attachment entry:** link the main doc; sub-bullet substantive attachments only. Skip routine attachments (Certificate of Service, etc.).
- **OCR-failed PDF (`ocr_status: 3`):** note the document is image-only and could not be summarized in detail; link to it anyway and briefly describe what the docket entry's `description` field says it is.
- **PACER fetch job fails:** log the failure status and message in your final summary, leave the entry as a one-line "filed but not yet available" mention in the draft, and continue.
- **Same-day re-run:** write `<YYYYMMDD>-v2.md` instead of overwriting. Warn Jake.
- **CourtListener API down or hard error:** stop. Tell Jake what failed. Do not write a partial issue file.

## One-time setup (already done — do not repeat)

- A docket alert subscription for docket 73445594 was created on 2026-06-13 (alert id 360661). Do not re-subscribe.
- `.croft-state.json` was bootstrapped from issue #1 on 2026-06-13. Do not re-bootstrap unless the file is missing.

## Style anchor (canonical voice)

Re-read `20260611.md` before drafting. Match its sentence length, its definitional pattern (bold term + plain-language explanation), its link density (every factual claim links to a PDF), and its closing structure (Key Links block).

---

## Appendix: useful CourtListener MCP tool signatures

- `mcp__claude_ai_CourtListener__call_endpoint` — GET against any endpoint. POST behavior for `recap-fetch` is to be confirmed at runtime; on method errors, fall back to asking Jake.
- `mcp__claude_ai_CourtListener__get_endpoint_schema` — use if a query returns a 400 / extra_forbidden error suggesting the schema has drifted.
- `mcp__claude_ai_CourtListener__search_document` — targeted phrase search inside a document; use first.
- `mcp__claude_ai_CourtListener__read_document` — paginated full read; use only when search hits aren't enough.
- `mcp__claude_ai_CourtListener__get_more_results` — paginate any list query.
- `mcp__claude_ai_CourtListener__subscribe_to_docket_alert` — already called; do not call again.
