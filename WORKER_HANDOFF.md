# Worker handoff — Data Fabric MacGyver / FS-S3 event

Portable briefing for a **new Cursor worker**. No secret values in this file.
Copy this repo (`riles_lab_test`), copy `.env.example` → `.env`, then fill STS / Splunk locally or from Infisical.

Written: 4 Nov 2026, 2:47 PM Australia/Sydney.
Source run: Cursor-hosted Cloud Agent `bc-372a1317-dfe6-447f-874e-19a9c0ea0765` (not a self-hosted worker). That VM does **not** transfer. Start a new agent on the target machine.

Download (branch `riles_lab_test`):

- This file: https://raw.githubusercontent.com/andrewkriley/datafabric-macgyver/riles_lab_test/WORKER_HANDOFF.md
- Secret catalog: https://raw.githubusercontent.com/andrewkriley/datafabric-macgyver/riles_lab_test/.env.example
- Infisical stub: https://raw.githubusercontent.com/andrewkriley/datafabric-macgyver/riles_lab_test/.infisical.yaml
- Clone: `git clone -b riles_lab_test https://github.com/andrewkriley/datafabric-macgyver.git`

---

## 1. What this event is

This live event is **not** the six-lab MacGyver SHOW.

| Field | Value |
|---|---|
| Event | Splunk - Federated Search Workshop with S3 and Glue |
| Content title | Becoming a Splunk Federated Search for Amazon S3 Master |
| Host | AWS Workshop Studio (browser), not a SHOW email pack |
| Type | Individual / test content |
| Level | Dashboard 200; welcome page says 300 |
| Region | `us-east-1` only |
| Start | 2 Nov 2026, 08:44 (Studio clock) |
| Duration | 72 hours from start |
| Join email | `andreril@cisco.com` |
| Dashboard | https://catalog.us-east-1.prod.workshops.aws/event/dashboard/en-US |
| Join URL | https://catalog.us-east-1.prod.workshops.aws/join?access-code=a97c-068433-82 |

Expected duration ~2 hours. Participants get Splunk **and** AWS labs.

### Studio lessons vs this repo

| Studio lesson | Repo path | Action |
|---|---|---|
| Getting Started / Architecture | `content/architecture.md` (FS-S3 slice only) | Read studio page; ignore MacGyver Labs 1/5/6 claims |
| Splunk Account Setup | none | **Skip create.** User already has a Splunk account |
| Lesson 1 CloudFront / first FS-S3 dataset | `content/lab3_using_splunk_fs_s3/lab3a_basics_of_fs_s3/` | Do this |
| Lesson 2 VPC Flow + Athena DDL | `content/lab3_using_splunk_fs_s3/lab3b_using_athena_ddl_for_aws_glue/` | Do this |
| Lesson 3 WAF + Splunk-managed catalog | `content/lab3_using_splunk_fs_s3/lab3c_spunk_managed_catalog/` | Do this (`spunk_` is a folder typo) |
| Lesson 4 FS-S3 use cases | not in repo | Do this in Studio |
| Lesson 5 time-based partitions | not in repo | Do this in Studio |
| MacGyver Labs 1, 2, 4, 5, 6 | repo only | **Out of scope on this stack** |

---

## 2. Bootstrap on `glam-cp` (Linux VM)

`trt-vm-cursor-worker-glam-cp` is already running on that box, registered **only** for `gitlab.home.therileys.team/therileys-team/glam-cp`. Same `machineId` as `trt-vm-cursor-worker-shedlife`. Do not send a `datafabric-macgyver` agent at the glam-cp worker name — Cursor will reject the repo mismatch.

On the VM, start a **dedicated** worker with its own data dir:

```bash
git clone -b riles_lab_test https://github.com/andrewkriley/datafabric-macgyver.git
cd datafabric-macgyver
cp .env.example .env
# fill AWS STS + Splunk locally; do not commit .env

agent worker \
  --name "trt-vm-cursor-worker-macgyver" \
  --worker-dir "$PWD" \
  --data-dir "$HOME/.local/share/cursor-agent-macgyver" \
  --computer-use \
  --connect-workspace \
  start
```

