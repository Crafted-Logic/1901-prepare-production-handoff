---
name: 1901-prepare-production-handoff
description: Packages one 1901 design's live queue, source, and readiness evidence into a read-only production handoff.
---

# 1901 Prepare Production Handoff

Builds a structured, read-only production handoff package for exactly one
1901 Main Street design. It gathers the live evidence needed for production
from three read-only checks, summarises every blocker, and packages the
design state so Walter can hand it to a later production agent or present it
to Jody for approval. It changes nothing: no Sheets, no Drive, no render
fields, no files, no rendering, no Printify, no Etsy, no publishing.

Core rule: **VERIFY, DON'T ASSUME.** The package contains only facts read in
the current run. Memory, chat history, prior runs, and guesses are not
production evidence. A blocker is never hidden and a failure to read a source
is preserved in the package, not papered over.

## When to Use

Trigger on requests such as:

- "Prepare the production handoff for 1901-003"
- "Package 1901-017 for production"
- "What does the next production step need for this design?"
- "Give Jody the handoff summary for <design_id>"

Do not use it to look up a row alone (`1901-read-idea-queue`), to resolve a
source alone (`1901-resolve-production-source`), or to judge readiness alone
(`1901-validate-readiness`). This skill coordinates evidence from those three
capabilities and does not replace them.

## Input

Exactly one `design_id`, for example `1901-003`. Trim surrounding whitespace
from the user's input, and nothing else. Match by exact string equality: no
case folding, no fuzzy match, no closest id, no normalising `1901-3` into
`1901-003`. No usable single id: `INVALID_RECORD`.

## Procedure

Run every step you can; stop only when nothing downstream could be evidence.

1. **Queue.** Run `1901-read-idea-queue` for the id in this run. Its `FOUND`
   output is the live record. `NOT_FOUND`, `DUPLICATE_ID`, `INVALID_REQUEST`,
   or `SCHEMA_WARNING`: blocker `INVALID_RECORD`. `SOURCE_UNAVAILABLE`:
   blocker `SOURCE_UNAVAILABLE`. Either way, stop after this step; the
   downstream checks have nothing live to read.
2. **Production source.** Run `1901-resolve-production-source` in this run.
   Map its result to a blocker: `DOCUMENTATION_CONFLICT` stays
   `DOCUMENTATION_CONFLICT`; `AMBIGUOUS` becomes `AMBIGUOUS_SOURCE`;
   `UNVERIFIED` becomes `SOURCE_UNVERIFIED`; `NOT_FOUND` becomes
   `MISSING_SOURCE`; `SOURCE_UNAVAILABLE` and `INVALID_RECORD` keep their
   names. `RESOLVED` fills `approved_source_file`.
3. **Readiness.** Run `1901-validate-readiness` in this run, in Production
   Validation Mode only. A result whose `validation_mode` is `test_fixture`
   is not evidence; treat it as `SOURCE_UNAVAILABLE`. Its `reason_code`, when
   not `READY`, is a blocker. Every additional failed check in its `checks`
   array is also a blocker under that check's own code (`status` gives
   `STATUS_NOT_APPROVED`, `human_decision` gives `MISSING_HUMAN_APPROVAL`,
   `HUMAN_REVISE`, or `HUMAN_REJECT` by its value, `soft_ip` gives
   `SOFT_IP_BLOCK`, `open_items` gives `OPEN_ITEM_BLOCK`, `documentation`
   gives `DOCUMENTATION_CONFLICT`, `budget` gives `BUDGET_BLOCK`).
4. **Collect.** Put only the live evidence those checks returned into
   `evidence`, one concise `{source, fact}` per fact. One blocker per code;
   do not invent a new code when an upstream reason code already describes
   the problem.
5. **Package.** Fill the JSON object below from the queue record and the
   upstream results. Decide `handoff_status` and `next_action` by the rules
   in the next two sections.
6. **Return** the object. Make no change anywhere. Do not perform the next
   action.

## Handoff Status

