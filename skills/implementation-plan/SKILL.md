---
name: implementation-plan
description: >
  Converts a business_plan.md into a detailed implementation_plan.md that guides an AI agent to make code changes with minimal additional thinking required. Use this skill whenever the user wants to create an implementation plan from a business plan, says things like "tạo implementation plan", "create implementation plan", "lên implementation plan", "generate implementation plan", "turn the business plan into a coding plan", or references a business_plan.md and wants a technical coding guide. This skill reads the business plan, explores the relevant codebase, collaborates with the human to resolve contradictions, and produces a thorough plan in git diff format so a coding agent can follow it step by step with minimal ambiguity. Always use this skill — do not improvise an implementation plan from scratch without it.
---

# Implementation Plan Skill

You help the user convert a business plan into a detailed, code-ready **implementation plan** — a document structured so that an AI coding agent can read it and make changes with minimal additional thinking.

The plan is written in **English**, lives in the **same directory as the input `business_plan.md`**, and is saved as `implementation_plan.md`.

> **Important:** The project may have a `docs/templates/` directory or other template files. Do NOT use those. Follow the structure defined in this skill.

---

## Step 1: Read Project Context

Before doing anything else, build a complete understanding of the project. Read these in order:

1. `AGENTS.md` at the project root — primary guide for AI agents; read it first
2. `.memory-bank/` directory — read all files here, especially `brief.md`, `rules.md`, and `product.md`
3. `CLAUDE.md` at the project root (if present)
4. `docs/` — any overview or architecture files (but **not** template files in `docs/templates/`)

These files define the system's role hierarchy, data patterns, guard conventions, naming rules, and domain concepts. You must understand them before analyzing code or proposing changes.

---

## Step 2: Read the Business Plan

Ask the user which `business_plan.md` to use if they haven't specified the path. Then read the file thoroughly.

Extract from the business plan:
- The feature being built and the problem it solves
- Functional requirements, business rules, and acceptance criteria
- Affected modules, APIs, and data models (from the Impact Analysis section)
- Any `⚠️ UNRESOLVED` items or `[TO CLARIFY]` placeholders already present

The implementation plan lives **in the same directory as the business plan**. E.g., if the business plan is at `docs/feature/async_media_download/business_plan.md`, output to `docs/feature/async_media_download/implementation_plan.md`.

---

## Step 3: Analyze the Codebase

Explore the code relevant to the feature. Focus on:

- Existing files and modules the feature will touch
- Current API routes, controllers, services, and their signatures
- Data models (entities, schemas, DTOs) — field names, types, constraints
- Patterns the codebase follows (naming, error handling, auth guards, decorators)
- Any tests that cover the areas being changed

You need enough depth to write precise `git diff`-style code changes. Shallow reading leads to wrong function signatures, wrong field names, or missed guard decorators — all of which force the coding agent to improvise.

---

## Step 4: Surface and Resolve Contradictions

As you analyze the business plan against the codebase, you will encounter:
- Requirements that conflict with how the existing system works
- Ambiguities the business plan left open that affect code decisions
- Scope that overlaps with existing features in a non-obvious way
- Technical constraints that make a requested behavior difficult or impossible

**Do not self-decide on these.** Raise each contradiction or ambiguity with the user and confirm their intent before writing the plan. Ask one focused question at a time.

If the user chooses to proceed with unresolved items, record them clearly in the plan (see Section 6 of the template). When the user later provides clarification, update the plan and mark the item resolved.

> **Coding gate:** If the user asks you to code according to this implementation plan and the plan still contains `⚠️ UNRESOLVED` items, **stop and surface them before coding**. Do not proceed until all contradictions are resolved or the user explicitly overrides.

---

## Step 5: Write the Implementation Plan

Use the document structure defined below. Every section must appear in the output. For detailed examples of each section, read `assets/template.md` in this skill's directory.

---

## Document Structure

```
# Implementation Plan: {Feature Title}

## Document Overview
(table: Feature ID, Git Branch, Version, Status, Created date, Author, Related Documents)

## Overview
(what changes, which modules/files/features are impacted — scannable in 30 seconds)
Impact Summary: bulleted list of impacted areas

## Table of Contents
(links to all sections below)

## 1. Routes Summary
(table of new/modified routes; omit section if no route changes)

## 2. Detailed Code Changes
(the core section — ALL changes in git diff format, with explanations)

## 3. Impact Analysis
(summary of impact on existing features, risks, and watch-outs)

## 4. Unresolved Contradictions
(⚠️ items still open; ✅ items resolved after clarification)

## 5. References
(link to business_plan.md; links to other related implementation plans)
```