Then start a **new** Cloud Agent at https://cursor.com/agents → Run on → My Machines → `trt-vm-cursor-worker-macgyver`.

`--data-dir` avoids the glam-cp / shedlife worker lock. `--computer-use` is required for Workshop Studio (Linux needs the desktop packages in the self-hosted docs). `--connect-workspace` needs Cursor 3.25+.

Confirm: `agent worker debug` lists remote `github.com/andrewkriley/datafabric-macgyver`.

To attach macgyver to the existing glam-cp process instead, restart that worker with both `--worker-dir` roots (glam-cp first). Prefer the dedicated name above.

This Cloud Agent VM’s `.env`, Studio cookies, and browser session **do not move**. Refresh AWS CLI creds from Studio on the new machine.

---

## 3. Join shortcut (already done once)

1. Open the join URL in a **real browser** (WebFetch only shows the Amazon-employee tile).
2. Sign-in: **Email one-time password**. Not Amazon employee. Not Builder ID.
3. Email: `andreril@cisco.com` → Send passcode.
4. 9-digit OTP, ~5 minutes to arrive, ~15 minutes to live.
5. Review → accept Terms → Join event.
6. Left nav → **Get AWS CLI credentials** (bash) → local `.env`. Region `us-east-1` only.
7. Left nav → **Open AWS console (us-east-1)** (right-click → new tab). Already signed in as Participant.

Skip if the same browser is still on the dashboard. Refresh STS when `aws sts get-caller-identity` fails (`ASIA…` + session token required).

---

## 4. AWS inventory (do not recreate)

Validated 4 Nov 2026 with `aws sts get-caller-identity`.

| Item | Value |
|---|---|
| Account | `565174714811` |
| Caller | `arn:aws:sts::565174714811:assumed-role/WSParticipantRole/Participant` |
| Console role label | ParticipantRole / Participant |
| Region | `us-east-1` |
| Stack | `splunk-fs-s3-workshop` — `CREATE_COMPLETE` (2 Nov 2026 08:41 UTC) |
| Stack outputs | **null** (Studio “Event outputs” table is also empty — ignore it) |
| CloudFront logs | `splunk-fs-s3-logs-565174714811-us-east-1` |
| WAF logs | `aws-waf-logs-splunk-565174714811-us-east-1` |
| Athena results | `splunk-fs-s3-workshop-athenaoutput-emhp0rlrxldk` |
| Participant assets | `splunk-fs-s3-workshop-participantassetsbucket-quuu2ejpvj0u` |
| Glue DB | `splunk-fs-s3-565174714811` |
| Glue table | `splunk_cloudfront_access_logs_w3c` (already present) |
| FS-S3 IAM role | `splunk-fs-s3-workshop-SplunkFsS3Role-Y1Wt836v08cr` |
| Lambda role | `splunk-fs-s3-workshop-LambdaExecutionRole-bEg8Ehbo25m6` |
| CloudFront function | `splunk-fs-s3-workshop-sample-website-rewrites` |
| EventBridge rule | `splunk-fs-s3-workshop-AutomatedScannerSchedule-nHzzHWpkIjSc` |

Lesson 1 starting state is already satisfied. Do **not** create a second Glue DB, CloudFront table, or log bucket.

Quick check on the new worker (after filling `.env`):

```bash
set -a && source .env && set +a
aws sts get-caller-identity
aws s3 ls
aws glue get-tables --database-name splunk-fs-s3-565174714811 --query 'TableList[].Name'
```

---

## 5. Splunk (next gate)

Studio does **not** print a SHOW stack, SCS tenancy, or HEC token.

| Do | Do not |
|---|---|
| Log in at https://login.splunk.com/?leadSource=default_signup | Create an account at idp.login.splunk.com (24h wait) |
| After login: **Instances** and **Product Logins → Splunk Cloud Services** | Guess a `*.splunkcloud.com` host |
| Put `SPLUNK_CLOUD_URL` / `SCS_TENANCY_URL` / username in local `.env` | Commit those values |

