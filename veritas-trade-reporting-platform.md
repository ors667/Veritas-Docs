---
title: Veritas — Trade Reporting Platform
owner: Platform Engineering
compliance_owner: Financial Reporting & Regulatory Affairs
tier: "1"
status: production
source_of_truth: confluence:FO/9043970
---
# Veritas — Trade Reporting Platform

| Field | Value |
| --- | --- |
| Owner | Platform Engineering |
| Compliance owner | Financial Reporting & Regulatory Affairs |
| Last updated | May 2026 |
| Status | ✅ Production |
| Tier | 1 — Trading and Market Data (cross-tier upgrade) |
| Cloud | AWS (infrastructure) + Kubernetes (workloads) |

---

## Platform Classification

**Tier 1 — Trading and Market Data**

Registered tier: Tier 2 (Regulatory Reporting). Effective tier: Tier 1, because Veritas
reads directly from trade execution records sourced from Tier 1 systems. Under the
Application Tier Classification Policy, a cross-tier application is always classified at
the highest applicable tier. All Tier 1 controls apply.

**What that means in practice:**

- **Access review cadence:** Quarterly, by the owning team lead and Compliance — not annual
- **IAM changes:** Every change to a Veritas IAM role or binding generates a notification to Compliance Engineering, not just Platform Engineering. See the Controls and Changing Things sections below.
- **Identity isolation:** Veritas IAM roles (veritas-ingestor, veritas-reporter) may not be shared with or assumed by any other application, regardless of tier
- **Audit logs:** Must be immutable and retained for the periods required by MiFID II (5 years), EMIR (10 years), and Dodd-Frank (5 years). Veritas uses a 10-year floor to satisfy all three.
- **Re-classification:** If the scope of Veritas expands — for example to handle pre-trade data or position records — notify Platform Engineering before that data access begins. Re-classification requires CISO review.
---

## What Does Veritas Do?

Veritas is the system that makes sure every trade we execute gets reported to the right
regulator, in the right format, on time. That's it. It sounds straightforward — it
isn't.

When a trade happens, a clock starts ticking. Under MiFID II, most trades have to be
reported to a trade repository by the next business day (T+1). Under EMIR, OTC
derivatives have to be reported the same day. Missing a deadline, submitting a malformed
report, or reporting the wrong data can result in regulatory fines, remediation demands,
or worse — a public finding.

Veritas sits between the trading systems and the regulators. It picks up every trade
event, validates it, makes sure it hasn't been reported before, builds the correct
regulatory report format, submits it, and keeps a tamper-proof archive of everything it
does. If a submission is rejected, it tries again and alerts the team. Nothing gets
dropped silently.

---

## Regulations Veritas Serves

| Regulation | What it covers | Reporting deadline |
| --- | --- | --- |
| MiFID II (RTS 22) | Equities, bonds, derivatives traded on EU venues | T+1 (next business day) |
| EMIR (REFIT) | OTC and exchange-traded derivatives | T+1 (same day for some) |
| Dodd-Frank (SDR) | US swap transactions | T+1 |

Record retention requirements range from 5 years (MiFID II) to 10 years (EMIR). Veritas
keeps everything in encrypted cold storage for 10 years so we're covered under all
three.

Cross-tier data access: Veritas receives trade events originating from Tier 1 systems
(Order Management, Execution Management). This is why it carries Tier 1 controls despite
being a reporting system. The data doesn't become lower-risk just because it's passed to
a downstream service.

---

## Controls

### Identities

Veritas provisions two IAM roles. Neither may be assumed by any other application, and
no other application's role may be bound to any Veritas resource. This isolation is a
hard Tier 1 requirement.

**veritas-ingestor** — the execution role for the ingestor pods. Permitted to: consume messages from the trade ingestion SQS queue (sqs:ReceiveMessage, sqs:DeleteMessage, sqs:GetQueueAttributes), read and write to the Aurora trade ledger (writer endpoint only), read and write to the ElastiCache Redis deduplication set, and use the Veritas KMS key for decrypt and generate-data-key operations. It may not read from the regulatory submission archive S3 bucket, publish to SNS, or assume any other role.

**veritas-reporter** — the execution role for the reporter pods and EOD CronJob. Permitted to: consume messages from the submission SQS queue, read from the Aurora trade ledger (reader endpoint only), write objects to the regulatory submission archive S3 bucket (s3:PutObject only), and use the Veritas KMS key for decrypt and generate-data-key operations. It may not delete or overwrite objects in S3, write to Aurora, or touch the trade ingestion queue.

