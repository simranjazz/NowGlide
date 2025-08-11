---
title: "Automating HR Background Screening with ServiceNow HRSD - NowGlide"
share-title: "Automating HR Background Screening with ServiceNow HRSD - NowGlide"
subtitle: "Designing and implementing a background screening process in ServiceNow leveraging its powerful automations and experiences"
description: Learn how to build a fully automated HR background screening process in ServiceNow HRSD. From database design and decision tables to flows, playbooks, and portals, this guide walks you through a real-world implementation while staying close to OOTB capabilities.
date: 2025-08-10
last-updated: 2025-08-10
tags: [expert, HRSD, background-screening, guides]
cover-img: ["/assets/images/posts/servicenow-hrsd-background-screening-guide/BGSCoverImage.png"]
thumbnail-img: "/assets/images/posts/servicenow-hrsd-background-screening-guide/BGSCoverImage.png"
head-extra: head-custom.html
readtime: true
css: "/assets/css/post.css"
author: "Gursimran Singh Saini"
show-avatar: true
permalink: /servicenow-hrsd-background-screening-guide/
published: true
---

* TOC
{:toc}

Background Screening (BGS) is a cornerstone of modern hiring processes, ensuring compliance, risk management, and candidate credibility. Yet in many organizations, it remains a patchwork of manual follow ups, emails, and disconnected spreadsheets.  

ServiceNow HRSD (Human Resources Service Delivery) offers all the foundational building blocks to automate this process, but connecting them into a cohesive, scalable, and user friendly system requires thoughtful design, that’s where I come in. :) I have previously had the privilege of implementing this very extensive automation in another SaaS product, and I’ve brought forward some of those tried and tested design principles.

This post documents my **end to end approach** to implementing a Background Screening process in HRSD. It’s based on real world experience, with minimal customization, and heavy use of out-of-the-box (OOTB) capabilities.

---

## Why automate BGS in ServiceNow?

Typical BGS pain points:
- **Inefficiency** - Manual task tracking slows down processing.
- **Disjointed communication** - HR, vendors, and candidates operate in separate channels.
- **Lack of transparency** - Candidates often don't know their status; HR teams lack a single view of progress.
- **Compliance risks** - Without audit trails, you can’t prove due diligence.

Automating in HRSD solves this by:
- Centralizing all data in the Now Platform (No, ServiceNow AI Platform, as it was recently renamed)
- Triggering flows automatically based on candidate intake
- Guiding HR agents with structured Playbooks
- Allowing candidate self-service via a secure portal
- Maintaining auditable records

---

## High-Level Design Principles

1. **Stay close to OOTB** - Only build custom when business rules demand it.
2. **Separate transactional from master data** - Avoid losing important results to case archival.
3. **Design BGS Candidate records as "instances of candidature"** -  
   Each record in `BGS_Candidate` represents a single screening instance for a person. This means one person may have multiple `BGS_Candidate` records over their lifetime with your organization, for example, if they are rehired years later and require a fresh screening.
4. **Keep scalability in mind** - Accommodate future needs like multi region differences.
5. **Design for all personas** - The process should be smooth for HR agents, candidates, and HR vendors.

---

## Step 1 - Create the Custom Application

Using **ServiceNow Studio** (or App Engine Studio), create a scoped application called **HRSD Background Screening**.  
This keeps all custom components isolated from standard HRSD configurations, and the rest of the platform.
![ServiceNow create new BGS app](/assets/images/posts/servicenow-hrsd-background-screening-guide/servicenow-hrsd-new-app.png)

During new app creation, the system might prompt you to define the roles that will be interacting with this app — typically HR BGS teams and admins.
![ServiceNow create new roles](/assets/images/posts/servicenow-hrsd-background-screening-guide/servicenow-hrsd-roles.png)

---

## Step 2 - Design the Database Schema

ServiceNow HRSD already has rich tables for HR cases, profiles, and documents. For BGS, you’ll extend this with **custom tables** to meet unique needs.

### Core Custom Tables

| Table | Purpose | Notes |
|-------|---------|-------|
| **BGS Candidate** | Stores candidate level data received from integration (like Taleo) or manual entry. | Represents **one screening instance** for a person. A single person may have multiple candidate records over time if re-screened. |
| **BGS Self Declaration** | Captures self-declaration data directly from the candidate, through an external portal (more on this in the next blog post). | One-to-many relationship with Candidate. |
| **BGS Employment History** | Previous employment records declared by the candidate. | Extend from `sn_hr_profile_employment` so you inherit OOTB fields & ACLs. One-to-many relationship with BGS Self Declaration. |
| **BGS Education History** | Educational records declared by the candidate. | Extend from `sn_hr_profile_education`. One-to-many relationship with BGS Self Declaration. |
| **BGS Address History** | Historical residential addresses. | Purely custom, not available OOTB. One-to-many relationship with BGS Self Declaration. |
| **BGS Criminal Disclosure** | Candidate's self-reported criminal records. | Purely custom. One-to-many relationship with BGS Self Declaration. |
| **BGS Outcome** | Stores the **final outcome** of the screening process. | This is master data, retained beyond the lifecycle of cases. One-to-many relationship with BGS Self Declaration. |

HRSD BGS custom database tables:
![ServiceNow create HRSD BGS custom database tables](/assets/images/posts/servicenow-hrsd-background-screening-guide/servicenow-hrsd-bgs-db.png)

Enabling extensibility in standard HRSD tables:
![ServiceNow HRSD enable extensibility](/assets/images/posts/servicenow-hrsd-background-screening-guide/servicenow-hrsd-enable-extensibility.png)

---

### Why the Outcomes Table is Critical