User already has a Splunk account. Username was not stored.

---

## 6. Infisical

**These keys are not in Infisical yet.** This environment had no CLI, token, or project ID.

Placeholder (committed, no values):

- `.infisical.yaml` — env `dev`, path `/datafabric-macgyver`, `project-id: REPLACE_WITH_INFISICAL_PROJECT_ID`
- `.env.example` — the key names to create in that folder

On the new worker, after `infisical login` and a real project ID:

```bash
# edit .infisical.yaml project-id
infisical sm secrets set --env=dev --path=/datafabric-macgyver --file=./.env
infisical sm export --env=dev --path=/datafabric-macgyver > .env
```

My Machines workers do **not** receive Cursor dashboard secrets. Infisical or a local `.env` on the VM is the store.

---

## 7. Files on `riles_lab_test`

| Path | Role | Secrets? |
|---|---|---|
| `WORKER_HANDOFF.md` | This pack | No |
| `RILES_LAB_NOTES.md` | Dated running log | No |
| `RILES_REVIEW.md` | MacGyver architecture-vs-steps review + drafted doc fixes | No |
| `.env.example` | Infisical / local key catalog; non-secret inventory filled | No |
| `.infisical.yaml` | Project stub | No |
| `.gitignore` | Blocks `.env`, `*.pem`, `*.ppk`, `/lab-downloads/` | — |
| `.env` | Local only, never in git | Yes |
| `content/`, `static/` | Workshop markdown + screenshots (main) | No |

PR: https://github.com/andrewkriley/datafabric-macgyver/pull/1

---

## 8. Gotchas (reuse these)

| ID | Finding | Shortcut |
|---|---|---|
| J1 | Join is Email OTP in a browser | `andreril@cisco.com`; ignore Amazon-employee tile |
| J2 | Event is FS-S3 Immersion Day | Studio lessons 1–5; repo Lab 3 only |
| J3 | No Splunk stack URL in Studio | Instances / SCS product login |
| J4 | Studio Event outputs empty | Use console / CLI; stack still `CREATE_COMPLETE` |
| J5 | Region lock | `us-east-1` only |
| J6 | Lab 3c folder `lab3c_spunk_managed_catalog` | Typo; links still work |
| J7 | `.env` values with spaces break `source` | Quote `WORKSHOP_EVENT_NAME` and `WORKSHOP_CONTENT_TITLE` |
| J8 | Open AWS console | Right-click → new tab |
| J9 | Lesson 1 Glue table already exists | Do not recreate |
| J10 | Infisical empty | Placeholder only until project ID |
| J11 | This Cloud Agent is Cursor-hosted | New agent on My Machines; chat does not migrate |
| J12 | glam-cp worker is the wrong repo | Dedicated `--name` + `--data-dir` + macgyver checkout |
| J13 | Repo Lab 3a S3-connection walkthrough is HTML-commented | This event already has stack + Glue; follow Studio Lesson 1 |
| J14 | MacGyver `architecture.md` oversells Lab 5 Analytics promote and Lab 6 FS-over-DDSS | Those steps are not in this event |

Drafted MacGyver markdown fixes (typos, Lab 5/6 claims) stay in `RILES_REVIEW.md`. Do not rewrite lab markdown on this branch unless asked.

---

## 9. First tasks on the new worker

1. Checkout `riles_lab_test`, `cp .env.example .env`.
2. Re-join Studio in a local browser if the session is gone; refresh bash STS into `.env`.
3. Confirm account `565174714811` and Glue table `splunk_cloudfront_access_logs_w3c`.
4. Log in to the existing Splunk account; record stack / SCS URLs.
5. Optional: fill Infisical project ID and push `.env` keys to `/datafabric-macgyver`.
6. Start Studio Lesson 1 (CloudFront). Do not start MacGyver Lab 1.

Append dated notes to `RILES_LAB_NOTES.md` as you discover more. Keep secrets out of git.
