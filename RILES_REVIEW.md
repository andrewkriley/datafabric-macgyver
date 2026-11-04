# Riles lab review — Data Fabric MacGyver

Review of `andrewkriley/datafabric-macgyver` before a live SHOW reproduction.
Stance: reproduce the lab **as written**, and **prove the claims the steps actually implement**. Where `architecture.md` promises work the markdown never does, capture it here rather than invent extra labs.

This branch also adds:

- `.env.example` — placeholders only
- `.gitignore` — `.env`, `.env.*` (except the example), `*.pem`, `*.ppk`, `/lab-downloads/`

Copy `.env.example` to `.env` locally. Do not paste secrets into git, chat, or GitHub.

---

## 1. Access check (question 4)

**Outbound access is not the blocker. Credentials are.**

This Cursor environment has unrestricted egress. From the review VM (4 Nov 2026, Australia/Sydney):

| Target | Result | Meaning |
|---|---|---|
| `sts.us-east-1.amazonaws.com` | HTTP 403 `MissingAuthenticationToken` | Network path works. No AWS identity. |
| `s3.amazonaws.com` | HTTP 307 | Reachable |
| `console.aws.amazon.com` | HTTP 302 | Reachable |
| `console.scs.splunk.com` | HTTP 200 | Reachable |
| `help.splunk.com` | HTTP 302 | Reachable |
| Process env (`AWS_*`, `SPLUNK_*`, `SHOW_*`, `HEC_*`) | Empty | No secrets loaded |
| AWS CLI | Not configured | Cannot assume a role yet |

We cannot log into the SHOW Splunk Cloud stack, SCS tenancy, or workshop AWS account until `.env` is filled from the forthcoming SHOW instructions. Once those values exist locally, the same VM can reach the APIs. The customer stack URL is unknown until those instructions arrive — `https://splunkcloud.com/` is not a real stack host.

---

## 2. Decisions locked from the grill

| # | Decision |
|---|---|
| 1 | Reproduce in a **real SHOW lab**, not a private stand-up. |
| 2 | Prove the claims the **lab steps** implement. |
| 3 | Draft fixes **in this file**. Do not rewrite workshop markdown on this branch. |
| 5 | Assume the SHOW stack and data generators are present. |
| 6–7 | Follow the guide. Region default `us-east-1`. Stack life **max 3 days**. |
| 8–9 | SHOW login instructions still incoming. No leftover connections assumed. |
| 10–11 | Follow the lab guide for Lab 5 wait and Lab 6 same-day archive limit. |
| 12 | **Note only:** Lab 5 never promotes to an Analytics table. See §5. |
| 13–19 | Follow the lab guide (parquetreader.com, synthetic PII display, bucket names, HEC/MDL redirects, EC2 leftover, DDSS prefix). |
| 15 | Screenshots are acceptable evidence. |
| 16 | Do not terminate the Lab 3b EC2; SHOW teardown will close it. |
| 20–22 | All credentials live in local `.env`. Include the access_combined HEC token. |

---

## 3. How we will prove each lab (follow the guide)

Do labs in order. Lab 2 creates `fs-s3-workshop` (Lab 3a’s connection walkthrough is commented out). Lab 1 creates `cc_trans_pii` that Lab 6 archives.

### Pre-flight (once SHOW is up)

Confirm live ingest before changing anything. Last 15 minutes must return events:

```spl
index=cc_trans_im
index=cisco-asa-logs
index=linux_secure
index=access_combined
```

Also confirm three consoles open: Splunk Cloud Search & Reporting, SCS tenancy, AWS.

### Lab 1 — PII split (streaming)

**Claim implemented:** Ingest Processor branches `cc_trans_im` into masked IM data and raw `cc_trans_pii`.

**Pass:**

1. Pipeline `split_mask_filter_pii_cc_trans` is applied (green).
2. Preview shows TRACK1/TRACK2/PIN_BLOCK/CC_CCV/PIN gone and `CC_NUMBER` masked.
3. `index="cc_trans_im"` last 15m matches the masked screenshot.
4. `index="cc_trans_pii"` last 15m is still raw.

### Lab 2 — Cisco ASA clone + filter (streaming + storage)

**Claim implemented:** Raw `cisco:asa` to S3 parquet; Built/Teardown/Disallow dropped from Splunk.

**Pass:**