| handoff_status | use when |
|---|---|
| `READY_FOR_PRODUCTION_HANDOFF` | Queue record `FOUND`, production source `RESOLVED`, readiness `READY`, and `blockers` empty. Means the package is complete enough for the next production step. **It does not authorise publishing.** |
| `BLOCKED` | Any blocker needs a human decision or a governing-rule decision: `DOCUMENTATION_CONFLICT`, `HUMAN_REJECT`, `SOFT_IP_BLOCK`, `OPEN_ITEM_BLOCK`, `BUDGET_BLOCK`, `AMBIGUOUS_SOURCE`, `UNKNOWN_BLOCKER` |
| `NOT_READY` | Only routine required fields or steps are incomplete: `MISSING_HUMAN_APPROVAL`, `HUMAN_REVISE`, `MISSING_SOURCE`, `SOURCE_NOT_MASTER`, `SOURCE_UNVERIFIED`, `STATUS_NOT_APPROVED` |
| `INVALID_RECORD` | The queue record is missing, duplicated, or materially malformed |
| `SOURCE_UNAVAILABLE` | Required live Sheets, Drive, or governing documentation could not be read |

Precedence when blockers of several kinds coexist: `INVALID_RECORD`, then
`SOURCE_UNAVAILABLE`, then `BLOCKED` if any blocker is in the `BLOCKED` set,
else `NOT_READY`. The status reflects the worst problem present; the lower
ones stay listed in `blockers`.

## Priority and Next Action

`blockers` holds every material blocker found, sorted by this priority.
`next_action` is one concise sentence for the highest-priority blocker only.

| # | code | next_action |
|---|---|---|
| 1 | `INVALID_RECORD` | Correct the Idea Queue record so exactly one well-formed row carries this design_id, then re-run. |
| 2 | `SOURCE_UNAVAILABLE` | Restore read access to the live Idea Queue, Drive, and governing documentation, then re-run. |
| 3 | `DOCUMENTATION_CONFLICT` | Jody must resolve the governing production-source rule before production can continue. |
| 4 | `HUMAN_REJECT` | human_decision is REJECT; no production action. Jody or Ame decide whether the design is revised or retired. |
| 5 | `SOFT_IP_BLOCK` | Ame must resolve or withdraw the active soft-IP concern before production can continue. |
| 6 | `OPEN_ITEM_BLOCK` | Resolve the unresolved Open Item that bears on the next production step and record the decision. |
| 7 | `BUDGET_BLOCK` | Jody must raise the cap under the governing budget rule or hold the design. |
| 8 | `MISSING_HUMAN_APPROVAL` | Jody or Ame must record the human_decision. |
| 9 | `HUMAN_REVISE` | human_decision is REVISE; complete the revision and obtain a fresh APPROVE before production. |
| 10 | `MISSING_SOURCE` | Record the exact approved source file in render_source_path through an authorized workflow. |
| 11 | `AMBIGUOUS_SOURCE` | Jody or Ame must record which candidate file is the approved artwork before a source can be set. |
| 12 | `SOURCE_NOT_MASTER` | Replace render_source_path with the exact approved master through an authorized workflow. |
| 13 | `SOURCE_UNVERIFIED` | Record the approval linkage for the source file in the governing production record, then re-run. |
| 14 | `STATUS_NOT_APPROVED` | Move the design to status Approved through the normal approval workflow. |
| 15 | `UNKNOWN_BLOCKER` | Review the unclassified blocker in the readiness result and record a decision. |
| 16 | `READY` | No action required; package is ready for the next production step. |

Walter reports the action. Walter never performs it.

## Output

Return exactly one JSON object:

```json
{
  "design_id": "1901-003",
  "handoff_status": "READY_FOR_PRODUCTION_HANDOFF | BLOCKED | NOT_READY | INVALID_RECORD | SOURCE_UNAVAILABLE",
  "queue": { "result": "", "row_number": null, "status": "", "human_decision": "", "art_path": "", "render_source_path": "", "render_status": "", "render_qa": "" },
  "production_source": { "result": "", "source_rule_status": "", "resolved_file": null, "candidates": [], "human_action_required": null },
  "readiness": { "ready": false, "state": "", "reason_code": "", "message": "", "human_action_required": null },
  "handoff_package": {
    "approved_source_file": null,
    "design_metadata": { "concept": "", "season": "", "style": "", "vibe": "" },
    "printify_id": "", "etsy_url": "", "notes": ""
  },
  "blockers": [ { "code": "", "source": "", "detail": "", "human_action_required": null } ],
  "next_action": "",
  "evidence": [ { "source": "", "fact": "" } ]
}
```

