---
name: text-tickets
description: "Create or manage a tracking ticket in the tickets/ folder. Use when: you want to create a ticket, track a task, add something to remember, move a ticket between states, list open tickets, check ticket status, or manage the ticket lifecycle. Tickets follow a lifecycle: todo → doing → done → verified."
---

# Create / Manage Ticket

Create, list, move, or close tracking tickets stored under `tickets/`.

## Ticket Folder Structure

Each ticket is a **folder** under `tickets/<state>/`, not a flat file. The folder contains the ticket definition plus any working state files.

```
tickets/<state>/<number>-<headline>/
  ticket.md          — The ticket definition (frontmatter + content)
  plan.md            — Implementation plan (created when work starts)
  progress.md        — Progress tracking (created when work starts)
  decisions.md       — Design/technical decisions (created when work starts)
  research/          — Research notes, references (created as needed)
  artifacts/         — Generated files, code snippets (created as needed)
```

- **States**: `do`, `doing`, `done`, `verified`
  - Note: The `do` folder represents the "todo" state (named `do` so it sorts first in file listings)
- **Number**: Auto-incremented integer (zero-padded to 4 digits, e.g. `0001`)
- **Headline**: Short kebab-case summary (e.g. `fix-contact-form`, `add-ga4-tracking`)

Example: `tickets/do/0001-build-seo-visualization-tool/ticket.md`

## Ticket Content Template

Each `ticket.md` file follows this Markdown template:

```markdown
---
id: <number>
created: <YYYY-MM-DD>
updated: <YYYY-MM-DD>
status: <todo|doing|done|verified>
client: <client-name-or-general>
priority: <low|medium|high|critical>
deadline: <YYYY-MM-DD or blank>
depends_on: <ticket numbers, comma-separated or none>
source: <email|internal|task>          # where the request came from
client_contact: <email address>        # who to notify (customer requests)
notified: <true|false|n/a>             # client notified of ticket creation
completion_notified: <true|false|n/a>  # client notified ticket is done
---

# <Human-readable title>

## Description
<What needs to be done, why, and any context>

## Acceptance Criteria
- [ ] Criterion 1
- [ ] Criterion 2

## Notes
<Any updates, blockers, or progress notes go here>
```

## Workflow

### Creating a New Ticket

**Deduplicate before creating (standing voice-ticket rule).** When the request comes from a voice note or message, list all state folders first. If an existing ticket already covers the request, do NOT create a duplicate — add the missing context to that ticket's Notes and update its `updated` field. Only create a new ticket when nothing existing covers it. Note the relationship to related open tickets in `depends_on`/Notes rather than duplicating their scope.

1. **Determine the next ticket number**:
   - List all directories under `tickets/` using `list_dir` on each state folder (`do/`, `doing/`, `done/`, `verified/`)
   - Extract the highest number from folder names matching `<number>-<headline>/`
   - Increment by 1, zero-pad to 4 digits (e.g., if highest is `0013`, next is `0014`)

2. **Gather requirements from the user**:
   - **Headline/Title** — what the ticket is about
   - **Client** — which client this relates to (or "general" if cross-cutting)
   - **Priority** — low, medium, high, or critical (default: medium)
   - **Description** — what needs to be done and why
   - **Acceptance Criteria** — how do we know it's done?

   If the user provides a short request (e.g. "add GA4 to brentwood"), infer the rest using context from the workspace where possible. Then ask targeted follow-up questions to fill in any gaps. Use the `vscode_askQuestions` tool to ask 1-3 relevant questions at a time.

3. **Ask follow-up questions for context** (only if information is missing or ambiguous):
   Use these as a guide — pick the 1-3 most relevant questions based on what the user already provided:

   - **Scope**: "Is this for a single page or the whole site?"
   - **Deadline**: "Is there a target date this needs to be completed by?"
   - **Dependencies**: "Does this depend on anything else being done first?"
   - **Related work**: "Is this connected to another ticket or piece of work?"
   - **Specific pages**: "Which pages should be affected?"
   - **Target audience**: "Is there a specific audience or market this targets?"
   - **Success metric**: "How will we measure if this is successful? (e.g., form submissions, traffic increase, ranking improvement)"
   - **References**: "Is there an example from another client site or a reference URL?"
   - **Blockers**: "Are there any known blockers or constraints I should be aware of?"
   - **Languages**: "Does this need to apply to translated versions of the site too?"

   Default values when the user doesn't specify:
   - Priority: `medium`
   - Client: inferred from context, or `general` if unclear
   - Deadline: none (blank)
   - Dependencies: none listed