1. Connection `fs-s3-workshop` exists with federated search + ingest abilities.
2. IAM role trust policy replaced; tag `splunk-assumable-role=true`.
3. Bucket `splunk-cisco-asa-s3-log-storage-<name>` (must start with `splunk-`).
4. Destination `cisco-asa-s3-logs` visible in SCS.
5. Pipeline `cisco_asa_pipeline` applied.
6. `index="cisco-asa-logs"` still receiving data after v1.
7. Parquet under `sourcetype=cisco%3Aasa/` hive folders; open one in parquetreader.com as the guide says.
8. After v2, last 2 minutes of Built/Teardown/Disallow is empty (or collapsing).
9. Ingest Processor metrics: S3 destination volume > Splunk indexer volume.

### Lab 3a — CloudFront FS-S3 (analysis)

**Claim implemented:** Glue-backed federated dataset on w3c CloudFront logs.

**Pass:**

1. CloudFront “Sample Web Application” logs to `splunk-fs-s3-logs-{account}-{region}`.
2. Glue table `splunk_cloudfront_access_logs_w3c` schema matches the downloaded log `#Fields`.
3. Dataset `cloudfront_access_logs` created (reuse Lab 2 connection; do not recreate it).
4. SPL2, All time:

```spl2
index=cloudfront_access_logs | head 100
```

First search may sit in “Resources are being provisioned” for 3–5 minutes. Retry. Empty results with no error usually means the time format (`time` / `%H:%M:%S` / `_time`) is wrong.

### Lab 3b — VPC Flow + Athena DDL (analysis)

**Claim implemented:** Hive parquet flow logs, Glue table via Athena DDL, federated search.

**Pass:**

1. Flow log `full-vpc-flow-log` on the workshop VPC, parquet + hive prefix, 1 minute aggregation.
2. EC2 launched in that VPC; flow objects appear in S3 (guide allows ~10 minutes).
3. Athena results bucket set to `splunk-fs-s3-workshop-athenaoutput-xxx`.
4. `CREATE EXTERNAL TABLE ... splunk_vpc_flow_logs` against the vpcflowlogs S3 URI.
5. Database still `splunk-fs-s3-{account}` when running `MSCK REPAIR TABLE splunk_vpc_flow_logs` (Athena resets to `default`).
6. Glue schema matches the parquet columns.
7. SPL2 `index=vpc_flow_logs | head 100` returns events.

### Lab 3c — WAF Splunk-managed catalog (analysis)

**Claim implemented:** JSON WAF logs + SQS notification + Splunk-managed crawler.

**Pass:**

1. Web ACL `splunk-fs-s3-workshop-webacl` logs to `aws-waf-logs-splunk-{account}-{region}`.
2. SQS `splunk-fs-s3-{account}` access policy allows `s3.amazonaws.com` → that bucket.
3. S3 event notification `aws-waf-s3-notification` on the WAF prefix.
4. Dataset `aws_waf_logs`, no catalog, JSON/gzip, discover schema via crawler, partitions year/month/day/hour/min.
5. SPL2 `index=aws_waf_logs | head 100`. `Dataset is still being created: PROCESSING` for ~5 minutes is expected.

### Lab 4 — MDL route + static promote (storage)

**Claim implemented:** `linux_secure` sourcetype lands in MDL raw; last 5 minutes statically promoted to DDAS.

**Pass:**

1. Dataset `linux_secure_mdl`, `sc_admin` with streaming/static promote + query + view, partition `sourcetype=linux_secure`.
2. Catalog search on `linux_secure_mdl` returns events (guide: up to ~10 minutes).
3. `index=linux_secure earliest=-2m` returns no new events.
4. Promote job `linux_secure_mdl_promoted_5mins` goes Pending → In progress → Complete.
5. Search on the promoted dataset returns quickly (DDAS).

### Lab 5 — HEC → MDL (storage)

**Claim implemented by the steps:** `access_combined` HEC default index flipped to `~.federated_access_combined_mdl`.

**Pass (follow the guide):**

1. Dataset `access_combined_mdl` with **empty** landing rules.
2. Catalog search is empty before the HEC change.
3. HEC token: add `~.federated_access_combined_mdl`, remove `access_combined`.
4. `index=access_combined earliest=-5m` quiet.
5. Catalog `access_combined_mdl` has events. Guide says this **may take around 100 minutes**. Wait that long before calling it a fail.

**Architecture claim not implemented — note for #12:**
`architecture.md` Lab 5 is titled “Linux Secure logs routing” and the Lab 5 index says we will “promote some data into Analytics table”. The steps never open the promote wizard and never create an analytics table. `linux_secure` routing is Lab 4. Treat Analytics-table promote as **out of scope for this run**. Draft fix in §5.

### Lab 6 — DDSS self-storage (storage)

**Claim implemented by the steps:** BYO S3 self-storage attached to `cc_trans_pii` at 100MB / 1 searchable day.

**Pass (same-day, follow the guide):**