Shape rules:

- `queue` carries the live values from the `FOUND` record: `""` for a blank
  cell, `null` for a column absent from the live schema. When no record was
  read, `result` names why and the rest stay at their empty defaults.
- `production_source` and `readiness` carry the upstream results verbatim.
  They stay at their empty defaults when the queue step stopped the run.
- `approved_source_file` is populated only when the production source result
  is `RESOLVED`, as `{drive_file_id, name, url, folder, mime_type}`;
  otherwise `null`.
- `design_metadata`, `printify_id`, `etsy_url`, and `notes` come from the
  queue record as read. They are context for the handoff, not evidence of
  readiness.
- `blockers[].source` names the check that surfaced it:
  `1901-read-idea-queue`, `1901-resolve-production-source`,
  `1901-validate-readiness`, or `request`.
- `evidence` entries look like `{"source": "Idea Queue row 4", "fact":
  "status = Approved; human_decision = blank"}`, `{"source":
  "1901-resolve-production-source", "fact": "DOCUMENTATION_CONFLICT: ..."}`,
  or `{"source": "1901-validate-readiness", "fact": "reason_code =
  MISSING_HUMAN_APPROVAL"}`.

## Authority

`READY_FOR_PRODUCTION_HANDOFF` does not mean publish. It does not authorise
rendering, Printify changes, Etsy changes, publishing, shipping, or storefront
changes. Jody remains the final publication and shipping authority, and human
approval remains authoritative over every result in this package.

## Never Do

This skill is packaging and reporting only. Never:

- write to Google Sheets, write `render_source_path`, or change
  `human_decision`, `status`, `render_status`, or `render_qa`
- write to Google Drive, create folders, create manifests, or copy or move
  artwork
- generate or edit images, upscale artwork, or render listing images
- call Printify, call Etsy, or publish anything
- infer approval, resolve human decisions, or resolve governing-document
  conflicts
- modify Open Items or budget records
- mutate any external system
- perform the `next_action`, or hand the package onward on your own

## Pitfalls

- **Readiness says READY but the source is AMBIGUOUS.** Not a handoff.
  `AMBIGUOUS_SOURCE` is a blocker and the status is `BLOCKED`.
- **Three problems, one sentence.** List all three in `blockers`, sorted by
  priority. `next_action` names only the first. Never drop the rest.
- **The queue read failed but you know the row from earlier.** Stop after
  step 1 with `SOURCE_UNAVAILABLE`. Nothing downstream can run on memory.
- **Readiness was run as a test fixture.** Not production evidence. Treat it
  as `SOURCE_UNAVAILABLE` and say so in the blocker detail.
- **Ready package, so publish.** No. The package goes to the next production
  step or to Jody. Publication is Jody's decision.
- **A new code would describe it better.** Use the upstream reason code that
  already exists. New codes break downstream consumers.

## Examples

Values are fixtures showing shape and logic. Upstream results are shown as the
sibling skills would return them. Live output always carries what was read.

### 1. READY_FOR_PRODUCTION_HANDOFF

Queue `FOUND`, source `RESOLVED`, readiness `READY`, no blockers.