3. **Generate the folder name**:
   - Headline: convert title to kebab-case (lowercase, spaces → hyphens, remove special chars)
   - Number: the next auto-incremented number
   - Folder: `<number>-<headline>` (e.g., `0014-add-ga4-to-brentwood`)

4. **Create the folder and ticket.md** — use `create_directory` then `create_file` at `tickets/do/<number>-<headline>/ticket.md`
   - **If the ticket came from a customer email** (`source: email`): also preserve the
     original so it can be referred to later. Write the full MIME to `email.raw`,
     a readable copy to `email.txt`, and any attachments to `artifacts/attachments/`.
     Set `client_contact:` to the customer's email address.

5. **Commit the ticket** using `run_in_terminal`:
   ```bash
   cd /Users/marketingagency/marketing-agency && git add tickets/do/<number>-<headline>/ && git commit -m "Ticket #<number>: <human-readable title>"
   ```

6. **Notify the client** (only when `source: email` — i.e. the ticket was created from a
   customer request). See **Notifying the Client** below. Set `notified: true` in the
   frontmatter once the email is confirmed delivered, then commit that change.
   - Skip this step for `source: internal` or `source: task` (set `notified: n/a`) —
     those are internal work, not customer requests.

7. **Report back** the created ticket with its number, folder path, `ticket.md` location,
   and confirmation of whether the client was notified.

### Notifying the Client

When a ticket is created from a **customer request** (email, form submission, etc.),
email the customer to let them know a tracking ticket was created for their request and
that you'll update them as you work through it. Do this **right after the ticket is
committed** so the confirmation is never lost.

**Send with the Gmail API** (service-account + domain-wide delegation — the same pattern
used elsewhere in this repo). Read from `scripts/seo/service-account.json`, delegate to
`michel@thinksmart.life`, and send:

```python
import base64, email.message, warnings
warnings.filterwarnings("ignore")
from google.oauth2 import service_account
from googleapiclient.discovery import build

USERS = "michel@thinksmart.life"
base = service_account.Credentials.from_service_account_file(
    "scripts/seo/service-account.json",
    scopes=["https://www.googleapis.com/auth/gmail.send"])
g = build("gmail", "v1", credentials=base.with_subject(USERS))

msg = email.message.EmailMessage()
msg["From"] = "Michel Laclé <michel@thinksmart.life>"
msg["To"] = "<client_contact>"
msg["Subject"] = "Received — tracking ticket created for your request"
msg.set_content("""Hi <first name>,

Thanks for sending this over — I've received your request and set up a
tracking ticket for it so nothing gets lost.

Here's what I've logged:
  • <request summary>

I'll let you know as I work through it.

Thanks,
Michel
""")
raw = base64.urlsafe_b64encode(msg.as_bytes()).decode()
created = g.users().messages().send(userId="me", body={"raw": raw}).execute()
print("sent id=", created["id"])
```

**If you need the client's input to proceed**, the email must make unmistakably clear
the ball is in the client's court. For the request block(s) with an open question, use
this exact wording (same as the `client-email-intake` skill):

```
Before I can move on this one, I need your input: <the question>.
The next step is on your side — as soon as you reply, I get moving.
```

When there is no open question, the status line is: `Received — we're on it, I'll
update you as we progress.` Never pair that status line with a question in the same
block — pick the block that matches the ticket's state.

