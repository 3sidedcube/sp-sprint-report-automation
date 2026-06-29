# SP Sprint Report — Weekly Automation

Automates the Thursday-morning refresh of the SP sprint report into the shared
Google Sheet. A scheduled GitHub Action queries Jira fresh each week, parses the
counts deterministically, and writes to two tabs.

**This was set up by a contractor whose engagement ends 22 July 2026.** The whole
design assumes the *code* stays put and only the *credentials* change hands. When
ownership transfers, you should not need to touch any code — only update the
repository secrets listed below. This document is the checklist for doing that.

---

## What it does, each Thursday 07:00 UTC

1. Runs four JQL queries against the open sprint of project `SP`:
   - `status = "Delivered"`  → **Needs Testing**
   - `status = "Finished"`   → **Awaiting PR**
   - `status in ("In Progress","Awaiting Response")` → **In Progress**
   - `status = "Open"`       → **To Do**
2. Counts each bucket directly from its own API response (nothing is carried
   forward between runs).
3. Writes to the sheet:
   - **'Current Ticket Status'** — appends a new dated column (e.g. `3rd July`)
     with Needs Testing, Awaiting PR, In Progress, iOS/Android/RN/BED/FED Pending,
     and JIRA Total. The **Unsolved ZD Total** cell is left blank for you to fill
     in manually (it is a Zendesk figure, not Jira).
   - **'Tickets By Project'** — overwrites the whole table: Project | Tickets |
     Security | To Do | In Progress | Awaiting PR | Delivered, sorted by total,
     with a Total row.

You can also trigger it any time from the **Actions** tab → *Weekly SP Sprint
Report* → **Run workflow**.

---

## Repository secrets (the only things that change at handover)

Set these under **Settings → Secrets and variables → Actions → Secrets**:

| Secret | What it is | Notes |
|---|---|---|
| `JIRA_BASE_URL` | `https://3sidedcube.atlassian.net` | Unlikely to change. |
| `JIRA_EMAIL` | The Atlassian account email the API token belongs to | **Change at handover.** Should be a service account, not a person who may leave. |
| `JIRA_API_TOKEN` | API token for that account | **Change at handover.** See "Rotating the Jira token". |
| `GOOGLE_SHEET_ID` | The sheet's ID (the long string in its URL between `/d/` and `/edit`) | Only changes if the sheet is recreated. |
| `GCP_SERVICE_ACCOUNT_KEY` | The **full JSON** of a Google service-account key | **Set once at setup.** See "Google service account". |

Optional, under **Variables** (not secrets):

| Variable | Default | Notes |
|---|---|---|
| `JIRA_PROJECT_KEY` | `SP` | Change only if reporting on a different project. |

> When the business decides who owns this, that person (or a shared service
> account) supplies a new `JIRA_EMAIL` + `JIRA_API_TOKEN`, an admin pastes them
> into the two secrets, and the next run uses them. No code change, no redeploy.

---

## One-time setup

### 1. Put this repo somewhere durable
Push these files to a repo **in the 3SC GitHub org** (not a personal account),
so it outlives any individual. Private is fine — Actions work in private repos.

### 2. Jira API token
1. The owning account signs in to Atlassian and creates a token at
   <https://id.atlassian.com/manage-profile/security/api-tokens>.
2. Put the account's email in `JIRA_EMAIL` and the token in `JIRA_API_TOKEN`.
3. The account needs Browse Projects permission on `SP`.

**Strongly prefer a service / bot account** over a personal login, so the
automation doesn't break when someone leaves. If 3SC IT can provision one, use it.

### 3. Google service account (for sheet write access)
1. In Google Cloud Console, create (or reuse) a project.
2. Enable the **Google Sheets API**.
3. Create a **service account**; under its Keys, **add key → JSON**, download it.
4. Paste the entire JSON file contents into the `GCP_SERVICE_ACCOUNT_KEY` secret.
5. Open the service account's email (looks like
   `name@project.iam.gserviceaccount.com`) and **share the Google Sheet with it
   as an Editor**, exactly like sharing with a colleague.