```json
{
 "design_id": "1901-003",
 "handoff_status": "READY_FOR_PRODUCTION_HANDOFF",
 "queue": {
  "result": "FOUND",
  "row_number": 4,
  "status": "Approved",
  "human_decision": "APPROVE",
  "art_path": "https://drive.google.com/file/d/1FILEc3stamp000000000000000000001/view",
  "render_source_path": "Masters/1901-003_redraw-c-stamp.png",
  "render_status": "",
  "render_qa": ""
 },
 "production_source": {
  "result": "RESOLVED",
  "source_rule_status": "VERIFIED",
  "resolved_file": {
   "drive_file_id": "1FILEc3stamp000000000000000000001",
   "name": "1901-003_redraw-c-stamp.png",
   "url": "https://drive.google.com/file/d/1FILEc3stamp000000000000000000001/view",
   "folder": "1901-003 Artwork (folder id 1AbCdEfGhIjKlMnOpQrStUvWxYz012345)",
   "mime_type": "image/png",
   "reason": "named by art_path and tied to the approval note"
  },
  "candidates": [],
  "human_action_required": null
 },
 "readiness": {
  "ready": true,
  "state": "Ready",
  "reason_code": "READY",
  "message": "All readiness checks passed on read evidence. 1901-003 may enter the next production step.",
  "human_action_required": null
 },
 "handoff_package": {
  "approved_source_file": {
   "drive_file_id": "1FILEc3stamp000000000000000000001",
   "name": "1901-003_redraw-c-stamp.png",
   "url": "https://drive.google.com/file/d/1FILEc3stamp000000000000000000001/view",
   "folder": "1901-003 Artwork (folder id 1AbCdEfGhIjKlMnOpQrStUvWxYz012345)",
   "mime_type": "image/png"
  },
  "design_metadata": {
   "concept": "Autumn Porch Cat",
   "season": "Fall",
   "style": "Vintage",
   "vibe": "Cozy"
  },
  "printify_id": "",
  "etsy_url": "",
  "notes": "Approved: Redraw C stamp remaster"
 },
 "blockers": [],
 "next_action": "No action required; package is ready for the next production step.",
 "evidence": [
  {
   "source": "Idea Queue row 4",
   "fact": "status = Approved; human_decision = APPROVE; render_source_path = Masters/1901-003_redraw-c-stamp.png"
  },
  {
   "source": "1901-resolve-production-source",
   "fact": "RESOLVED: 1901-003_redraw-c-stamp.png"
  },
  {
   "source": "1901-validate-readiness",
   "fact": "reason_code = READY"
  }
 ]
}
```

### 2. NOT_READY: missing human_decision

Source resolved, but readiness fails on a blank `human_decision`.

```json
{
 "design_id": "1901-017",
 "handoff_status": "NOT_READY",
 "queue": {
  "result": "FOUND",
  "row_number": 18,
  "status": "Approved",
  "human_decision": "",
  "art_path": "https://drive.google.com/file/d/1FILEc3stamp000000000000000000001/view",
  "render_source_path": "Masters/1901-003_redraw-c-stamp.png",
  "render_status": "",
  "render_qa": ""
 },
 "production_source": {
  "result": "RESOLVED",
  "source_rule_status": "VERIFIED",
  "resolved_file": {
   "drive_file_id": "1FILEc3stamp000000000000000000001",
   "name": "1901-003_redraw-c-stamp.png",
   "url": "https://drive.google.com/file/d/1FILEc3stamp000000000000000000001/view",
   "folder": "1901-003 Artwork (folder id 1AbCdEfGhIjKlMnOpQrStUvWxYz012345)",
   "mime_type": "image/png",
   "reason": "named by art_path and tied to the approval note"
  },
  "candidates": [],
  "human_action_required": null
 },
 "readiness": {
  "ready": false,
  "state": "Not Ready",
  "reason_code": "MISSING_HUMAN_APPROVAL",
  "message": "Status is Approved but human_decision is blank. A human must record APPROVE before production.",
  "human_action_required": "Jody or Ame: set human_decision to APPROVE, REVISE, or REJECT for 1901-017."
 },
 "handoff_package": {
  "approved_source_file": {
   "drive_file_id": "1FILEc3stamp000000000000000000001",
   "name": "1901-003_redraw-c-stamp.png",
   "url": "https://drive.google.com/file/d/1FILEc3stamp000000000000000000001/view",
   "folder": "1901-003 Artwork (folder id 1AbCdEfGhIjKlMnOpQrStUvWxYz012345)",
   "mime_type": "image/png"
  },
  "design_metadata": {
   "concept": "Autumn Porch Cat",
   "season": "Fall",
   "style": "Vintage",
   "vibe": "Cozy"
  },
  "printify_id": "",
  "etsy_url": "",
  "notes": ""
 },
 "blockers": [
  {
   "code": "MISSING_HUMAN_APPROVAL",
   "source": "1901-validate-readiness",
   "detail": "Status is Approved but human_decision is blank. A human must record APPROVE before production.",
   "human_action_required": "Jody or Ame: set human_decision to APPROVE, REVISE, or REJECT for 1901-017."
  }
 ],
 "next_action": "Jody or Ame must record the human_decision.",
 "evidence": [
  {
   "source": "Idea Queue row 18",
   "fact": "status = Approved; human_decision = blank; render_source_path = Masters/1901-003_redraw-c-stamp.png"
  },
  {
   "source": "1901-resolve-production-source",
   "fact": "RESOLVED: 1901-003_redraw-c-stamp.png"
  },
  {
   "source": "1901-validate-readiness",
   "fact": "reason_code = MISSING_HUMAN_APPROVAL"
  }
 ]
}
```

