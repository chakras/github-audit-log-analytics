# GitHub Audit Log Analytics on Microsoft Fabric

Build a powerful, near real-time analytics solution for GitHub audit logs using Microsoft Fabric's Real-Time Intelligence (RTI).

![Solution Overview](GH+Fabric.jpg)

## Table of Contents

- [Overview](#overview)
- [Understanding GitHub Audit Logs](#understanding-github-audit-logs)
- [Why Microsoft Fabric?](#why-microsoft-fabric)
- [Solution Architecture](#solution-architecture)
- [Step-by-Step Setup Guide](#step-by-step-setup-guide)
  - [1. Provision an EventStream in Fabric](#1-provision-an-eventstream-in-fabric)
  - [2. Configure Audit Log Streaming from GitHub](#2-configure-audit-log-streaming-from-github)
  - [3. Preview Logs on EventStream](#3-preview-logs-on-eventstream)
  - [4. Configure EventHouse as Target](#4-configure-eventhouse-as-target)
  - [5. Data Sampling and Testing](#5-data-sampling-and-testing)
- [Medallion Architecture — Silver and Gold Layers](#medallion-architecture--silver-and-gold-layers)
  - [Silver Layer Tables](#silver-layer-tables)
  - [Gold Layer Tables (Materialized Views)](#gold-layer-tables-materialized-views)
  - [Validate Data in Silver and Gold Layers](#validate-data-in-silver-and-gold-layers)
- [Real-Time Dashboard](#real-time-dashboard)
- [Data Agent](#data-agent)
- [Future Enhancements](#future-enhancements)

---

## Overview

GitHub audit logs capture a detailed record of every significant action within your GitHub organization or repositories. Each log entry records **who** performed an action, **what** they did, **when** it happened, and the **relevant context** (e.g., which repository or setting was affected).

Consolidating these logs into an analytics platform unlocks comprehensive **security, compliance, and operational insights**. This solution streams GitHub audit logs into Microsoft Fabric in near real-time and transforms them through a medallion architecture for dashboarding and analysis.

## Understanding GitHub Audit Logs

Key categories of events captured include:

| Category | Examples |
|---|---|
| **User Access Events** | User sign-ins, authentication attempts (username, timestamp) |
| **Repository Changes** | Repo creation/deletion, branch pushes and merges, branch protection rule changes |
| **Permission & Membership Updates** | User additions/removals in orgs or teams, repository access permission changes |
| **Security/Settings Modifications** | Security policy updates, webhook configurations, PAT approvals, settings changes |

For example, when an administrator adds a new team member or a developer pushes code, the audit log records the actor's identity, the action performed, the target repository/organization, and the timestamp. This granular event data helps administrators track user activity, ensure security compliance, and audit changes within GitHub.

## Why Microsoft Fabric?

**Microsoft Fabric** is an AI-powered, end-to-end analytics platform that unifies data engineering, data warehousing, real-time analytics, and business intelligence. Built on a SaaS foundation, it integrates services like Power BI, Data Factory, and Synapse into a single ecosystem — simplifying governance, accelerating insights, and enabling scalable, secure, enterprise-ready data solutions.

**Real-Time Intelligence (RTI)** within Fabric enables ingestion and analysis of high-volume event data through EventStreams. This solution leverages GitHub's **Audit Log Streaming** capability to continuously funnel events into Fabric and builds analytical layers using RTI.

## Solution Architecture

Below is the end-to-end solution architecture:

![Solution Architecture](Designer.png)

---

## Step-by-Step Setup Guide

### 1. Provision an EventStream in Fabric

GitHub natively supports streaming audit logs in near real-time. Fabric's EventStream provides an **Azure Event Hub endpoint** where GitHub audit logs are ingested — eliminating the need to provision a separate Azure Event Hub namespace.

1. Create an EventStream in your Fabric workspace.
2. Choose **Custom Endpoint** as the source and publish it.
3. Click on the published EventStream and **note down the Event Hub connection details**.

![EventStream Raw](EventStream-raw.jpg)

### 2. Configure Audit Log Streaming from GitHub

> **Note:** Administrator privileges are required to access the audit log section.

1. Log into your GitHub account and click on the user icon.
2. Open **Enterprise** and navigate to:
   **Settings → Audit Log → Log Streaming**
3. Create a new streaming job by entering the Event Hub details captured in the previous step.
4. Start the stream.

![GitHub Log Streaming Settings](github-log-streaming-settings.png)

### 3. Preview Logs on EventStream

Once the streaming job is running, return to the Fabric EventStream and click on it to preview incoming data.

> It typically takes a few minutes for the first logs to arrive. Be patient and refresh the data preview window.

![Source Data Preview](source-data-preview.png)

### 4. Configure EventHouse as Target

**EventHouse** is RTI's analytics-optimized database for streaming data. Built on Azure Data Explorer's **Kusto** technology, it is designed for massive scale and fast queries on log and semi-structured data.

1. Click **Edit** mode on the EventStream to add details for the EventHouse, the KQL Database, and the target table.
2. Click **Publish**. Upon successful completion, preview the data by clicking the EventHouse target on the EventStream job.

![EventHouse Target Settings](eventhouse-target-settings.png)

### 5. Data Sampling and Testing

1. Open the KQL Database in your EventHouse and click the target table for quick data validation.
2. Open a new **KQL Queryset** from the top of the screen.

![KQL Queryset Creation](kql-queryset-creation.png)

Run the following queries to verify data is loading in real-time:

```kql
// View a sample of records
YOUR_TABLE_HERE
| take 100

// Count total records
YOUR_TABLE_HERE
| count

// Check ingestion rate (every 5 minutes)
YOUR_TABLE_HERE
| summarize IngestionCount = count() by bin(ingestion_time(), 5m)
```

![EventHouse Data Preview](eventhouse-data-preview.png)

---

## Medallion Architecture — Silver and Gold Layers

The entire data transformation from raw through gold layer operates in **near real-time**.

### Silver Layer Tables

Silver layer tables are created using KQL Database **update policies**. These policies automatically transform and route new data as it arrives in the raw table.

> **Reference:** [Tutorial: Route Data Using Table Update Policies - Kusto | Microsoft Learn](https://learn.microsoft.com/en-us/kusto/management/update-policy-tutorial)

![Raw Table Schema](raw-table-schema.png)

**Create silver layer tables:**

```kql
.create table silver_action (
    action: string,
    action_category: string,
    action_name: string,
    operation_type: string,
    timestamp: datetime
)

.create table silver_actor (
    actor_id: long,
    actor: string,
    timestamp: datetime
)

.create table silver_org (
    org: string,
    org_id: long,
    business: string,
    timestamp: datetime
)

.create table silver_github_logs based-on YOUR_RAW_TABLE
```

**Create transformation functions and update policies:**

```kql
.execute database script <|

.create-or-alter function Get_Action() {
    YOUR_RAW_TABLE
    | extend
        action = tostring(action),
        action_category = tostring(extract(@"^([^.]+)", 1, action)),
        action_name = tostring(extract(@"\.([^.]+)$", 1, action)),
        operation_type = tostring(operation_type),
        timestamp = unixtime_milliseconds_todatetime(timestamp)
    | project action, action_category, action_name, operation_type, timestamp
}

.create-or-alter function Get_Actor() {
    YOUR_RAW_TABLE
    | extend
        actor_id = tolong(actor_id),
        actor = tostring(actor),
        timestamp = unixtime_milliseconds_todatetime(timestamp)
    | project actor_id, actor, timestamp
}

.create-or-alter function Get_Org() {
    YOUR_RAW_TABLE
    | extend
        org = tostring(org),
        org_id = tolong(org_id),
        business = tostring(business),
        timestamp = unixtime_milliseconds_todatetime(timestamp)
    | project org, org_id, business, timestamp
}

.create-or-alter function Get_GitHub_Logs() {
    YOUR_RAW_TABLE
}
```

**Attach update policies to silver tables:**

```kql
.execute database script <|

.alter table silver_github_logs policy update
    "[{\"IsEnabled\":true,\"Source\":\"YOUR_RAW_TABLE\",\"Query\":\"Get_GitHub_Logs\",\"IsTransactional\":false,\"PropagateIngestionProperties\":true,\"ManagedIdentity\":null}]"

.alter table silver_action policy update
    "[{\"IsEnabled\":true,\"Source\":\"YOUR_RAW_TABLE\",\"Query\":\"Get_Action\",\"IsTransactional\":false,\"PropagateIngestionProperties\":true,\"ManagedIdentity\":null}]"

.alter table silver_actor policy update
    "[{\"IsEnabled\":true,\"Source\":\"YOUR_RAW_TABLE\",\"Query\":\"Get_Actor\",\"IsTransactional\":false,\"PropagateIngestionProperties\":true,\"ManagedIdentity\":null}]"

.alter table silver_org policy update
    "[{\"IsEnabled\":true,\"Source\":\"YOUR_RAW_TABLE\",\"Query\":\"Get_Org\",\"IsTransactional\":false,\"PropagateIngestionProperties\":true,\"ManagedIdentity\":null}]"
```

> **Note:** The attributes of your raw table will dictate how many silver and gold layer tables you need. Adjust the scripts to match your schema.

### Gold Layer Tables (Materialized Views)

Gold layer tables use KQL Database **materialized views** to provide deduplicated, aggregated data.

> **Reference:** [Create Materialized View - Kusto | Microsoft Learn](https://learn.microsoft.com/en-us/kusto/management/materialized-views/materialized-view-create-or-alter)

```kql
.create materialized-view with (backfill=true) gold_action on table silver_action {
    silver_action
    | summarize arg_max(timestamp, *) by action, action_category, action_name, operation_type
}

.create materialized-view with (backfill=true) gold_actor on table silver_actor {
    silver_actor
    | summarize arg_max(timestamp, *) by actor_id, actor
}

.create materialized-view with (backfill=true) gold_org on table silver_org {
    silver_org
    | summarize arg_max(timestamp, *) by org, org_id, business
}
```

### Validate Data in Silver and Gold Layers

Once the silver and gold layer tables are created, query them the same way you queried the raw table. It generally takes a few minutes for data to appear in the preview.

The full lineage of the raw → silver → gold real-time pipeline is visible from the **Entity Diagram (Preview)** button on the KQL Database page.

![Entity Diagram](entity-daigram.png)

---

## Real-Time Dashboard

Microsoft Fabric RTI provides built-in **real-time dashboards** out of the box.

> **Reference:** [Create a Real-Time Dashboard - Microsoft Fabric | Microsoft Learn](https://learn.microsoft.com/en-us/fabric/real-time-intelligence/dashboard-real-time-create)

![Real-Time Dashboard](pbi-report.jpg)

The KQL queries used to build each visual are available in this repository under `KQL-Real-Time-Dashboard-Queries.kql`.

## Data Agent

You can also build a **conversational AI layer** on top of your GitHub audit logs using Microsoft Fabric Data Agents.

> **Reference:** [Fabric Data Agent Creation - Microsoft Fabric | Microsoft Learn](https://learn.microsoft.com/en-us/fabric/data-science/concept-data-agent)

## Future Enhancements

This solution can be customized and extended with additional capabilities:

- **Build an ontology and graph database** for relationship-based analysis
- **Power BI reports** for self-service BI users
- **Data pipelines** to extract and load GitHub Copilot metrics
- **Dashboards for GitHub Copilot adoption** tracking and insights

---

## License

This project is provided as-is for educational and reference purposes.