### 4. Confirm the sheet layout matches
The script assumes:
- **'Current Ticket Status'**: row 1 = date headers, rows 2–11 = the metrics in
  this order: Needs Testing, Awaiting PR, In Progress, iOS Pending, Android
  Pending, RN Pending, BED Pending, FED Pending, JIRA Total, Unsolved ZD Total.
  New weeks are appended as the next empty column.
- **'Tickets By Project'**: row 1 = header, data from row 2 down, columns A–G =
  Project, Tickets, Security, To Do, In Progress, Awaiting PR, Delivered.

If the tab names or row order change, update the constants near the top of
`src/refresh_sprint_report.py` (`STATUS_TAB`, `PROJECT_TAB`, and the row order in
`build_status_column`).

---

## Keepalive — why there's a timestamp commit each week

GitHub **disables a scheduled workflow after 60 days of no commit activity** in
the repo, and emails the admins when it does. For a once-a-week job that nobody
is actively babysitting, that's a real risk — so the workflow protects itself two
ways on every run:

1. **Timestamp commit** — writes `.keepalive/last-run.txt` and commits it, so the
   repo is never idle. (You'll see weekly `chore: keepalive timestamp` commits;
   that's expected and harmless.)
2. **API re-enable** — calls the GitHub API to re-enable this workflow, a no-op
   if it's already on. This is the reliable backstop, because a commit made by
   the built-in `GITHUB_TOKEN` does **not** always reset the 60-day timer.

**Important limit:** the keepalive only protects an *otherwise-healthy* job. If
the real refresh has been failing for weeks (e.g. the Jira token expired and
nobody rotated it), GitHub can still disable the workflow. If you ever notice the
Thursday updates have stopped:
- Go to **Actions → Weekly SP Sprint Report**. If it says the workflow is
  disabled, click **Enable workflow**, then **Run workflow** to test.
- Check the most recent run log for the underlying failure (usually credentials)
  and fix that first — re-enabling a broken job just fails again next Thursday.

If the weekly keepalive commits ever feel like noise, you can instead delete the
"Keepalive — timestamp commit" step and rely solely on the API re-enable step;
the doc above explains the trade-off.

---

## Rotating the Jira token (at handover or on expiry)1. New owner creates a token (step 2 above) under their / the service account.
2. Admin updates `JIRA_EMAIL` and `JIRA_API_TOKEN` secrets.
3. Trigger a manual run (Actions → Run workflow) to confirm it still writes.
4. Revoke the departing person's old token in their Atlassian profile.

---

## Troubleshooting

- **Run failed at the Jira step** — token expired or account lost SP access.
  Check the run log; rotate the token.
- **Run failed at the Sheets step** — service account no longer has Editor
  access (sheet re-shared / recreated), or `GOOGLE_SHEET_ID` is wrong. Re-share
  the sheet with the service-account email.
- **"JIRA Total != project grand total" warning** — expected if some tickets
  have no parent epic; those are excluded from the project table but still
  counted in JIRA Total. Investigate only if the gap is unexpected.
- **Counts look wrong** — every run logs the four bucket sizes at the top of the
  job output. Compare against the board. The script never reuses prior values, so
  a wrong number means a JQL/permission issue, not stale data.

---

## What the contractor cannot leave you
- A Jira token or Google key tied to a personal account that will be
  deprovisioned — these **must** be reissued under an owner who is staying.
- The decision of *who* that owner is. Until that's set, the safest interim is a
  shared service account for both Jira and Google.

## Files
```
.github/workflows/weekly-sprint-report.yml   # schedule + manual trigger
src/refresh_sprint_report.py                 # fetch, parse, write
requirements.txt                             # Python deps
HANDOVER.md                                  # this file
```