### 3. BLOCKED: production-source DOCUMENTATION_CONFLICT

The governing source rule is unresolved. Readiness fails on the same point; one blocker, not two.

```json
{
 "design_id": "1901-003",
 "handoff_status": "BLOCKED",
 "queue": {
  "result": "FOUND",
  "row_number": 4,
  "status": "Approved",
  "human_decision": "APPROVE",
  "art_path": "https://drive.google.com/file/d/1FILEc3stamp000000000000000000001/view",
  "render_source_path": "Masters/1901-003_redraw-c-stamp.png",
  "render_status": "",
  "render_qa": ""
 },
 "production_source": {
  "result": "DOCUMENTATION_CONFLICT",
  "source_rule_status": "UNRESOLVED",
  "resolved_file": null,
  "candidates": [],
  "human_action_required": "Jody must resolve the governing production-source rule (approved master vs upscaled print file vs derived export) before this design can be resolved."
 },
 "readiness": {
  "ready": false,
  "state": "Blocked",
  "reason_code": "DOCUMENTATION_CONFLICT",
  "message": "The governing render-stage source rule cannot be identified, so the source master cannot be judged.",
  "human_action_required": "Jody: resolve the governing production-source rule."
 },
 "handoff_package": {
  "approved_source_file": null,
  "design_metadata": {
   "concept": "Autumn Porch Cat",
   "season": "Fall",
   "style": "Vintage",
   "vibe": "Cozy"
  },
  "printify_id": "",
  "etsy_url": "",
  "notes": "Approved: Redraw C stamp remaster"
 },
 "blockers": [
  {
   "code": "DOCUMENTATION_CONFLICT",
   "source": "1901-resolve-production-source",
   "detail": "DOCUMENTATION_CONFLICT: Governing docs do not settle whether the render stage consumes the approved master or the upscaled print file; the question is recorded as open.",
   "human_action_required": "Jody must resolve the governing production-source rule (approved master vs upscaled print file vs derived export) before this design can be resolved."
  }
 ],
 "next_action": "Jody must resolve the governing production-source rule before production can continue.",
 "evidence": [
  {
   "source": "Idea Queue row 4",
   "fact": "status = Approved; human_decision = APPROVE; render_source_path = Masters/1901-003_redraw-c-stamp.png"
  },
  {
   "source": "1901-resolve-production-source",
   "fact": "DOCUMENTATION_CONFLICT: Governing docs do not settle whether the render stage consumes the approved master or the upscaled print file; the question is recorded as open."
  },
  {
   "source": "1901-validate-readiness",
   "fact": "reason_code = DOCUMENTATION_CONFLICT"
  }
 ]
}
```

### 4. BLOCKED: soft-IP

Source resolved; readiness reports an active soft-IP concern from Ame.