**Verification (required):** a successful `send()` call is not proof of delivery. Read the
sent message back with a **separate `gmail.readonly` credential** (the send credential
can't read) and confirm it carries the `SENT` label and the correct `To:` address before
reporting success. If the read-back fails, do **not** claim the client was notified.

**Batch requests:** if one customer email contains several distinct requests, either
create one ticket per request and list them all in a single confirmation email (preferred),
or create one ticket and list its sub-items. Don't send one confirmation email per ticket
in a burst.

**If the send fails** (scope, network, or invalid address): still finish the ticket and
report the failure plainly to the user — do **not** silently skip the notification or
mark `notified: true` when the email didn't go out.

### Notifying the Client of Completion

When a **customer-request ticket** (`source: email`) is moved to `done`, email the client
to let them know the ticket is complete. This is the counterpart to the creation
notification. The goal is that the client gets a clear, self-contained summary so they can
act on it without needing to remember what they originally asked for.

**The email must contain three things, in this order:**

1. **The original request, restated.** Start by reminding the client of what *they*
   requested — quote or closely paraphrase their own words from the source
   (`email.txt` / `email.raw` in the ticket folder). Clients frequently forget the exact
   scope of what they asked for; this re-anchors them. Don't assume they remember.

2. **What was actually done.** A concise, concrete list of the changes made. Name the
   specific deliverables, pages, files, or settings touched. Be specific enough that the
   client can verify it themselves.

3. **How to verify / what's live.** Tell the client where to look (URL, page, setting) and
   that it's live/complete. Offer to adjust if anything looks off.

**Ticket number in the subject (required):** the subject line must include the ticket
number in the format `Ticket #<number>` so that when the client replies, the thread is
unambiguously tied to a specific ticket for follow-up tracking. Use the same ticket number
used at creation so both emails (created + completed) share a traceable identifier.

**Send with the Gmail API** — same pattern as the creation notification
(`scripts/seo/service-account.json`, delegate to `michel@thinksmart.life`, `gmail.send`
scope). Adjust the body to the three-part structure above:

```python
import base64, email.message, warnings
warnings.filterwarnings("ignore")
from google.oauth2 import service_account
from googleapiclient.discovery import build

USERS = "michel@thinksmart.life"
base = service_account.Credentials.from_service_account_file(
    "scripts/seo/service-account.json",
    scopes=["https://www.googleapis.com/auth/gmail.send"])
g = build("gmail", "v1", credentials=base.with_subject(USERS))

msg = email.message.EmailMessage()
msg["From"] = "Michel Laclé <michel@thinksmart.life>"
msg["To"] = "<client_contact>"
# Ticket number MUST be in the subject for follow-up traceability
msg["Subject"] = "Ticket #<number> complete — <short headline>"
msg.set_content("""Hi <first name>,

Quick update — Ticket #<number> is done and live.

What you requested
  <Restate the client's original request in their own words / close paraphrase,
   pulled from email.txt in the ticket folder.>

What we did
  • <Concrete deliverable 1 — name the page/file/setting>
  • <Concrete deliverable 2>
  • <Concrete deliverable 3>

It's live now at <URL or location> — refresh and you'll see <what to look for>.

If anything looks off, let me know and I'll adjust.

Thanks,
Michel
""")
raw = base64.urlsafe_b64encode(msg.as_bytes()).decode()
created = g.users().messages().send(userId="me", body={"raw": raw}).execute()
print("sent id=", created["id"])
```

**Verification (required):** same rule as the creation notification — read the sent
message back with a **separate `gmail.readonly` credential** and confirm the `SENT` label
and correct `To:` address before reporting success. Then set `completion_notified: true`
in the frontmatter and commit.

**Pull the original request from the ticket folder:** for `source: email` tickets, read
`email.txt` (or `email.raw`) in the ticket folder to get the client's exact original words —
do not rely on memory or the `ticket.md` description alone, since the description may have
been condensed. This is what makes the "what you requested" section accurate.

**Skip for non-customer tickets:** `source: internal` or `source: task` tickets set
`completion_notified: n/a` — there's no customer to notify.

**If the send fails:** still complete the ticket move, but report the failure plainly and
do **not** mark `completion_notified: true` when the email didn't go out.

### Cross-references

Before citing another ticket number ("see #NNNN", "covers NNNN's promise", "depends on NNNN") in a new ticket, read that ticket's title/frontmatter and confirm it is actually the referenced subject. A remembered number can point at an unrelated ticket (e.g. citing #0031 — an Orubilla email ticket — as a reviews-page reference). When in doubt, omit the number and describe the relationship in words; wrong cross-references rot the record.

### Deduplicate incoming content against the live page

Before adding client-supplied content (reviews, photos, copy) to a page, scan the page's existing content for duplicates/near-duplicates. Clients frequently send content that is already on the page — or that was itself derived from placeholder copy on the page (real names grafted onto fake reviews), so the page can end up showing the same quote under two names. Compare incoming items against existing ones (exact + first-sentence match), and when they collide, keep the real named version and drop the duplicate — then note the collision in the ticket so the client's original ask ("keep the fake ones") can be reconciled deliberately rather than by accident.

**Standing policy — no fake reviews/testimonials.** Michel's standing rule (2026-09-10) overrides any client request to "keep the fake ones": placeholder/fabricated reviews and testimonials must not appear on client sites. If a reviews page contains invented testimonials (anonymous "X client" cards, generic praise with no verifiable author), remove them and keep only real named reviews — even when the client asked to keep them. Note the removal in the ticket and tell Michel.

**Standing policy — minimize PII on published pages.** Client names shown publicly (review authors, testimonials, case studies) should be displayed as **first name + last-name initial** (e.g. "Courtney R."), not full names — even when the client sent the full name in the source material. The full name stays in the ticket/email record; the published site shows the abbreviated form.

### Scope discipline (important)
Work **only what the ticket's description and acceptance criteria ask**. Do not expand
scope: don't add extra acceptance criteria, don't surface adjacent decisions ("what should
we do with the rest of the page?") as if they were part of this ticket, and don't make
changes beyond the literal request. If related follow-on work exists, it belongs in its own
ticket (e.g. a different change-request number) — note the cross-reference in Notes but leave
the work itself alone. If a client or the user says "just do what the ticket says," close the
ticket out to exactly that and notify them of completion. Real correction: on a "remove the
fake-review disclaimer" ticket, an out-of-scope decision about the remaining placeholder
reviews was added and had to be pulled back.

### Listing Tickets

When asked to list or review tickets:

1. List all directories under each state folder (`do/`, `doing/`, `done/`, `verified/`) using `list_dir`
2. For each ticket folder, read `ticket.md` to extract frontmatter: id, status, client, priority, title
3. Present a summary table:

| # | Status | Client | Priority | Title |
|---|--------|--------|----------|-------|
| 0001 | todo | brentwood | high | Add GA4 tracking |

4. If a specific state is requested (e.g. "show do tickets" or "show todo tickets"), filter to the `do/` folder only

### Moving a Ticket Between States

When asked to move a ticket (e.g. "start working on ticket 3" or "mark ticket 5 as done"):

1. **Find the ticket folder** — list directories under each state folder, look for folder starting with `<number>-`
2. **Read `ticket.md`** inside the folder to get the full path and frontmatter
3. **Update the `status` and `updated` fields** in `ticket.md` using `replace_string_in_file`
4. **Move the entire folder** using `run_in_terminal` with `mv <old_path> <new_path>`:
   - "start working" / "begin" / "pick up" → move to `tickets/doing/`
   - "done" / "complete" / "finish" → move to `tickets/done/`
   - "verify" / "verified" / "approved" → move to `tickets/verified/`
   - "put back" / "pause" / "block" → move to `tickets/do/`
5. **Append a note** under the `## Notes` section in `ticket.md` with the date and state transition
6. **If moving to `done` and the ticket is a customer request** (`source: email`): send the
   completion notification — see **Notifying the Client of Completion** below. Set
   `completion_notified: true` in the frontmatter once the email is confirmed delivered.
   Skip for `source: internal` / `source: task` (set `completion_notified: n/a`).
7. **Commit the change** using `run_in_terminal`:
   ```bash
   cd /Users/marketingagency/marketing-agency && git add tickets/ && git commit -m "Ticket #<number>: moved to <new_state>"
   ```

### Updating a Ticket

When asked to add notes, update criteria, or modify a ticket:

1. **Find and read** the `ticket.md` file inside the ticket folder
2. **Apply changes** using `replace_string_in_file`:
   - Add progress notes under `## Notes`
   - Check off acceptance criteria (`- [ ]` → `- [x]`)
   - Update `updated` date in frontmatter
3. **Commit the change** using `run_in_terminal`:
   ```bash
   cd /Users/marketingagency/marketing-agency && git add tickets/ && git commit -m "Ticket #<number>: <brief description of update>"
   ```
4. **Report** what was changed

### Adding Working State Files

When work begins on a ticket, the implementing agent creates additional files inside the ticket folder:

- `plan.md` — implementation plan and task breakdown
- `progress.md` — current progress, what's done, what's next
- `decisions.md` — design/technical decisions with rationale
- `research/` — research notes and reference materials
- `artifacts/` — generated files, code snippets, temporary outputs

These files live alongside `ticket.md` in the same folder. They are the working scratchpad for the ticket.

### Committing to the Agency Repo

- **Stage explicit paths, never `git add -A`.** The parent repo holds client-submodule pointers; `-A` sweeps in unrelated drift — a client submodule sitting at a detached HEAD, or scratch files left in ticket artifact folders. If a submodule pointer diff shows up in a commit, verify the pinned commit is on that repo's remote `main` before committing it; otherwise leave the drift in the worktree (it re-appears in `git status`, which is the honest state) and mention it in the report.
- Keep each commit scoped to one ticket's work so history stays auditable per client request.

### Closing an Umbrella / Multi-Item Ticket

Some tickets (brainstorm lists, "update the whole website" requests) bundle many items, and by close time most of the work has shipped under *other* tickets. Don't close these on vibes — audit item by item:

1. **Audit every item** against what actually shipped: mark each done (and which ticket/commit did it), superseded, client's-call, or deferred. Write the item-by-item resolution into `## Notes` so the record survives.
2. **Hunt for trivial stragglers before closing.** Items that look "obviously handled elsewhere" are the ones that slip — verify each in the live code before crediting it. If a small item is genuinely open, finish it (push + verify) rather than closing with it dangling.
3. **Create follow-up tickets for deferred work** so nothing is lost: one ticket per remaining item of real scope, `source: internal`, cross-referencing the original ticket in Notes.
4. **The completion email must reflect the split** — "X of N are live, Y are your decision, Z are in the works" — not "complete" when it's a partial close. Restate the original list so the client sees the full picture.

### Closing / Deleting a Ticket

Tickets are generally kept for history. Only delete if explicitly requested by the user. Prefer moving to `verified/` as the final state.

## Edge Cases

- **Multiple tickets in one request**: voice notes often bundle several asks ("ticket number one … ticket number two …"). Assign consecutive numbers in the order asked, cross-reference related tickets in Notes/`depends_on` (a later ticket often builds on an earlier ticket's capability), and commit them together in a single commit whose message lists all numbers — the one-commit-per-ticket rule exists to keep unrelated work out of a commit, not to split a single request.
- **Duplicate headlines**: If a headline already exists, append a disambiguator (e.g. `-2`)
- **Missing number in folder name**: If a folder doesn't match the `<number>-<headline>` pattern, skip it when computing the next number
- **Empty state folders**: No special handling needed — listing simply shows 0 tickets for that state

## Examples

**User**: "Create a ticket to add Google Analytics to Beauty By Yulis"
→ Creates `tickets/do/0014-add-google-analytics-to-beauty-by-yulis/ticket.md`

**User**: "Start working on ticket 2"
→ Moves folder from `tickets/do/0002-.../` to `tickets/doing/0002-.../`, updates status to `doing`

**User**: "Ticket 2 is done"
→ Moves folder from `tickets/doing/0002-.../` to `tickets/done/0002-.../`, updates status to `done`

**User**: "I've verified ticket 2 looks good"
→ Moves folder from `tickets/done/0002-.../` to `tickets/verified/0002-.../`, updates status to `verified`

**User**: "Show me all open tickets"
→ Lists all ticket folders in `do/` and `doing/` states
