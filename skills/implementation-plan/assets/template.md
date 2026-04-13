# Implementation Plan: {Feature Title}

## Document Overview

| Field | Value |
|-------|-------|
| **Feature ID** | `feature/{feature_key}` |
| **Git Branch** | `feature/{feature_key}` |
| **Version** | `1.0` |
| **Status** | Draft |
| **Created** | {YYYY-MM-DD} |
| **Author** | {Name / Role} |
| **Related Documents** | [Business Plan](./business_plan.md) |

---

## Overview

{2–4 sentences: what this implementation does, which modules change, what the net effect on the codebase is. Written to be scanned in 30 seconds by a coding agent or reviewer picking up the task.}

**Impact Summary:**
- `src/modules/{module}/` — {brief description of what changes}
- `src/modules/{module}/{file}.service.ts` — {new method added / signature changed / etc.}
- `src/modules/{module}/{file}.controller.ts` — {new endpoint added}
- `src/modules/{module}/{file}.module.ts` — {new provider registered}
- `{other files}` — {description}

---

## Table of Contents

- [1. Routes Summary](#1-routes-summary)
- [2. Detailed Code Changes](#2-detailed-code-changes)
  - [2.1 {Entity / Schema}](#21-entity--schema)
  - [2.2 {DTO}](#22-dto)
  - [2.3 {Service}](#23-service)
  - [2.4 {Controller}](#24-controller)
  - [2.5 {Module}](#25-module)
- [3. Impact Analysis](#3-impact-analysis)
- [4. Unresolved Contradictions](#4-unresolved-contradictions)
- [5. References](#5-references)

---

## 1. Routes Summary

{Only include this section if routes are added or modified. Otherwise replace with: `> No route changes in this implementation.`}

| Method | Path | Handler | Auth Guard | Change Type | Notes |
|--------|------|---------|------------|-------------|-------|
| `POST` | `/api/v1/{resource}` | `{Controller}.{method}` | `{GuardName}` | New | {Brief note} |
| `GET` | `/api/v1/{resource}/:id/status` | `{Controller}.{method}` | `{GuardName}` | New | {Brief note} |
| `PATCH` | `/api/v1/{existing-resource}/:id` | `{Controller}.{method}` | `{GuardName}` | Modified | {What changed} |

---

## 2. Detailed Code Changes

> This is the core section. Every change is specified in git diff format with rationale. A coding agent should be able to follow this section from top to bottom with minimal improvisation.

Changes are ordered from low-level (data model) to high-level (controller, module), following the natural dependency chain.

---

### 2.1 Entity / Schema

#### `src/modules/{module}/entities/{entity}.entity.ts`

**Why:** {Explain which business requirement drives this change, e.g., "FR-02 requires tracking async download status per media item. We add a `downloadStatus` field and `downloadedAt` timestamp to the existing Media entity."}

> 📌 **NOTE:** {Any non-obvious detail about the data model decision — e.g., "Using enum instead of boolean allows future status values without schema migration."}

```diff
--- a/src/modules/{module}/entities/{entity}.entity.ts
+++ b/src/modules/{module}/entities/{entity}.entity.ts
@@ -1,6 +1,8 @@
 import { Entity, Column, PrimaryGeneratedColumn, CreateDateColumn } from 'typeorm';
+import { DownloadStatus } from '../enums/{entity}-download-status.enum';
 
 @Entity('{table_name}')
 export class {Entity} {
   @PrimaryGeneratedColumn('uuid')
   id: string;
@@ -15,4 +17,14 @@ export class {Entity} {
   @Column({ type: 'varchar', length: 255 })
   name: string;
+
+  @Column({ type: 'enum', enum: DownloadStatus, default: DownloadStatus.PENDING })
+  downloadStatus: DownloadStatus;
+
+  @Column({ type: 'timestamp', nullable: true })
+  downloadedAt: Date | null;
 }
```

**Rationale:** {Why this approach — e.g., "Using nullable `downloadedAt` instead of a default timestamp makes it clear when a download has never been attempted. The enum approach matches the existing `ProcessingStatus` pattern used in `Frame` entity."}

---

#### `src/modules/{module}/enums/{entity}-download-status.enum.ts` *(new file)*

**Why:** {Explain — e.g., "Centralising download states as an enum prevents magic strings and makes state transitions explicit. Referenced by the entity, service, and DTO."}

```diff
--- /dev/null
+++ b/src/modules/{module}/enums/{entity}-download-status.enum.ts
@@ -0,0 +1,8 @@
+export enum DownloadStatus {
+  PENDING = 'PENDING',
+  IN_PROGRESS = 'IN_PROGRESS',
+  COMPLETED = 'COMPLETED',
+  FAILED = 'FAILED',
+}
```

**Rationale:** {e.g., "Four states cover the full lifecycle. `PENDING` is the default (set in entity). `FAILED` allows retry logic without ambiguity."}

---

### 2.2 DTO

#### `src/modules/{module}/dto/create-{entity}.dto.ts` *(new file)*

**Why:** {e.g., "FR-03 requires the client to submit X and Y when initiating a download. This DTO validates both fields at the API boundary."}

```diff
--- /dev/null
+++ b/src/modules/{module}/dto/create-{entity}.dto.ts
@@ -0,0 +1,16 @@
+import { IsUUID, IsNotEmpty, IsOptional, IsString } from 'class-validator';
+
+export class Create{Entity}Dto {
+  @IsUUID()
+  @IsNotEmpty()
+  mediaId: string;
+
+  @IsString()
+  @IsOptional()
+  callbackUrl?: string;
+}
```

**Rationale:** {e.g., "The `callbackUrl` is optional per BR-02 — clients may poll status instead of using webhooks. Both patterns are supported."}

---

#### `src/modules/{module}/dto/{entity}-status.dto.ts` *(new file)*

**Why:** {Explain the response shape.}

```diff
--- /dev/null
+++ b/src/modules/{module}/dto/{entity}-status.dto.ts
@@ -0,0 +1,12 @@
+import { DownloadStatus } from '../enums/{entity}-download-status.enum';
+
+export class {Entity}StatusDto {
+  id: string;
+  status: DownloadStatus;
+  downloadedAt: Date | null;
+  downloadUrl: string | null;
+}
```

**Rationale:** {Explain.}

---

### 2.3 Service

#### `src/modules/{module}/{module}.service.ts`

**Why:** {e.g., "Two new methods are needed: `enqueueDownload` (called by the controller to start the async job) and `getDownloadStatus` (called by the status polling endpoint). The existing `findById` method remains unchanged."}

> 🔴 **HIGH IMPACT:** {e.g., "The `updateStatus` method changes the signature from `(id: string, status: string)` to `(id: string, status: DownloadStatus)`. Any existing callers must be updated — see Section 3."}

```diff
--- a/src/modules/{module}/{module}.service.ts
+++ b/src/modules/{module}/{module}.service.ts
@@ -1,8 +1,12 @@
 import { Injectable, NotFoundException } from '@nestjs/common';
 import { InjectRepository } from '@nestjs/typeorm';
 import { Repository } from 'typeorm';
+import { InjectQueue } from '@nestjs/bull';
+import { Queue } from 'bull';
 import { {Entity} } from './entities/{entity}.entity';
+import { DownloadStatus } from './enums/{entity}-download-status.enum';
+import { {Entity}StatusDto } from './dto/{entity}-status.dto';
 
 @Injectable()
 export class {Module}Service {
   constructor(
     @InjectRepository({Entity})
     private readonly repo: Repository<{Entity}>,
+    @InjectQueue('{queue-name}')
+    private readonly downloadQueue: Queue,
   ) {}
 
   async findById(id: string): Promise<{Entity}> {
@@ -18,4 +22,28 @@ export class {Module}Service {
     return entity;
   }
+
+  /**
+   * Enqueue an async download job.
+   * FR-03: The system must accept download requests and process them asynchronously.
+   */
+  async enqueueDownload(mediaId: string, tenantId: string): Promise<{ jobId: string }> {
+    const entity = await this.repo.findOne({ where: { id: mediaId, tenantId } });
+    if (!entity) throw new NotFoundException(`Media ${mediaId} not found`);
+
+    await this.repo.update(mediaId, { downloadStatus: DownloadStatus.IN_PROGRESS });
+    const job = await this.downloadQueue.add('process-download', { mediaId, tenantId });
+    return { jobId: String(job.id) };
+  }
+
+  /**
+   * Return current download status for a media item.
+   * FR-04: Clients can poll this endpoint to check job progress.
+   */
+  async getDownloadStatus(mediaId: string, tenantId: string): Promise<{Entity}StatusDto> {
+    const entity = await this.repo.findOne({ where: { id: mediaId, tenantId } });
+    if (!entity) throw new NotFoundException(`Media ${mediaId} not found`);
+
+    return {
+      id: entity.id,
+      status: entity.downloadStatus,
+      downloadedAt: entity.downloadedAt,
+      downloadUrl: entity.downloadStatus === DownloadStatus.COMPLETED ? entity.downloadUrl : null,
+    };
+  }
 }
```

**Rationale:** {e.g., "We update the status to `IN_PROGRESS` synchronously before enqueuing so that a rapid status poll immediately after enqueue reflects the correct state. The job processor (see processor file below) is responsible for updating to `COMPLETED` or `FAILED`."}

---

### 2.4 Controller

#### `src/modules/{module}/{module}.controller.ts`

**Why:** {e.g., "Two new endpoints are needed per Section 1 Routes Summary. Both are guarded by the existing tenant auth guard."}

```diff
--- a/src/modules/{module}/{module}.controller.ts
+++ b/src/modules/{module}/{module}.controller.ts
@@ -1,10 +1,14 @@
-import { Controller, Get, Param, UseGuards } from '@nestjs/common';
+import { Controller, Get, Post, Param, Body, UseGuards } from '@nestjs/common';
 import { {Module}Service } from './{module}.service';
+import { Create{Entity}Dto } from './dto/create-{entity}.dto';
 import { TenantGuard } from '../auth/guards/tenant.guard';
+import { CurrentTenant } from '../auth/decorators/current-tenant.decorator';
 
 @Controller('{resource}')
 @UseGuards(TenantGuard)
 export class {Module}Controller {
   constructor(private readonly service: {Module}Service) {}
 
+  @Post('download')
+  async enqueueDownload(
+    @Body() dto: Create{Entity}Dto,
+    @CurrentTenant() tenantId: string,
+  ) {
+    return this.service.enqueueDownload(dto.mediaId, tenantId);
+  }
+
+  @Get(':id/download-status')
+  async getDownloadStatus(
+    @Param('id') id: string,
+    @CurrentTenant() tenantId: string,
+  ) {
+    return this.service.getDownloadStatus(id, tenantId);
+  }
+
   @Get(':id')
   async findOne(@Param('id') id: string) {
     return this.service.findById(id);
```

**Rationale:** {e.g., "Using `@CurrentTenant()` decorator (already exists in the codebase) instead of extracting tenant from headers manually keeps the pattern consistent with all other controllers. The `TenantGuard` at class level applies to all endpoints including the two new ones."}

---

### 2.5 Module

#### `src/modules/{module}/{module}.module.ts`

**Why:** {e.g., "The new `Queue` dependency must be registered with BullModule. Without this, the `@InjectQueue` in the service will fail at startup."}

```diff
--- a/src/modules/{module}/{module}.module.ts
+++ b/src/modules/{module}/{module}.module.ts
@@ -1,10 +1,14 @@
 import { Module } from '@nestjs/common';
 import { TypeOrmModule } from '@nestjs/typeorm';
+import { BullModule } from '@nestjs/bull';
 import { {Entity} } from './entities/{entity}.entity';
 import { {Module}Service } from './{module}.service';
 import { {Module}Controller } from './{module}.controller';
+import { {Entity}Processor } from './{entity}.processor';
 
 @Module({
-  imports: [TypeOrmModule.forFeature([{Entity}])],
+  imports: [
+    TypeOrmModule.forFeature([{Entity}]),
+    BullModule.registerQueue({ name: '{queue-name}' }),
+  ],
-  providers: [{Module}Service],
+  providers: [{Module}Service, {Entity}Processor],
   controllers: [{Module}Controller],
   exports: [{Module}Service],
 })
 export class {Module}Module {}
```

**Rationale:** {e.g., "The `{Entity}Processor` handles the actual download work and must be registered as a provider so NestJS can instantiate it and wire the queue listeners."}

---

## 3. Impact Analysis

| Affected Area | Nature of Impact | Risk Level | Notes |
|---------------|------------------|------------|-------|
| `{ExistingService}` | {e.g., Method signature changed} | Medium | {e.g., Update all callers} |
| `{ExistingModule}` | {e.g., New dependency added} | Low | {e.g., Additive — no existing code removed} |
| `{ExistingEntity}` | {e.g., New columns added via migration} | Medium | {e.g., Run migration before deploying} |
| Background job queue | {e.g., New queue consumer registered} | Low | {e.g., Requires Redis to be running} |

**Watch out for:**
- {e.g., The entity migration must be run before the new endpoints are live — a missing column causes a startup crash.}
- {e.g., Any existing test that mocks `{Module}Service` may need to add the new queue provider to the test module.}
- {e.g., If `{ExistingService}` is used in other modules, check those callers for the signature change.}

---

## 4. Unresolved Contradictions

> ⚠️ **UNRESOLVED [IP-01]:** {Describe the contradiction} — e.g., "The business plan states that download requests should be rate-limited to 3 per minute per tenant, but the existing rate-limiting middleware only supports global limits. Clarify whether to implement tenant-scoped rate limiting or defer it." Needs confirmation before implementation.

> ✅ **RESOLVED [IP-02]:** {Describe the original question} — e.g., "Should the download URL be pre-signed or permanent?" → **Resolution:** Pre-signed S3 URLs with 1-hour expiry, matching the pattern used in `FrameService.getSignedUrl`. Decided {date}.

{If no contradictions: `> No contradictions identified.`}

---

## 5. References

- **Business Plan:** [./business_plan.md](./business_plan.md)
- **Related Implementation Plans:**
  - {e.g., [feature/1_s3_frame_layout/implementation_plan.md](../1_s3_frame_layout/implementation_plan.md) — reuses `AwsS3Service` pattern}