Any change to either role — including adding a permission to an existing statement —
requires a Compliance Engineering reviewer on the PR before merge. This is not optional
and is not delegable to Platform Engineering alone.

### Immutability

The regulatory submission archive S3 bucket must use Object Lock in COMPLIANCE mode with
a 10-year default retention period. This is the mechanism required by SEC Rule 17a-4 and
EMIR for non-rewritable, non-erasable storage. Object Lock in COMPLIANCE mode prevents
deletion or overwrite by any principal — including the account root — for the duration
of the retention period.

COMPLIANCE mode is stricter than GOVERNANCE mode and is the correct setting for
regulatory records. Object Lock must be enabled at bucket creation time; it cannot be
added to an existing bucket retroactively.

Retention period: 10 years (3,650 days). This satisfies MiFID II (5 years), Dodd-Frank
(5 years), and EMIR (10 years) simultaneously.

### Encryption

**At rest:** All Veritas data at rest is encrypted using a single customer-managed KMS key (veritas-key). This covers: Aurora cluster storage, ElastiCache Redis, both SQS queues (trade ingestion and submission), the regulatory submission archive S3 bucket, and CloudWatch log groups. AWS-managed keys are not acceptable for Tier 1 data. Key rotation must be enabled.

**In transit:** All data in transit uses TLS 1.2 or higher. The S3 bucket must enforce this via a bucket policy that denies any request where aws:SecureTransport is false — SSL-only access is a stated obligation, not a default. Aurora connections from application pods must use SSL. The SQS queues enforce HTTPS by default via the AWS SDK.

**Redis authentication:** Redis AUTH must be enabled. The auth token must be stored in a Kubernetes secret and injected as an environment variable — it must not be hardcoded in the Helm values file or in any image. In-transit encryption must also be enabled on the ElastiCache cluster (transit_encryption_enabled = true). Both controls are required; either one alone is insufficient.

### Audit Trail

AWS CloudTrail must be enabled and configured to write to a dedicated audit S3 bucket
separate from the regulatory submission archive. That bucket must also use Object Lock
in COMPLIANCE mode and retain records for 7 years. CloudTrail logs must be encrypted
with the Veritas KMS key.

VPC Flow Logs must be enabled for the Veritas VPC and written to CloudWatch Logs with a
7-year retention period, encrypted with the Veritas KMS key.

Both are regulatory obligations under the frameworks listed above, not optional
operational tooling. The audit trail is what Meridian Capital hands to regulators and
auditors on request. If either is absent or misconfigured, it is a Tier 1 compliance gap
requiring immediate remediation.

### Alerting

Two CloudWatch alarms are required:

**Trade ingestion DLQ depth** — fires immediately when any message lands in the dead-letter queue. Routes to the on-call SNS topic. A message in the DLQ means a trade was not processed — potential MiFID II reporting gap. This alert must never be silenced during trading hours.

**Submission DLQ depth** — fires immediately when any message lands in the reporter dead-letter queue. Routes to the on-call SNS topic and the Regulatory Affairs distribution list. A submission failure after 5 retries requires human intervention before the T+1 deadline.

Both alarms are defined in Terraform. They must not be created or modified manually in
the AWS Console.

---

## How It Works

A trade goes through three stages:

**Ingestion**

A trade event lands in a queue (SQS). The ingestor picks it up, validates the data
(counterparty LEIs, instrument codes, price fields), checks whether a UTI (Unique Trade
Identifier) for this trade has already been generated (checked against Redis to prevent
duplicates), then writes the validated trade to the trade ledger database (Aurora
PostgreSQL). It then pushes the trade to a second queue for the reporter.

**Reporting**

The reporter picks up from the submission queue. It builds the regulatory report in the
required format (MiFID II RTS 22 XML, or EMIR REFIT XML depending on the trade type),
submits it to the appropriate trade repository, and waits for a receipt. If the
repository rejects the report, it retries up to 5 times. If it still fails, it goes into
the dead-letter queue and the ops team is alerted.

**Archival**

Every report generated, every submission receipt, every rejection notice is written to
S3 with a 10-year retention policy. This is the paper trail that regulators can ask to
see.

At 18:00 UTC on every trading day, a scheduled batch runs to produce the end-of-day
consolidated report for any trades that didn't flow through in real time.

---

## What Lives Where

