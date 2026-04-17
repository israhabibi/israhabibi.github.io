---
title: "How I Use Claude as a Data Engineer"
date: 2026-04-16
authors:
  - isra
categories:
  - Tooling
tags:
  - ai
  - claude
  - workflow
  - data-engineering
  - productivity
---

# How I Use Claude as a Data Engineer

Claude has become a core part of my day-to-day workflow — not as a chatbot, but as an active collaborator that helps me investigate, build, and document faster.

<!-- more -->

## Context

I'm a data engineer working on a data lakehouse stack (Spark, Airflow, S3, Redshift). My work spans pipeline debugging, schema investigations, SQL writing, and tooling automation. Over the past months I've been integrating Claude deeply into how I work — not just for quick answers, but for multi-step agentic tasks.

---

## 1. Investigating EMR Jobs

One of the most time-consuming parts of my job used to be debugging failed Spark jobs. I'd manually switch between Dynatrace (for logs) and Metabase (for pipeline metadata), cross-referencing job IDs, table names, and error messages.

Now I have a skill called `emr-job-investigator` that Claude follows automatically. I paste a job ID, and Claude:

1. Runs a DQL query against Dynatrace to pull the relevant Spark logs
2. Queries Metabase for pipeline metadata (table name, schedule, ETL config)
3. Synthesizes a summary of what happened — rows inserted, errors, duration

The DQL pattern I use filters out noise from stack traces:

```
fetch logs, from: now()-7d
| filter k8s.cluster.name == "your-cluster"
  AND `kubernetes.labels.emr-containers.amazonaws.com/job.id` == "{job_id}"
| filter NOT matchesPhrase(content, "\tat ")
  AND NOT matchesPhrase(content, "py4j")
| sort timestamp asc
| fields timestamp, content
| limit 100
```

This used to take 20–30 minutes manually. Now it's under 2 minutes.

---

## 2. Pipeline Schedule Checks

Before rescheduling any table, I need to verify that all source tables run *before* the target. This involves dependency mapping — checking which tables feed into which, and what cron schedules they have.

I have a `pipeline-schedule-check` skill for this. I give Claude a table name, and it:

1. Queries Metabase card 10841 (*Dependency Mapping with schedule*) via browser automation + JWT token
2. Parses source tables and their cron schedules
3. Converts everything to WIB (UTC+7) and checks if sources complete before the target starts

This replaced a complicated SQL + Python approach I was building. The pre-built Metabase card does the heavy lifting — Claude just needs to call it correctly and reason about the results.

---

## 3. Building AI-Powered Reporting Tools

I built two reporting tools that extract data from Firebase Performance and CleverTap using Claude's browser automation:

- **`firebase-performance-extractor`** — navigates the Firebase UI and extracts app start time metrics for a given date range
- **`clevertap-extractor`** — extracts funnel and pivot data from CleverTap dashboards
- **`gsheet-report-generator`** — takes a Google Sheets HTML export and generates an interactive HTML report with per-section AI analysis buttons

The honest tradeoff:

| Tool | Pros | Cons |
|---|---|---|
| Firebase/CT extractors | No API needed, no waiting for data pipelines | Requires browser session open, slow to load |
| GSheet report generator | Auto-generates narrative summaries | Manual download + upload step, not fully automated |

I demoed this to my EM as "AI-accelerated development" — the pitch was honest: these tools move fast but have rough edges. The value is shipping something useful in days instead of sprints.

---

## 4. SQL Debugging and Refactoring

A lot of my day involves writing or debugging complex SQL — CTEs with 10+ steps, Spark SQL with tricky date expressions, Redshift DDL.

Claude is particularly useful for:

- **Tracing NULL propagation** — I paste a CTE chain and ask why a specific column is null for a given ID. It traces the join path and identifies where the row gets filtered out.
- **Date expression translation** — converting between `date_format(timestamp(...), 'yyyy-MM-dd')` Spark patterns and standard SQL equivalents
- **Schema reformatting** — taking a column definition list and restructuring it to match a required JSON schema template

The workflow is conversational: I paste the query, describe the symptom, and iterate. Claude doesn't just answer — it explains *why*, which means I learn the pattern and catch it myself next time.

---

## 5. Desktop Commander + Local File Access

One of the biggest unlocks was connecting Claude to my local filesystem via Desktop Commander. This means Claude can:

- Read and write files directly to my MkDocs blog (like this post)
- Run shell commands on my Mac
- Interact with local tools and scripts

Combined with the skills system (markdown files that tell Claude *how* to approach specific tasks), this has turned Claude into something closer to a local agent than a chatbot.

---

## Key Takeaways

- Claude works best when it has **context** — skills, past conversation history, and tool access compound each other
- **Browser automation** (Chrome extension + MCP) fills the gap where APIs don't exist
- The skills system is worth investing in — a well-written SKILL.md file saves 10+ minutes on every repeated task
- Be honest about limitations when demoing AI tools internally — rough edges don't kill adoption, overselling does
- The biggest productivity unlock isn't any single tool — it's having Claude maintain continuity across investigations that would otherwise require switching between 4–5 different interfaces
