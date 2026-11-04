# Riles lab notes — running log

Shortcuts and fixes discovered while joining and running the live event.
**No secrets in this file.** Credentials live in local `.env` only.

Last updated: 4 Nov 2026, 2:00 PM Australia/Sydney.

---

## Event we actually joined

This is **not** the full six-lab Data Fabric MacGyver SHOW.

| Field | Value |
|---|---|
| Event | Splunk - Federated Search Workshop with S3 and Glue |
| Content title | Becoming a Splunk Federated Search for Amazon S3 Master |
| Type | Individual Workshop Studio event (test content) |
| Level | 200 in dashboard / 300 in welcome page |
| Region | `us-east-1` only |
| Start | 2 Nov 2026, 8:44 AM (studio clock) |
| Duration | 72 hours |
| Join email | `andreril@cisco.com` |
| Dashboard | https://catalog.us-east-1.prod.workshops.aws/event/dashboard/en-US |
| Join URL | https://catalog.us-east-1.prod.workshops.aws/join?access-code=a97c-068433-82 |

Welcome page says participants get access to **both Splunk and AWS labs**. Expected duration ~2 hours. Event outputs were empty at join (no CloudFormation outputs yet).

### Lessons vs this repo

| Studio lesson | Closest repo path | Notes |
|---|---|---|
| Getting Started / Architecture | `content/architecture.md` + Lab 3 intro | Studio architecture is FS-S3 only |
| Splunk Account Setup | none | Login to an existing Splunk identity; skip account create |
| Lesson 1 CloudFront | `content/lab3_.../lab3a_basics_of_fs_s3/` | Same claim: first FS-S3 dataset |
| Lesson 2 VPC Flow + Athena DDL | `content/lab3_.../lab3b_using_athena_ddl_for_aws_glue/` | Same claim |
| Lesson 3 WAF + Splunk-managed catalog | `content/lab3_.../lab3c_spunk_managed_catalog/` | Same claim; folder typo `spunk_` |
| Lesson 4 FS-S3 use cases | **not in repo** | Extra vs MacGyver |
| Lesson 5 time-based partitions | **not in repo** | Extra vs MacGyver |
| MacGyver Labs 1, 2, 4, 5, 6 | repo only | **Out of scope for this event** unless we find them later |

Do not start MacGyver Lab 1 (PII pipeline) or Lab 6 (DDSS) on this stack. Those claims are not this workshop.

---

## Join shortcut (already done)

1. Open the join URL (access code is in the query string).
2. Sign-in method: **Email one-time password (OTP)**. Not Amazon employee. Not Builder ID.
   - `curl` / WebFetch of the join URL only shows the Amazon-employee tile. The OTP tile is browser-only.
3. Email: `andreril@cisco.com`. Send passcode.
4. 9-digit code arrives within ~5 minutes, lives ~15 minutes. Paste and Sign In.
5. Review event → accept Terms → Join event.
6. Left nav → **Get AWS CLI credentials** (bash). Copy into local `.env`. Use `us-east-1` only.
7. Left nav → **Open AWS console (us-east-1)** for browser work.

OTP used 4 Nov 2026 (Australia/Sydney afternoon) succeeded on the first try.

### Rerun skip list

- Do not recreate a Splunk account. User already has one. Skip the register form and the “wait up to 24 hours” path.
- Do not re-join if the same browser session is still on the dashboard.
- Refresh CLI creds from the dashboard if STS calls start failing (ASIA keys + session token).

---

## AWS access (this VM)

- CLI credentials stored in gitignored `.env` (4 Nov 2026). Validated with `aws sts get-caller-identity`.
- Assumed role: `WSParticipantRole/Participant` (console also shows ParticipantRole/Participant).
- Prefix `ASIA…` = temporary STS. Session token is required. Refresh from dashboard if STS fails.
- Event Outputs table on the Studio dashboard is empty even though the stack exists. Use console / CLI, not that table.

### Pre-created account inventory (do not recreate)