This is the most important thing to understand about how Veritas is built: the data
infrastructure lives in AWS, managed by Terraform. The application logic runs in
Kubernetes, managed by Helm. They're deliberately separate.

### AWS (managed by Terraform)

These are the durable, stateful parts of the system. You don't log into these —
Terraform manages them, changes go through a PR.

**Trade Ledger (Aurora PostgreSQL)** The database of record for every trade Veritas has seen. Aurora Serverless v2 scales automatically with trade volume. There's a writer instance and a read replica; the reporter uses the replica so reporting queries don't compete with ingestion writes. Encrypted with CMK. 35-day point-in-time recovery. Automated backups. Multi-AZ with automatic failover in ~30 seconds.

**Reference Data Cache (ElastiCache Redis)** Instrument data, counterparty LEI lookups, and the UTI deduplication set live here. The UTI check is why Redis matters: if the ingestor crashes mid-batch and restarts, it needs to know which trade identifiers it already generated so it doesn't create duplicates. Redis holds that set with a 72-hour TTL (covers T+1 plus weekends). Redis AUTH enabled. Encrypted in transit and at rest.

**Trade Ingestion Queue (SQS)** The front door for incoming trade events. Upstream trading systems push events here; the ingestors consume them. Dead-letter queue captures any trade that fails processing 3 times — those require manual review. The ops alarm fires immediately when anything lands in the DLQ, because under MiFID II you cannot silently discard a trade.

**Regulatory Submission Archive (S3)** Every report generated and submitted, plus all receipts and rejections, lands here. Encrypted with CMK. Object Lock in COMPLIANCE mode — 10-year retention, no deletions or overwrites by any principal. Objects move to Glacier after 365 days but are retained for 10 years. SSL-only access policy enforced via bucket policy.

**Audit Trail (CloudTrail + VPC Flow Logs)** Every API call made in our AWS account is logged via CloudTrail to a dedicated audit bucket. Every network connection in and out of our VPC is logged via VPC Flow Logs to CloudWatch. Both are encrypted with the Veritas CMK and retained for 7 years. This is what we hand to auditors.

### Kubernetes (managed by Helm)

These are the stateless application workloads. All persistent state lives in AWS.

**Ingestor (3–12 pods, scales with queue depth)** Consumes trade events, validates, deduplicates, writes to Aurora. Scales up during high-volume periods and down during quiet periods. Always runs across 3 availability zones — no two pods on the same machine.

**Reporter (2–8 pods)** Builds reports, submits to regulators, archives results. Memory-heavy because it assembles large XML documents. Pods never stop mid-submission — the scale-down window is 10 minutes to avoid interrupting an active regulatory submission.

**EOD CronJob (runs at 18:00 UTC weekdays)** Triggers the end-of-day T+1 batch. Has a 2-hour deadline — it must finish before the next trading day opens. Only one instance can run at a time (concurrencyPolicy: Forbid).

---

## Architecture at a Glance

```
Trading Systems
      │
      ▼
  [SQS Queue]  ←────────────────── DLQ → ⚠️ Ops Alert (MiFID: no silent drops)
      │
      ▼
[Ingestor Pods]  ←── Redis (UTI dedup, reference data)
      │
      ├──► Aurora PostgreSQL (trade ledger — writer)
      │
      └──► [Submission Queue]
                │
                ▼
        [Reporter Pods]  ←── Aurora (read replica — reporting queries)
                │
                ├──► Regulatory Trade Repositories (MiFID2 ARM / EMIR TR)
                │
                └──► S3 (reports/ receipts/ rejections/)

```

---

## Quickstart

You need: AWS CLI, kubectl, helm 3.x, Terraform 1.7+. VPN connection for anything
touching production.

### First time — deploy everything

```
# 1. Stand up AWS infrastructure
cd terraform
terraform init
terraform plan -out=tfplan
terraform apply tfplan

# 2. Connect kubectl to the new cluster
aws eks update-kubeconfig \
  --region us-east-1 \
  --name $(terraform output -raw eks_cluster_name)

# 3. Create namespace and inject Aurora endpoints as a K8s secret
kubectl create namespace veritas
kubectl create secret generic veritas-aurora-secret \
  --namespace veritas \
  --from-literal=host=$(terraform output -raw aurora_cluster_endpoint) \
  --from-literal=reader_host=$(terraform output -raw aurora_reader_endpoint)

# 4. Deploy the Helm chart, wiring in Terraform outputs
helm upgrade --install veritas ./helm \
  --namespace veritas \
  --set ingestor.env[0].value=$(terraform output -raw trade_events_queue_url) \
  --set ingestor.serviceAccount.annotations."eks.amazonaws.com/role-arn"=$(terraform output -raw ingestor_role_arn) \
  --set reporter.serviceAccount.annotations."eks.amazonaws.com/role-arn"=$(terraform output -raw reporter_role_arn) \
  --set reporter.env[2].value=$(terraform output -raw regulatory_archive_bucket)

```