HR cases (`sn_hr_core_case`) are **transactional** in nature, subject to archival or purging once resolved. However, the final screening decision is **long term reference data**.

If you don’t separate the outcome from the case, you risk losing the decision record when the case is archived. By storing it in `BGS Outcomes`:
- You preserve the **final result** (e.g., “Cleared”, “Cleared with Conditions”, “Rejected”).
- You retain **decision metadata** (date, rationale, reference links).
- You keep an immutable reference tied to the relevant `BGS Candidate` record.
- You can report historically without relying on transactional data.

Think of `BGS Candidates` + `BGS Outcomes` as your **BGS master data** set.

Records in BGS Outcomes table would be created automatically, through a daily cron job (ServiceNow > System Definition > Scheduled Jobs) that looks for BGS Cases closed on the previous day, and creates the BGS Outcomes records.

---

### Relevant Standard Tables

Where possible, reuse standard HRSD structures:
- `sn_hr_integrations_background_check_package` - Packages of checks (from **com.sn_hr_integrations_background_check** plugin)
![ServiceNow HRSD BGS Packages](/assets/images/posts/servicenow-hrsd-background-screening-guide/servicenow-hrsd-bgs-packages.png)
- `sn_hr_core_case` - The main HR Case table for BGS case records
- `sn_hr_core_profile` - Candidate and other (More on this in the next blog post) external persona profiles
- `sn_hr_core_case_task` - Stores individual screening tasks under one sn_hr_core_case record
- `sn_hr_case_document_type` - Master data for document types of candidates throughout the BGS lifecycle
![ServiceNow HRSD BGS Document Types](/assets/images/posts/servicenow-hrsd-background-screening-guide/servicenow-hrsd-document-types.png)

---

## Step 3 - Decision Tables

Decision Tables let you externalize screening rules from the flow logic. Complex decision making is separated so that the flow becomes easier to manage and more readable.

**Two key decision tables:**
1. **BGS Packages Decision Table** - Selects the screening package based on candidate intake data (region, job level, etc.).
2. **BGS Tasks Decision Table** - Determines the specific tasks for the selected package.

HRSD BGS Packages decision table:
![ServiceNow create HRSD BGS Packages decision table](/assets/images/posts/servicenow-hrsd-background-screening-guide/servicenow-hrsd-package-decision-table.png)

HRSD BGS Tasks decision table:
![ServiceNow create HRSD BGS Tasks decision table](/assets/images/posts/servicenow-hrsd-background-screening-guide/servicenow-hrsd-task-decision-table.png)

---

## Step 4 - Build the Main Flow

**Trigger:**  
A new `BGS Candidate` record is created (via ATS integration or manual entry).

**Flow Steps:**
1. Run **BGS Package Decision Table**
2. Update candidate record with selected HR background screening package
3. Create a new HR Core Case (type: Background Screening)
4. Run **BGS Task Decision Table**
5. Create individual screening tasks
6. Assign the case (skills based routing)
7. Notify the assigned agent

**Note:** If reading/creating/updating HRSD standard tables from the custom app scope, you would have to create a cross-scope privilege record and then approve the Restricted Caller Access Privilege records (run time cross application access) for safe CRUD operations.

Raise a new cross-scope privilege record:
![ServiceNow create cross-scope privilege](/assets/images/posts/servicenow-hrsd-background-screening-guide/servicenow-hrsd-cross-scope-privilege.png)

Approve the corresponding restricted caller access privilege record:
![ServiceNow approve restricted caller access privilege records](/assets/images/posts/servicenow-hrsd-background-screening-guide/servicenow-hrsd-rcap.png)

---

## Step 5 - Playbook Experience for Agents

Playbooks offer a **guided, contextual** process for agents, ideal for multi step screenings. Playbooks would help guide newly recruited HR BGS agents in processing the background screening case in a guided manner, avoiding human errors and at the same time training the new agents.

**Approach:**
- One Playbook per major region (APAC, AMER, EMEA)
- Playbook guidance covers:
  - Priority order for checks
  - Links to vendor systems
  - Data points to confirm
  - Escalation criteria
  - Where and how to record the final outcome

Example: **APAC Playbook Steps**:
1. Review self-declaration
2. Run criminal record check
3. Verify employment history
4. Validate education credentials
5. Confirm address history
6. Record decision in `BGS Outcome` (Automatic, through a cron job)
7. Close tasks and case

---

## Step 6 - AI & Automation

Out-of-the-box AI features you can leverage:
- **NowAssist** - Suggest KBs, related cases
- **Predictive Intelligence** - Categorization & assignment
- **Gen AI** - Draft task/case summaries or communications

---

## Step 7 - Candidate Interaction via External Portal

High-level flow:
- Candidate logs in to external service portal (secured through multi-factor authentication)
- Submits self-declaration & documents
- Tracks progress
- Exchanges messages with HR securely
- ACLs ensure they see only their own data

---

## Step 8 - Compliance & Data Retention

- **Transactional vs Master Data** - Keep outcomes in `BGS Outcomes`
- **Audit Trails** - Full change history
- **PII Security** - ACLs + encryption at rest
- **Regional Compliance** - Adapt to GDPR, PDPA, etc.

---

## Wrapping Up

Following this architecture gives you:
- A scalable, automated BGS process
- Clear separation of master & transactional data
- Playbook guided agent work
- Candidate friendly self-service
- Minimal custom code

**Next Post:** How to design the HR External Portal for self-declaration, document uploads, and secure candidate case interactions. The HR External Portal would cater to four personas: candidates, HR vendors, employees on long leaves of absence and former employees.

{{ site.post_footer_author }}

---

{% if site.share-buttons %} {% include social-share.html %} {% endif %}

*Have thoughts to share? Join the discussion below!*

{% include giscus-comments.html %}