| Item | Value |
|---|---|
| Account | `565174714811` |
| Region | `us-east-1` |
| Stack | `splunk-fs-s3-workshop` (`CREATE_COMPLETE`, 2 Nov 2026 08:41 UTC) |
| CloudFront logs bucket | `splunk-fs-s3-logs-565174714811-us-east-1` |
| WAF logs bucket | `aws-waf-logs-splunk-565174714811-us-east-1` |
| Athena results | `splunk-fs-s3-workshop-athenaoutput-emhp0rlrxldk` |
| Participant assets | `splunk-fs-s3-workshop-participantassetsbucket-quuu2ejpvj0u` |
| Glue DB | `splunk-fs-s3-565174714811` |
| Glue table | `splunk_cloudfront_access_logs_w3c` (CloudFront w3c, already present) |
| FS-S3 IAM role | `splunk-fs-s3-workshop-SplunkFsS3Role-Y1Wt836v08cr` |
| Lambda role | `splunk-fs-s3-workshop-LambdaExecutionRole-bEg8Ehbo25m6` |

Lesson 1 starting state is already satisfied: stack, Glue catalog, and CloudFront table exist. Click Next; do not create a second Glue DB or CF logs bucket.

CloudFront function seen in stack search: `splunk-fs-s3-workshop-sample-website-rewrites`.
EventBridge rule: `splunk-fs-s3-workshop-AutomatedScannerSchedule-nHzzHWpkIjSc`.

---

## Splunk (next gate)

Studio **does not** give a SHOW stack URL, SCS tenancy, or HEC token.

Getting Started → Splunk Account Setup says:

1. Create account at https://idp.login.splunk.com/signin/register — **skip; account exists**.
2. New-account verification can take 24 hours / +1-855-775-8657 option 2 — **skip**.
3. Test login: https://login.splunk.com/?leadSource=default_signup
4. After login, the profile menu should show My Dashboard, **Instances**, My Training, and Product Logins (Observability Cloud, **Splunk Cloud Services**, AppDynamics, On-Call, Synthetic Monitoring, Web Optimization).

Next time: log in with the existing Splunk account, then check **Instances** and **Product Logins → Splunk Cloud Services** for the stack/tenancy this event expects. Do not guess a `*.splunkcloud.com` host.

---

## Fixes / gotchas found so far

| # | Discovery | Shortcut / fix |
|---|---|---|
| J1 | Join page is OTP + access code, not a SHOW email pack | Drive it in a real browser; email OTP to `andreril@cisco.com` |
| J2 | Event is FS-S3 Immersion Day, not MacGyver Labs 1–6 | Follow studio lessons 1–5; use repo Lab 3 as the parallel guide |
| J3 | No stack URL in studio pages | After Splunk login, look at Instances / SCS product login |
| J4 | Event outputs empty at join | Do not invent bucket names; wait for lab-created stacks or console inventory |
| J5 | Region lock | Only `us-east-1` |
| J6 | Repo Lab 3c folder is `lab3c_spunk_managed_catalog` | Known typo; links still work |
| J7 | Studio Event Outputs table stays empty | Inventory with console / `aws` CLI; stack still `CREATE_COMPLETE` |
| J8 | `.env` values with spaces break `source` | Quote `WORKSHOP_EVENT_NAME` and `WORKSHOP_CONTENT_TITLE` |
| J9 | Open AWS console from Studio | Right-click → new tab; console is already signed in as Participant |
| J10 | Lesson 1 Glue table already exists | Skip any “create the CloudFront Glue table” if the studio copy implies it is missing |
| J11 | Workshop keys are **not** in Infisical yet | Placeholder only: `.infisical.yaml` + `.env.example`. Need project ID / `infisical login` to create the folder |

---

## Infisical (4 Nov 2026)

Checked this repo, this VM, and process env. **None of the workshop variables are in Infisical from here.**

| Check | Result |
|---|---|
| `.infisical.yaml` / `.infisical.json` in repo | Missing (placeholder added) |
| `infisical` CLI | Not installed |
| `INFISICAL_*` process env | Empty |
| Token / project ID | Not available — cannot list or create live secrets |

Placeholder now in git (no values):

- `.infisical.yaml` → env `dev`, path `/datafabric-macgyver`, `project-id: REPLACE_WITH_INFISICAL_PROJECT_ID`
- `.env.example` → same key names we would create in that folder

To actually provision: send the Infisical org/project ID (or run `infisical login` + `infisical init` on this VM), then we can `secrets set --file=./.env` from the gitignored local file.

---

## Still open

1. Infisical project ID / login so placeholders can become real secrets.
2. Splunk login with the existing account (username not in `.env` yet).
3. Discover SPLUNK_CLOUD_URL / SCS_TENANCY_URL from Instances or product logins.

When those land, add a dated section below rather than rewriting the join shortcut.