```json
{
 "design_id": "1901-040",
 "handoff_status": "BLOCKED",
 "queue": {
  "result": "FOUND",
  "row_number": 41,
  "status": "Approved",
  "human_decision": "APPROVE",
  "art_path": "https://drive.google.com/file/d/1FILEc3stamp000000000000000000001/view",
  "render_source_path": "Masters/1901-003_redraw-c-stamp.png",
  "render_status": "",
  "render_qa": ""
 },
 "production_source": {
  "result": "RESOLVED",
  "source_rule_status": "VERIFIED",
  "resolved_file": {
   "drive_file_id": "1FILEc3stamp000000000000000000001",
   "name": "1901-003_redraw-c-stamp.png",
   "url": "https://drive.google.com/file/d/1FILEc3stamp000000000000000000001/view",
   "folder": "1901-003 Artwork (folder id 1AbCdEfGhIjKlMnOpQrStUvWxYz012345)",
   "mime_type": "image/png",
   "reason": "named by art_path and tied to the approval note"
  },
  "candidates": [],
  "human_action_required": null
 },
 "readiness": {
  "ready": false,
  "state": "Blocked",
  "reason_code": "SOFT_IP_BLOCK",
  "message": "Ame has an active soft-IP concern on the slogan. Production is blocked until she resolves it.",
  "human_action_required": "Ame: resolve or withdraw the soft-IP concern on 1901-040."
 },
 "handoff_package": {
  "approved_source_file": {
   "drive_file_id": "1FILEc3stamp000000000000000000001",
   "name": "1901-003_redraw-c-stamp.png",
   "url": "https://drive.google.com/file/d/1FILEc3stamp000000000000000000001/view",
   "folder": "1901-003 Artwork (folder id 1AbCdEfGhIjKlMnOpQrStUvWxYz012345)",
   "mime_type": "image/png"
  },
  "design_metadata": {
   "concept": "Autumn Porch Cat",
   "season": "Fall",
   "style": "Vintage",
   "vibe": "Cozy"
  },
  "printify_id": "",
  "etsy_url": "",
  "notes": "Approved: Redraw C stamp remaster"
 },
 "blockers": [
  {
   "code": "SOFT_IP_BLOCK",
   "source": "1901-validate-readiness",
   "detail": "Ame has an active soft-IP concern on the slogan. Production is blocked until she resolves it.",
   "human_action_required": "Ame: resolve or withdraw the soft-IP concern on 1901-040."
  }
 ],
 "next_action": "Ame must resolve or withdraw the active soft-IP concern before production can continue.",
 "evidence": [
  {
   "source": "Idea Queue row 41",
   "fact": "status = Approved; human_decision = APPROVE; render_source_path = Masters/1901-003_redraw-c-stamp.png"
  },
  {
   "source": "1901-resolve-production-source",
   "fact": "RESOLVED: 1901-003_redraw-c-stamp.png"
  },
  {
   "source": "1901-validate-readiness",
   "fact": "reason_code = SOFT_IP_BLOCK"
  }
 ]
}
```

### 5. INVALID_RECORD

No row carries the id. The run stops after the queue step.

```json
{
 "design_id": "1901-999",
 "handoff_status": "INVALID_RECORD",
 "queue": {
  "result": "NOT_FOUND",
  "row_number": null,
  "status": "",
  "human_decision": "",
  "art_path": "",
  "render_source_path": "",
  "render_status": "",
  "render_qa": ""
 },
 "production_source": {
  "result": "",
  "source_rule_status": "",
  "resolved_file": null,
  "candidates": [],
  "human_action_required": null
 },
 "readiness": {
  "ready": false,
  "state": "",
  "reason_code": "",
  "message": "",
  "human_action_required": null
 },
 "handoff_package": {
  "approved_source_file": null,
  "design_metadata": {
   "concept": "",
   "season": "",
   "style": "",
   "vibe": ""
  },
  "printify_id": "",
  "etsy_url": "",
  "notes": ""
 },
 "blockers": [
  {
   "code": "INVALID_RECORD",
   "source": "1901-read-idea-queue",
   "detail": "NOT_FOUND: no single well-formed row for 1901-999",
   "human_action_required": null
  }
 ],
 "next_action": "Correct the Idea Queue record so exactly one well-formed row carries this design_id, then re-run.",
 "evidence": [
  {
   "source": "1901-read-idea-queue",
   "fact": "result = NOT_FOUND"
  }
 ]
}
```

### 6. SOURCE_UNAVAILABLE

The Idea Queue could not be read. The run stops after the queue step.

```json
{
 "design_id": "1901-003",
 "handoff_status": "SOURCE_UNAVAILABLE",
 "queue": {
  "result": "SOURCE_UNAVAILABLE",
  "row_number": null,
  "status": "",
  "human_decision": "",
  "art_path": "",
  "render_source_path": "",
  "render_status": "",
  "render_qa": ""
 },
 "production_source": {
  "result": "",
  "source_rule_status": "",
  "resolved_file": null,
  "candidates": [],
  "human_action_required": null
 },
 "readiness": {
  "ready": false,
  "state": "",
  "reason_code": "",
  "message": "",
  "human_action_required": null
 },
 "handoff_package": {
  "approved_source_file": null,
  "design_metadata": {
   "concept": "",
   "season": "",
   "style": "",
   "vibe": ""
  },
  "printify_id": "",
  "etsy_url": "",
  "notes": ""
 },
 "blockers": [
  {
   "code": "SOURCE_UNAVAILABLE",
   "source": "1901-read-idea-queue",
   "detail": "the authoritative Idea Queue could not be read",
   "human_action_required": null
  }
 ],
 "next_action": "Restore read access to the live Idea Queue, Drive, and governing documentation, then re-run.",
 "evidence": [
  {
   "source": "1901-read-idea-queue",
   "fact": "SOURCE_UNAVAILABLE"
  }
 ]
}
```