1. Self Storage Location `cc-trans-pii-s3`.
2. Bucket name uses the **prefix the UI prints** (example shape `fs2-s3-shw-…`). Do not invent it.
3. Generated bucket policy pasted; **Test** succeeds; test object visible in S3.
4. Index `cc_trans_pii`: max raw 100**MB**, DDSS selected, searchable days = 1, location = `cc-trans-pii-s3`.

**Not same-day:** objects aging off to DDSS. Guide says wait ~24 hours and that labs usually survive 1–2 days (SHOW window here is max 3 days). Set a reminder; do not fail the lab on day 1 for missing archive objects.

**Architecture claim not implemented:** `architecture.md` says we will “configure Federated Search to search that DDSS data”. Lab 6 never creates a federated dataset on the DDSS bucket. Draft fix in §5.

---

## 4. Draft fixes — workshop markdown

Apply these later in a docs PR. Not applied on this branch.

### 4.1 Must-fix (will mislead a first-time student)

**A. Lab 1 index typo**

`content/lab1_cc_trans/02_section_two_creating_first_blank_pipeline.md` says:

> Send the raw logs to a new special PII index called `cc_trains_pii`

Draft:

> Send the raw logs to a new special PII index called `cc_trans_pii`

**B. Lab 3a connection is commented out, but the lesson still reads as if you create one**

`content/lab3_using_splunk_fs_s3/lab3a_basics_of_fs_s3/3-creating-out-first-fs-s3-index.en.md` wraps the entire “Creating a Splunk Amazon S3 Connection” walkthrough in HTML comments. The live steps jump to “Creating CloudFront Dataset” and say “Select the connection we created”.

Draft — add a visible note at the top of that section:

> This lesson reuses the Amazon S3 connection `fs-s3-workshop` created in Lab 2. Do not create a second connection. If you skipped Lab 2, go back and finish the connection, IAM trust policy, and `splunk-assumable-role=true` tag first.

**C. Lab 5 / Lab 6 architecture oversell**

`content/architecture.md` Lab 5 title and body do not match `content/lab5_machine_data_lake_ingestion/`. Lab 6 promises federated search the lab never runs.

Draft Lab 5 architecture block:

> ## Lab 5: Ingest access_combined straight into Machine Data Lake
>
> The team wants a new home for `access_combined` web logs. They are not searched often.
> This lab creates a second MDL raw table and points the existing HEC token at it,
> so new events land in MDL without an Ingest Processor sourcetype partition.

Draft Lab 6 architecture last paragraph — delete or replace the FS sentence:

> In this lab we configure a DDSS archive on your own S3 bucket and attach it to
> `cc_trans_pii` at the smallest retention Splunk allows (100MB or 1 day).
> Federated search over DDSS is out of scope for this workshop; the Self Storage
> Test and the index Additional storage setting are the validation.

**D. Lab 5 promised Analytics promote that is not in the steps**

`content/lab5_machine_data_lake_ingestion/index.en.md` implementation list:

> Lastly we will promote some data into Analytics table to see how that works.

Draft — drop that bullet, or add a real section. Preferred for this workshop (smaller change):

> - Check `access_combined` is coming into Splunk DDAS
> - Create a second MDL raw table with no landing partition
> - Point the `access_combined` HEC token at the MDL federated index
> - Confirm new events leave DDAS and appear in the MDL catalog

If the Analytics promote is still a teaching goal, copy Lab 4’s promote wizard and select **Static analytics table**. That is new lab content, not a one-line fix.

**E. Lab 5 wait time contradicts Lab 4**

Lab 4: “Search results may take up to 10 minutes.”
Lab 5: “The data may take around 100 minutes to appear in the data catalog.”

Draft: keep 100 minutes only if SHOW confirms HEC→MDL is that slow. Otherwise:

> Catalog results may take up to 10 minutes after the HEC token is saved, same as Lab 4.

Until SHOW confirms, this reproduction **follows the written 100 minutes**.

### 4.2 Should-fix (typos / numbering / names)

