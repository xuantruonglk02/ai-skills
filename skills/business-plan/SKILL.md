---
name: business-plan
description: >
  Generates a structured business plan document for a new feature. Use this skill whenever the user wants to plan a new feature, document business requirements, describe what a feature should do, create a business plan, or says things like "I want to plan a new feature", "help me write a business plan for X", "I need to document the requirements for X", "let's plan out this feature", or "tôi muốn lên kế hoạch cho tính năng". Automatically reads project overview, finds related existing plans, and produces a detailed business_plan.md ready to feed into the next step of technical/code planning.
---

# Business Plan Skill

You help the user generate a well-structured, detailed business plan document for a new software feature.

The plan is written in **English**, lives at `docs/feature/{feature_key}/business_plan.md`, and is detailed enough to serve as the primary input for the next step — creating a technical/code implementation plan.

> **Important:** The project may have a `docs/templates/` directory or other template files. Do NOT use those. Use the document structure defined in this skill (below). The structure here is intentionally different — it requires more detail than a typical template and is tuned for feeding into code planning.

---

## Step 1: Gather Context

Build a thorough understanding of the project and the feature before writing anything.

### 1a. Read project overview (in this order)

1. `AGENTS.md` at the project root — this is the primary guide for AI agents; read it first
2. `.memory-bank/` directory — read all files here, especially `brief.md`, `rules.md`, and `product.md`
3. `CLAUDE.md` at the project root (if present)
4. `docs/` — any overview or architecture files (but **not** template files in `docs/templates/`)

These files define the system's role hierarchy, data patterns, guard conventions, and domain concepts that must be reflected accurately in the plan.

### 1b. Find related existing feature plans

Search `docs/feature/` for existing `business_plan.md` files. Read those related to the user's request. This tells you:
- What features already exist and how they're described
- Dependencies this new feature might have
- The numbering convention for feature IDs (e.g., `1_s3_frame_layout`, `2_bulk_user_import`)

### 1c. Clarify if needed

If the user's description lacks enough detail, ask focused questions about what's missing — the problem being solved, who the users are, key business rules, constraints. Don't ask for information you can infer from the codebase or the user's message.

---

## Step 2: Define the Feature Key

Derive a short English identifier:
- Format: `feature/key_word_of_task` (lowercase, underscores)
- Examples: `feature/contact_form_submission`, `feature/photo_session_history`, `feature/bulk_user_import`
- Match the numbering convention if one is in use (e.g., `feature/3_photo_session_history`)
- Output file: `docs/feature/{key_word}/business_plan.md`

---

## Step 3: Resolve Contradictions and Ambiguities Before Writing

As you analyze the feature against the existing system, you will sometimes encounter:
- Requirements that conflict with existing business logic or data models
- User descriptions that are ambiguous or could be interpreted in more than one way
- Scope that overlaps with an existing feature in a non-obvious way
- Constraints from the codebase that make a requested behavior difficult or impossible

**Before writing the plan**, surface these to the user and confirm their intent. Ask one focused question at a time. Do not write the plan while open contradictions remain — a plan built on an unresolved assumption will create problems in the technical planning step.

If you reach the writing step and some items are still unresolved (e.g., the user chose to proceed anyway), document them in the plan using the format:

```
> ⚠️ **UNRESOLVED:** [describe the contradiction or ambiguity]. Needs confirmation before implementation.
```

Place these callouts immediately after the section or requirement they affect.

---

## Step 4: Write the Business Plan

Use **exactly this document structure**. Every section listed here must appear in the output. Do not substitute sections from any project template you found.

```
# Business Plan: {Feature Title}

## Document Overview
(table: Feature ID, Git Branch, Status, Created date, Owner)

## Overview
(2–4 sentences: what, why, how at high level)
Goals: bulleted list
Scope: In scope / Out of scope

## User Flow Overview
(narrative walkthrough of the main user journey; who does what and sees what)
Main flow: numbered steps
Alternative flows: what happens on errors, edge paths, cancellations

## Table of Contents
(links to all sections below)

## 1. Business Context & Objective
  ### 1.1 Problem Statement
  ### 1.2 Strategic Objective
  ### 1.3 Stakeholders (table: stakeholder, role, interest)

## 2. Detailed Plan
  ### 2.1 {Functional Area — e.g., "Data Model", "API Endpoints", "Admin UI", "Notifications"}
    Functional Requirements: (FR-01, FR-02... — be concrete, include field names, types, limits)
    Business Rules: (BR-01, BR-02... — rules the system must enforce)
    Acceptance Criteria: (table: #, criterion, verification method — must be testable)
    Edge Cases: (table: scenario, expected behavior)
  
  ### 2.2 {Another Functional Area}
    (same structure)
  
  (add as many 2.X sections as needed — at least 2, typically 3–6)

## 3. Non-Functional Requirements
(Performance, Scalability, Security, Availability, Compliance)

## 4. Constraints & Dependencies
(Dependencies: what must exist first; Constraints: limits on the solution; Risks table)

## 5. Success Criteria & KPIs
(table: KPI, target, how to measure)

## 6. Impact Analysis
(table: affected area, nature of impact, risk level, notes)
(Be thorough — cover all modules, APIs, data models, or services this feature touches)

## 7. Related Features & Plans
(table: feature ID, description, relationship, link)
```

### Highlighting rules

Use these consistently so readers can scan the plan quickly:

- **`> ⚠️ UNRESOLVED:`** — an open contradiction or ambiguity that must be confirmed before implementation
- **`> 🔴 HIGH IMPACT:`** — a decision or requirement that is hard to reverse, affects multiple modules, or carries significant risk. Call these out at the top of the relevant section or requirement.
- **`> 📌 NOTE:`** — an assumption you made that the user should verify, or a technical detail that is non-obvious

Use these callouts sparingly — if everything is highlighted, nothing is. Reserve them for things that genuinely need attention.

### What makes the Detailed Plan (section 2) good

This section is the most important — it's what a developer or agent reads to create a technical implementation plan. Make it thorough:

- **At least 2 functional areas**, typically 3–6. Split by concern: API layer, data model, UI, notifications, auth, etc.
- **Functional Requirements** are concrete: not "user can upload files" but "user can upload CSV files up to 5MB; accepted MIME type: text/csv"
- **Business Rules** capture logic constraints: limits, state transitions, permissions, validation rules
- **Acceptance Criteria** are testable statements, not goals. Each should have a clear pass/fail.
- **Edge Cases** cover failure modes, boundary conditions, and non-obvious scenarios. At least 2–3 per area.
- **Role names must be precise**: use the correct role name from the system (`Admin` = super admin / Lumilab team; `Manager` = tenant owner; `Employee` = store staff). Never use "admin" to mean "manager".

For the **Overview**: make it scannable in 30 seconds by someone seeing the feature for the first time.

For the **Impact Analysis**: be explicit about what existing code is affected. Even "additive only, no changes to X" is useful to document.

For **placeholders**: if something is genuinely unclear, write `[TO CLARIFY: ...]` so the user knows what needs filling in before technical planning starts.

---

## Step 5: Write the File

Create the directory and write:
```
docs/feature/{feature_key}/business_plan.md
```

After writing, confirm the path, briefly summarize what was captured, and call out:
- Any `⚠️ UNRESOLVED` items that need user decision before technical planning
- Any `🔴 HIGH IMPACT` areas the user should review carefully
- Any `[TO CLARIFY]` placeholders you left

---

## Reference

For detailed examples of each section, read `assets/template.md` in this skill's directory.