### 7. Multiple blockers, one next_action

Blank `human_decision`, an unresolved Open Item, and two candidate source files. All three are listed by priority; `next_action` follows the Open Item.

```json
{
 "design_id": "1901-045",
 "handoff_status": "BLOCKED",
 "queue": {
  "result": "FOUND",
  "row_number": 46,
  "status": "Approved",
  "human_decision": "",
  "art_path": "https://drive.google.com/file/d/1FILEc3stamp000000000000000000001/view",
  "render_source_path": "",
  "render_status": "",
  "render_qa": ""
 },
 "production_source": {
  "result": "AMBIGUOUS",
  "source_rule_status": "VERIFIED",
  "resolved_file": null,
  "candidates": [
   {
    "drive_file_id": "1FILEc3stamp000000000000000000001",
    "name": "1901-003_redraw-c-stamp.png",
    "url": "https://drive.google.com/file/d/1FILEc3stamp000000000000000000001/view",
    "reason": "matches 'Redraw C stamp' in the approval note"
   },
   {
    "drive_file_id": "1FILEc3remas000000000000000000003",
    "name": "1901-003_redraw-c-stamp-remaster.png",
    "url": "https://drive.google.com/file/d/1FILEc3remas000000000000000000003/view",
    "reason": "matches 'remaster' in the approval note"
   }
  ],
  "human_action_required": "Jody or Ame: record which of the listed files is the approved artwork for this design, then re-run."
 },
 "readiness": {
  "ready": false,
  "state": "Blocked",
  "reason_code": "OPEN_ITEM_BLOCK",
  "message": "Open Item OI-102 (back print placement) is unresolved and bears on the next production step.",
  "human_action_required": "Resolve OI-102 for 1901-045 and record the decision, then re-run readiness."
 },
 "handoff_package": {
  "approved_source_file": null,
  "design_metadata": {
   "concept": "Autumn Porch Cat",
   "season": "Fall",
   "style": "Vintage",
   "vibe": "Cozy"
  },
  "printify_id": "",
  "etsy_url": "",
  "notes": "Approved: Redraw C stamp remaster"
 },
 "blockers": [
  {
   "code": "OPEN_ITEM_BLOCK",
   "source": "1901-validate-readiness",
   "detail": "Open Item OI-102 (back print placement) is unresolved and bears on the next production step.",
   "human_action_required": "Resolve OI-102 for 1901-045 and record the decision, then re-run readiness."
  },
  {
   "code": "MISSING_HUMAN_APPROVAL",
   "source": "1901-validate-readiness",
   "detail": "human_decision is empty",
   "human_action_required": null
  },
  {
   "code": "AMBIGUOUS_SOURCE",
   "source": "1901-resolve-production-source",
   "detail": "AMBIGUOUS: 2 candidate files remain",
   "human_action_required": "Jody or Ame: record which of the listed files is the approved artwork for this design, then re-run."
  }
 ],
 "next_action": "Resolve the unresolved Open Item that bears on the next production step and record the decision.",
 "evidence": [
  {
   "source": "Idea Queue row 46",
   "fact": "status = Approved; human_decision = blank; render_source_path = blank"
  },
  {
   "source": "1901-resolve-production-source",
   "fact": "AMBIGUOUS"
  },
  {
   "source": "1901-validate-readiness",
   "fact": "reason_code = OPEN_ITEM_BLOCK"
  }
 ]
}
```

## Verification

The skill worked if the reply is one JSON object in the shape above, every
`evidence` entry was read in this run, every blocker found is listed and
sorted by priority, `next_action` matches the highest-priority blocker,
`approved_source_file` is set only when the source is `RESOLVED`,
`READY_FOR_PRODUCTION_HANDOFF` appears only with an empty `blockers` list,
and no sheet cell, Drive file, render field, document, or external system
changed during the run.