### Deploy a new version

```
helm upgrade veritas ./helm \
  --namespace veritas \
  --set ingestor.tag=NEW_VERSION \
  --set reporter.tag=NEW_VERSION \
  --reuse-values

```

### Check what's running

```
kubectl get pods -n veritas
kubectl get hpa -n veritas
kubectl get cronjobs -n veritas
kubectl logs -n veritas -l app.kubernetes.io/component=ingestor --tail=50

```

### Manually trigger the EOD batch (for testing or catch-up)

```
kubectl create job --from=cronjob/veritas-eod-report manual-eod-$(date +%Y%m%d) -n veritas

```

---

## Common Problems

**Trades piling up in the DLQ** This is the most critical alert. Go to SQS in the console and look at the message bodies — they'll contain the trade data and the error. Common causes: schema validation failure (field missing or wrong format), LEI lookup failure (Redis down or GLEIF API unreachable), Aurora write failure. Do not delete messages from the DLQ without first understanding why they failed and reprocessing them — every trade must be accounted for.

**"UTI already exists" errors in ingestor logs** Normal if the ingestor restarted mid-batch. Redis contains the dedup set. If the same UTI is being rejected and it genuinely shouldn't exist, check whether an upstream system is replaying old events.

**Reporter submitting but getting rejections from the trade repository** The trade repository will send back a rejection XML with an error code. The reporter logs these and writes them to s3://veritas-regulatory-archive-*/rejections/. Pull the rejection file, read the error code, cross-reference with the MiFID II technical specification or EMIR validation rules, fix the underlying data, and resubmit. For systematic rejections, loop in Regulatory Affairs.

**EOD batch didn't run / timed out** Check the CronJob history: `kubectl describe cronjob veritas-eod-report -n veritas`. If it timed out (>2 hours), check whether Aurora was under heavy load — the EOD batch is read-heavy on the replica. You can manually re-trigger it (command above). Alert Regulatory Affairs if the batch won't complete before the T+1 deadline.

**kubectl not connecting** The EKS API endpoint is private. You must be on the VPN.

---

## Changing Things

| What you want to change | How to do it |
| --- | --- |
| Aurora instance size | Change in terraform/main.tf, PR, apply via CI |
| Redis node type | Change in terraform/main.tf, PR — note: causes brief failover |
| SQS retention or retry policy | Change in terraform/main.tf, PR |
| Ingestor or reporter pod count (permanent) | Change replicaCount in helm/values.yaml, PR, deploy |
| Regulatory submission endpoint URL | Update MIFID2_ARM_ENDPOINT in the ConfigMap in helm/templates/network-config.yaml — loop in Regulatory Affairs before changing |
| EOD batch schedule | Update reporter.scheduledReport.schedule in values.yaml — loop in Regulatory Affairs |
| Any IAM role or binding change | Edit aws_iam_role_policy.ingestor or .reporter in [main.tf](http://main.tf), PR, apply via CI — Tier 1 requirement: Compliance Engineering must be notified before merge, not after |
| Add a new permission to an existing role | As above — keep scope as narrow as possible, justify in the PR description, tag #compliance-eng for review |

Rule: Terraform changes go through CI, never from a laptop in production. Helm changes
are fine from a local machine in staging.

Tier 1 reminder: Because Veritas is classified Tier 1, IAM changes are not routine
infrastructure changes. If you are unsure whether a permission change is appropriate,
ask in #compliance-eng before opening the PR — not after.

---

## Who to Contact

| Topic | Who |
| --- | --- |
| Infrastructure, AWS, Terraform | #platform-eng |
| Kubernetes, pod issues, Helm | #platform-eng |
| Trade data quality, field mapping | Financial Reporting team |
| Regulatory submission rejections | Regulatory Affairs |
| MiFID II / EMIR compliance questions | Legal & Compliance |
| Incidents, DLQ alerts | On-call (#incidents) |
| Aurora schema changes | Data Engineering + Financial Reporting |