| File | Issue | Draft |
|---|---|---|
| `README.md` | “Snoflake”, “if t hat doesn’t” | Snowflake; if that doesn’t |
| `content/lab1_cc_trans/01_section_one_viewing_existing_data.md` | “out goal” | our goal |
| `content/lab1_cc_trans/02_section_two_creating_first_blank_pipeline.md` | “exmaple”; steps numbered 1, 1, 2, 3, 5… | example; renumber 1–16 |
| `content/lab1_cc_trans/04_section_four_validate_our_pipeline_is_working.md` | two step 2s | 1, 2, 3 |
| `content/lab2_cisco_firewall_logs/01_configuring_amazon_s3_destination.md` | “AWS accound id” | AWS account id |
| `content/lab2_cisco_firewall_logs/03_route_cisco_asa_s3_and_filter_logs.md` | “exa,ple”, “lets mow create”, duplicate step 10 | example; let’s now create; renumber |
| `content/lab3_using_splunk_fs_s3/lab3a_basics_of_fs_s3/3-creating-out-first-fs-s3-index.en.md` | “accound-id” | account-id |
| `content/lab3_using_splunk_fs_s3/lab3c_spunk_managed_catalog/` | folder `spunk_` | rename to `splunk_managed_catalog` and update every `next` link |
| `content/lab3_using_splunk_fs_s3/lab3c_spunk_managed_catalog/index.en.md` | “how how to configure” | how to configure |
| `content/lab3_using_splunk_fs_s3/lab3c_spunk_managed_catalog/2-configuring-aws-waf-federated-dataset.en.md` | “WAS WAF”; “VPC Flow Log Federated Dataset” in the WAF test heading | AWS WAF; AWS WAF Log Federated Dataset |
| `content/lab4_machine_data_lake_101/02_promote_data_mdl.md` | `linux_secure_mdl_promotedd_5mins` | `linux_secure_mdl_promoted_5mins` |
| `content/lab5_machine_data_lake_ingestion/01_create_second_mdl_dataset.md` | “we we will use”; “HTTP Event Collect”; “now results” | we will use; HTTP Event Collector; no results |
| `content/lab6_configuring_fs_ddss/02_configure_ddss_index.md` | “Splunk your Splunk console” | From your Splunk console |
| Athena / Glue / WAF lessons | duplicate step numbers (12/12, 13/13, 17/17) | sequential numbers |

### 4.3 Nice-to-have

- Lab 3b: say explicitly that the EC2 must land in the **workshop VPC** (defaults usually do; a student who changes the network section will get empty flow logs).
- Lab 3c: “Switch to the new WAF console” is easy to miss. Put it in the first numbered step, not only a warning.
- Lab 2 / 3b: parquetreader.com is third-party. Fine for this synthetic SHOW data; add one line that production logs should be opened locally.
- Pre-flight page: a single “before you start” checklist (three URLs, four indexes hot, AWS account id copy button, region).

---

## 5. Claim tracker — architecture vs steps vs this run

| Architecture claim | Implemented in steps? | This run |
|---|---|---|
| Lab 1 clone + mask PII into two indexes | Yes | Prove with the two searches |
| Lab 2 raw ASA to S3 + filter into Splunk | Yes | Prove with S3 parquet + 2-minute filter search + metrics |
| Lab 3 CloudFront / VPC / WAF via FS-S3, three catalog methods | Yes | Prove with three SPL2 searches |
| Lab 4 DDAS → MDL raw → promote subset back to DDAS | Yes | Prove with quiet DDAS + catalog + promoted index |
| Lab 5 “Linux Secure logs routing” | No — that is Lab 4 | Note only |
| Lab 5 ingest `access_combined` to MDL via HEC | Yes | Prove with HEC flip + catalog (wait up to 100 min) |
| Lab 5 promote into Analytics table | **No** | Note only. Do not invent the wizard this run. |
| Lab 6 DDSS on your S3 bucket attached to an index | Yes | Prove with Test object + index UI |
| Lab 6 federated search of DDSS | **No** | Note only. Same-day validation stops at Test + index settings. |

---

## 6. Secrets and local files

| Item | Where it lives |
|---|---|
| Splunk / SCS / AWS / HEC | `.env` (gitignored) |
| Template | `.env.example` (committed, empty values) |
| Lab 3b EC2 key | `EC2_KEY_PATH` on disk; `*.pem` / `*.ppk` gitignored |
| Downloaded CF / WAF / parquet samples | `/lab-downloads/` (gitignored) |

When SHOW instructions arrive, fill `.env` and confirm:

- `AWS_ACCOUNT_ID` is digits only (console Copy button).
- `AWS_REGION` matches the guide (`us-east-1` unless SHOW overrides).
- `S3_DDSS_BUCKET` is not guessed — copy the Self Storage prefix from the UI in Lab 6.
- `S3_ASA_BUCKET` starts with `splunk-`.

---

## 7. Still waiting

1. SHOW instruction pack: three URLs, usernames, AWS STS creds, HEC token, any stack-specific bucket/role suffixes.
2. Confirmation that generators are already writing to the four indexes in §3.
3. Optional: SHOW owner confirmation whether Lab 5’s “100 minutes” is real.

No live lab clicks until `.env` is populated. This review is the runbook for that session.
