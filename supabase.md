# Supabase Setup — AutoSpec Architect

This document describes the **entire Supabase setup** for AutoSpec Architect.  
It includes all **enums, tables, RLS policies, triggers, storage, and helper views** in one place.

---

## 🔑 Overview

- Authentication: handled by **Supabase Auth** (`auth.users`).
- Profiles: `public.profiles` table extends user metadata.
- Uploads: `public.uploads` tracks all PRD document uploads.
- Jobs: `public.jobs` tracks background processing tasks.
- Artifacts: `public.artifacts` stores generated design outputs.
- Billing: `public.billing` tracks user usage/plan.
- Enums: `job_status`, `upload_stage`, `artifact_type`.
- Storage: `prd_files` bucket for raw document files.
- Security: Row-Level Security (RLS) on all user-linked tables.
- Hooks: trigger to auto-create a profile when a new auth user is registered.

---

## 📦 Extensions

We enable two PostgreSQL extensions:
- `pgcrypto` → provides `gen_random_uuid()`.
- `pg_trgm` → optional, for advanced text search.

---

## 🏷 Enums

### `job_status`
Represents the lifecycle of a processing job.
- `queued`, `processing`, `completed`, `failed`

### `upload_stage`
Represents where a PRD file is in the pipeline.
- `received`, `extracting`, `parsed`, `refined`, `generating`, `completed`, `failed`

### `artifact_type`
Represents the type of artifact generated.
- `parser_output`, `refined_output`
- `hld_narrative`, `hld_components`, `hld_techstack`, `hld_constraints`, `hld_assumptions`
- `dataflow_flowchart`, `sequence_diagram`
- `lld_structure`, `lld_api`
- `db_schema`, `sql_ddl`, `sql_dml`

---

## 👤 Profiles Table

Tracks each authenticated user.

| Column     | Type      | Notes |
|------------|-----------|-------|
| id         | uuid PK   | references `auth.users(id)` |
| email      | text      | user’s email |
| full_name  | text      | optional |
| plan       | text      | `free` by default |
| created_at | timestamptz | auto timestamp |
| updated_at | timestamptz | auto timestamp |

🔒 **RLS:** Users can only `SELECT` and `UPDATE` their own profile.

---

## 📂 Uploads Table

Tracks PRD documents uploaded by users.

| Column      | Type        | Notes |
|-------------|-------------|-------|
| id          | uuid PK     | unique ID |
| user_id     | uuid FK     | references `auth.users` |
| filename    | text        | original file name |
| content_hash| text        | hash for deduplication |
| mime        | text        | MIME type |
| size_bytes  | bigint      | must be ≥ 0 |
| stage       | upload_stage | pipeline stage |
| error_msg   | text        | error if any |
| created_at  | timestamptz | upload time |
| updated_at  | timestamptz | auto-updated |

🔒 **RLS:** Users can only access their own uploads.  
📌 **Indexes:** unique on `(user_id, content_hash)` for deduplication.

---

## ⚙ Jobs Table

Tracks background processing tasks for uploaded documents.

| Column      | Type        | Notes |
|-------------|-------------|-------|
| id          | uuid PK     | unique ID |
| upload_id   | uuid FK     | references `uploads` |
| user_id     | uuid FK     | references `auth.users` |
| status      | job_status  | current job state |
| error_msg   | text        | error details |
| created_at  | timestamptz | auto timestamp |
| updated_at  | timestamptz | auto timestamp |

🔒 **RLS:** Users can only see their own jobs.

---

## 📑 Artifacts Table

Stores design artifacts generated from uploaded PRDs.

| Column      | Type        | Notes |
|-------------|-------------|-------|
| id          | uuid PK     | unique ID |
| upload_id   | uuid FK     | references `uploads` |
| user_id     | uuid FK     | references `auth.users` |
| type        | artifact_type | kind of artifact |
| content     | jsonb       | structured JSON output |
| created_at  | timestamptz | auto timestamp |

🔒 **RLS:** Users can only access their own artifacts.

---

## 💳 Billing Table

Tracks user usage and plan enforcement.

| Column        | Type        | Notes |
|---------------|-------------|-------|
| id            | uuid PK     | unique ID |
| user_id       | uuid FK     | references `auth.users` |
| plan          | text        | free / paid |
| usage_count   | int         | number of PRDs processed |
| period_start  | timestamptz | billing cycle start |
| period_end    | timestamptz | billing cycle end |
| created_at    | timestamptz | auto timestamp |
| updated_at    | timestamptz | auto timestamp |

🔒 **RLS:** Users can only see their own billing data.

---

## 📦 Storage (Supabase Storage)

- Bucket: `prd_files`  
- Holds raw uploaded PRD files.  
- Access policy:  
  - Users can only read/write files inside their own folder (prefix = user_id).  
  - Example path: `prd_files/{user_id}/{file_id}.pdf`

---

## 🔒 Row-Level Security (RLS)

Enabled for:
- `profiles`
- `uploads`
- `jobs`
- `artifacts`
- `billing`

Each table enforces **per-user access**:  
- Users can only access rows where `user_id = auth.uid()`.

---

## 🔔 Auth Hook

A trigger on `auth.users` automatically inserts into `profiles` whenever a new user signs up.

---

## 📊 Dashboard View

We created a helper view `public.user_dashboard` that joins:
- profiles
- uploads
- jobs
- artifacts

This view makes it easy for the backend/frontend to query all user-related data in one place.

---

## ✅ Summary

Supabase now provides:
1. **Auth** → user login + profiles auto-created.
2. **Tables** → uploads, jobs, artifacts, billing.
3. **Enums** → for structured job states, pipeline stages, and artifact types.
4. **Storage** → raw PRD file storage in `prd_files`.
5. **RLS** → per-user isolation across all resources.
6. **Dashboard view** → easy querying of user data.
7. **Hooks** → profile auto-creation on signup.

This ensures a **multi-tenant SaaS backend** with secure per-user isolation and a clean schema.

---
