# aws-dms-table-failure-monitoring

**Serverless observability framework for detecting table-level AWS DMS (CDC) replication failures from CloudWatch logs and alerting on-call teams via SNS.**

---

## Overview

In enterprise data platforms, AWS DMS is commonly used to replicate “hot data” from multiple sources into a centralized data warehouse (Amazon Redshift in this case). While DMS is highly capable, operations teams often face a visibility gap:

* A **single DMS task can include many tables**
* **Table-level failures** (specific table apply/CDC errors) can occur while the task continues running
* Native monitoring and alarms tend to be **task-centric**, making it hard to identify *which table* is failing in time

This project closes that gap by implementing a lightweight, cost-efficient, serverless monitoring layer that **detects table-level failures by scanning DMS CloudWatch logs** and **notifies on-call responders** automatically.

Architecture reference: 

---

## Problem Statement

### What breaks in real-world DMS usage

In “Full load + ongoing CDC” DMS tasks:

* Failures are not always clean “task failed” events.
* Individual tables can produce repeated errors (mapping issues, truncation errors, type mismatches, missing PKs, apply failures, target constraint violations, etc.).
* Without table-level alerting, these errors can go unnoticed until:

  * downstream jobs break,
  * data quality issues are discovered late,
  * analytics users report incorrect dashboards,
  * or an SLA is missed.

### Why this is a big operational risk

* You can have a “running” task that is silently **skipping/looping errors** for one or more tables.
* DMS operational dashboards often don’t provide “table failed” as a first-class alarm signal.
* On-call engineers need a **fast, actionable signal**: *which task, which table (or log stream), what error*.

---

## Why CloudWatch Metrics Alone Are Not Enough

CloudWatch metrics for DMS are useful (latency, throughput, CDC backlog, etc.), but they typically do **not** provide:

* a reliable “table-level failure” metric,
* error context (message patterns, table name, failing step),
* actionable details for triage without opening logs manually.

This framework uses **CloudWatch Logs as the source of truth** for failure signals and turns them into **real-time operational alerts**.

---

## Architecture

High-level flow (from architecture diagram): 

### 1) Task Inventory (Discovery)

The framework supports maintaining a list of DMS task ARNs in S3 (JSON) to track tasks being monitored. Inventory can be populated by:

* a bootstrap process to capture existing tasks
* a task-creation event flow to append new tasks as they are created

*(Some deployments can also choose to discover tasks dynamically via `DescribeReplicationTasks`.)*

### 2) Scheduled Monitoring

An EventBridge schedule triggers the monitoring Lambda on a fixed interval (example: every 15–30 minutes depending on your operational need).

The Lambda:

* reads the list of monitored tasks (from S3 inventory or DMS API)
* derives the CloudWatch log stream name for each task
* scans recent log events (rolling lookback window)
* matches failure/error patterns (example: “Failed”)
* composes a concise, actionable alert payload

### 3) Alerting

If failures are found, the system publishes to an SNS topic that notifies the on-call DBA/operations team.

---

## How This Keeps Systems Stable (Operational Safety)

This design is intentionally **low-risk** and **operationally safe**:

### Controlled load on CloudWatch Logs

* Uses time-bounded scans (example: last 15/30 minutes)
* Avoids long “full history” reads
* Can be tuned with schedule interval + lookback window

### Blast-radius control

* Does not modify DMS tasks
* Read-only inspection of logs + notify
* No impact to replication performance

### Noise reduction patterns (recommended)

To prevent alert fatigue, the framework can be extended with:

* deduping (same task + same error signature within N minutes)
* severity routing (critical tasks → paging; noncritical → email)
* whitelisting/blacklisting known benign patterns

### Least privilege

IAM permissions are limited to:

* reading DMS task metadata (Describe)
* reading CloudWatch logs (Get/Filter)
* publishing to SNS

---

## Cost Efficiency

This is designed to be **very low cost**:

* **No EC2**, no always-on agents, no servers to maintain
* **Pay-per-use** serverless execution
* EventBridge + Lambda + SNS costs scale with monitoring frequency and volume

In practice, this pattern is much cheaper than running persistent monitoring hosts or complex third-party tooling just to detect log-based error conditions.

---

## What You Get (Outcomes)

✅ Faster detection of table-level DMS failures
✅ Reduced downstream impact (data delays / broken pipelines)
✅ On-call receives actionable signal with context
✅ Improved reliability for CDC-based ingestion pipelines
✅ Serverless approach with minimal operational overhead

---

## Repository Contents (Suggested)

Typical structure (adjust to match your repo):

* `cloudformation/` – IaC templates (Lambda, IAM, SNS, EventBridge)
* `lambda/` – monitoring code that scans log streams
* `config/` – sample task inventory JSON (optional)
* `architecture/` – diagram and design references
* `docs/` – runbook + troubleshooting guide

---

## Deployment (High-Level)

1. Deploy the CloudFormation stack(s)
2. Configure:

   * log group name for DMS tasks
   * SNS subscription endpoint(s)
   * schedule interval (EventBridge cron/rate)
3. Ensure DMS tasks have CloudWatch logging enabled
4. Validate by injecting a test pattern / reproducing a known failure and confirming notification

---

## Future Enhancements

If you want to showcase even more maturity (great for interviews), add these as a “Roadmap” section:

* DynamoDB checkpointing (track last processed timestamp per task)
* Alert dedupe + suppression windows
* Publish custom CloudWatch metrics (`DmsTableFailureCount`)
* Slack/MS Teams integration via SNS / Chatbot
* Multi-account aggregation (central monitoring account)
* Tier-based routing (critical tasks → paging; others → email)

---

## Author

Syamprasad Agiripalli
Principal Database Administrator | Cloud Automation Engineer
AWS | RDS | DMS | Redshift | Reliability Automation | Observability

---