---

## Section-by-Section Writing Guide

### Document Overview

Match the style of the business plan's Document Overview table:

| Field | Value |
|-------|-------|
| **Feature ID** | `feature/{feature_key}` |
| **Git Branch** | `feature/{feature_key}` |
| **Version** | `1.0` |
| **Status** | Draft |
| **Created** | {YYYY-MM-DD} |
| **Author** | {Name / Role} |
| **Related Documents** | [Business Plan](./business_plan.md) |

### Overview

2–4 sentences describing what this implementation plan covers, which modules are affected, and what the net change to the codebase is. Follow with a bulleted **Impact Summary** listing every file or module that will change.

### Section 1: Routes Summary

Only include this section if the feature adds or modifies HTTP routes. Use a table:

| Method | Path | Handler | Auth | Change Type | Notes |
|--------|------|---------|------|-------------|-------|
| `POST` | `/api/v1/...` | `ControllerName.method` | Guard name | New / Modified | Brief note |

If no route changes: write `> No route changes in this implementation.` and skip the table.

### Section 2: Detailed Code Changes

This is the most important section. It must contain every change the coding agent needs to make, organized by file. For each file:

1. **State the file path** as a heading (e.g., `#### src/modules/media/media.service.ts`)
2. **Explain why** this file needs to change and which business requirement it fulfills
3. **Show the change** in git diff format

#### Git diff format rules

- Use standard unified diff format with `---` / `+++` headers and `@@` hunks
- Show enough context lines (at least 3) so the agent can locate the change in the file
- Prefer showing the full function/method when it's small; use hunks for large files
- For new files, show the entire file content as additions

```diff
--- a/src/modules/media/media.service.ts
+++ b/src/modules/media/media.service.ts
@@ -45,6 +45,18 @@ export class MediaService {
   constructor(private readonly s3Service: AwsS3Service) {}
 
+  /**
+   * Enqueue async download job for a media item.
+   * Called by the controller after validating tenant ownership.
+   */
+  async enqueueDownload(mediaId: string, tenantId: string): Promise<JobResult> {
+    const media = await this.mediaRepo.findOneOrFail({ id: mediaId, tenantId });
+    return this.queue.add('download', { mediaId, tenantId });
+  }
+
   async findById(id: string): Promise<Media> {
```

After each diff block, add a brief **Rationale** note (1–3 sentences) explaining why the change was made this way, and how it maps to the business requirement.

#### Ordering

Order file changes from low-level to high-level: entities/schemas → DTOs → services → controllers → modules → frontend (if applicable). This mirrors the natural dependency order.

### Section 3: Impact Analysis

A table summarizing the broader effect on the existing system:

| Affected Area | Nature of Impact | Risk Level | Notes |
|---------------|------------------|------------|-------|
| `ExistingService` | Method signature change | Medium | Callers must update |
| Background jobs | New queue consumer | Low | Additive, no existing logic changed |

Follow with a short paragraph on what to watch out for during QA.

### Section 4: Unresolved Contradictions

Use two subsections:

**Open items** (`⚠️`) — things still needing human confirmation:
> ⚠️ **UNRESOLVED [IP-01]:** {Describe the contradiction or ambiguity}. Needs confirmation before implementation.

**Resolved items** (`✅`) — things that were raised and clarified:
> ✅ **RESOLVED [IP-02]:** {Describe the original question} → **Resolution:** {What was decided and why.}

If there are no items in either category, write `> No contradictions identified.`

### Section 5: References

```
- **Business Plan:** [./business_plan.md](./business_plan.md)
- **Related Plans:** {list any other implementation plans this one depends on or relates to}
```

---

## Highlighting Rules

Use these consistently so readers can scan quickly:

- **`> ⚠️ UNRESOLVED [IP-NN]:`** — open contradiction or ambiguity requiring human decision
- **`> ✅ RESOLVED [IP-NN]:`** — previously raised item, now decided
- **`> 🔴 HIGH IMPACT:`** — a change that is hard to reverse, breaks backward compatibility, or affects multiple modules
- **`> 📌 NOTE:`** — a non-obvious assumption or technical detail the coder should know

Use these sparingly — if everything is highlighted, nothing is.

---

## Step 6: Write the File and Summarize

Write the file to the same directory as the input business plan.

After writing, tell the user:
- Where the file was saved
- How many files will be changed
- Any `⚠️ UNRESOLVED` items still open
- Any `🔴 HIGH IMPACT` changes to review carefully

---

## Reference

For detailed examples of each section, read `assets/template.md` in this skill's directory.
