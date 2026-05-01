# PantryClip Plan Draft v1

This document refines [initial-plan.md](/home/ec2-user/workspace/other/reciplease/docs/roadmap/initial-plan.md) into an execution-ready v1 spec. The original remains the source of truth for product vision and goals. This draft sharpens that vision into concrete contracts, architecture, schemas, milestones, and acceptance criteria for a solo developer building a Flutter-first product.

## 1. Goal and constraints

PantryClip is a mobile-first recipe saver that converts user-submitted social posts, links, screenshots, and text into structured recipes that can be edited, saved, searched, and reused. The design must optimize for fast time-to-first-value, bounded operating cost, legal caution around third-party content, and an implementation path that a solo developer can ship incrementally.

Constraints:

- Flutter remains the mobile stack.
- The app is private-library-first, not a social network in MVP.
- Imports must preserve provenance and uncertainty instead of pretending extraction is perfect.
- The backend must tolerate partial failures in fetch, OCR, transcription, and LLM structuring without corrupting saved recipes.
- iOS release work still requires macOS/Xcode even if most development happens elsewhere.

## 2. Product stance

### 2.1 What stays true from the original plan

- Fast recipe capture from social links and messy content.
- Structured recipe editing before save.
- Personal library, collections, search, and shopping lists.
- Multi-device sync after sign-in.

### 2.2 MVP product principles

- Preserve source context: never discard original URL, caption text, or user edits.
- Draft first, then save: imports create reviewable drafts, not silent final recipes.
- Deterministic before generative: use fetch/OCR/STT/rules first, then LLM structuring.
- Bounded cost: each import has explicit duration, byte-size, and model-budget caps.
- Legal caution by design: only process user-submitted content; do not build a crawler.

## 3. MVP cut line

### 3.1 Week 1 tracer-bullet slice

Goal: prove the end-to-end loop on one public-link path.

Included:

- Flutter app with email sign-in and authenticated API access.
- Paste public URL into app.
- Backend creates `ImportJob`.
- Fetch caption/title/thumbnail metadata from supported public pages.
- Run one LLM structuring pass on fetched text only.
- Show review screen with parsed title, ingredients, steps, and warnings.
- Save edited recipe to personal library on one device.

Explicitly not in week 1:

- Share sheet integration.
- Screenshot OCR.
- Audio transcription.
- Collections.
- Shopping lists.
- Cross-device conflict handling beyond basic pull-after-save.

Exit criteria:

- A user can go from URL paste to saved recipe in under 60 seconds for at least one supported public source path.
- Failed imports surface a reason and preserve the raw source URL for retry.

### 3.2 Week 6 MVP

Included:

- Email/password, Apple, and Google sign-in through one auth provider.
- Paste-link import, share-sheet import, screenshot import, and manual text import.
- Extraction pipeline with caption, OCR, STT, and source-page fallback.
- Draft review/edit flow with confidence and warnings.
- Saved recipe library with search, collections, and recipe detail.
- Shopping list generation from one or more recipes.
- Multi-device sync with offline read/write support.
- Basic quotas, observability, and admin diagnostics for imports.

Not required by week 6:

- Meal planner.
- Subscription billing.
- Browser extension.
- Semantic/vector search.
- Public sharing and moderation workflows for community content.

### 3.3 Post-MVP

- Meal planning and calendar views.
- Subscription/paywall.
- Nutrition enrichment.
- Unit conversion and scaling intelligence.
- Browser extension and desktop/web app.
- Semantic search and recommendations.
- Shared/family collections.

## 4. Concrete tech choices

| Area | Choice | Rationale |
| --- | --- | --- |
| Mobile app | Flutter | Already scaffolded; one mobile codebase is the right trade-off for a solo developer. |
| Flutter state management | Riverpod with `flutter_riverpod` and code generation | Cleaner dependency graph and testability than ad hoc `setState`; less ceremony than Bloc for this app. |
| Navigation | `go_router` | Deep-linking and share-sheet entry points are easier to model with declarative routing. |
| HTTP client | `dio` | Better interceptors, retries, cancellation, and upload progress than bare `http`. |
| Local storage | Drift over SQLite | Strong typed schema, migrations, relational queries, offline cache, and mutation queue support. Prefer this over Hive because the app has real relational data. |
| Backend language/framework | Python 3.12 + FastAPI + Pydantic | One language across API and import workers; Python is a better fit than Node for OCR/STT/media tooling and parsing libraries. |
| ORM/migrations | SQLAlchemy 2 + Alembic | Mature Postgres support and explicit migrations. |
| Database | PostgreSQL 16 | Primary transactional store, relational integrity, JSONB where needed, and built-in full-text search for MVP. |
| Object storage | Amazon S3 | Durable storage for raw import artifacts, thumbnails, screenshots, and transient audio files. |
| Queue | Amazon SQS with dead-letter queues | Simple, durable async job orchestration without introducing Redis as another critical dependency. |
| Worker runtime | Python worker service polling SQS | Easier to debug than serverless chains for long-running media jobs. |
| LLM provider/router | LiteLLM gateway with a single primary provider in MVP | Keeps provider integration behind one interface. Primary use is schema-constrained recipe structuring; fallback providers can be added later without changing business logic. |
| STT | Managed STT provider for MVP, wrapped behind `TranscriptionAdapter` | Faster path than self-hosting Whisper. Pick the cheapest provider that supports predictable JSON timestamps and language metadata. |
| Video/audio download | `yt-dlp` for user-submitted public URLs where allowed; platform fetchers fall back to metadata-only mode if download fails | Lowest-effort way to support public media retrieval, but must sit behind provider guards and legal controls. |
| OCR | On-device OCR where available for screenshots, backend OCR fallback for uploaded images | Reduces cost and latency; backend still handles cross-platform consistency and recovery. |
| Search | Postgres full-text search + trigram indices | Good enough for personal-library search; no standalone search cluster in MVP. |
| Vector store | None in MVP; if needed later, use `pgvector` in the same Postgres cluster | Avoids premature infrastructure. Semantic retrieval is not required to ship core value. |
| Auth | Firebase Authentication | Best Flutter ergonomics for email/password, Apple, and Google sign-in; backend only needs token verification and user bootstrap. |
| Hosting | AWS: ECS Fargate for API/workers, RDS Postgres, S3, SQS, CloudWatch | Cohesive stack for async media processing and signed object workflows. |
| Analytics | PostHog | Good product analytics for a solo dev without heavy instrumentation overhead. |
| Error tracking | Sentry | Mobile and backend visibility from one product. |
| CI/CD | GitHub Actions | Run Flutter tests, backend tests, migrations, Docker builds, and deploys from one place. |

## 5. Module layout

### 5.1 Mobile

```text
lib/
  core/                 app config, logging, environment, shared types
  presentation/         screens, widgets, route shells
  application/
    auth/               session state and auth flows
    imports/            import commands, draft polling, review orchestration
    recipes/            library, detail, edit, search
    collections/        collection CRUD and membership actions
    shopping_lists/     generation and checklist mutations
    sync/               sync coordinator and connectivity-triggered flush
  data/
    local/              Drift database, DAOs, mutation queue
    remote/             REST clients, DTOs, token injection
    repositories/       app-facing repositories composing local/remote sources
```

Ownership notes:

- `presentation/` renders state only and should not know transport details.
- `application/` owns use cases and state transitions.
- `data/local/` is the source of truth while offline.
- `data/remote/` mirrors backend contracts exactly.

### 5.2 Backend

```text
backend/
  api/
    auth/               Firebase token verification, user bootstrap
    imports/            create/poll/finalize import jobs
    recipes/            CRUD, search, metadata updates
    collections/        collection CRUD and membership
    shopping_lists/     generate lists and mutate items
    sync/               delta feeds and mutation acknowledgements
  domain/
    imports/            job state machine, orchestration policies
    parsing/            recipe draft assembly, validation, confidence
    normalization/      ingredient/unit normalization
    recipes/            recipe aggregate logic
  infrastructure/
    db/                 SQLAlchemy models, repositories
    storage/            S3 artifact access
    queue/              SQS publish/consume and DLQ handling
    fetchers/           platform/page adapters
    ocr/                OCR adapter
    transcription/      STT adapter
    llm/                structuring provider adapter
    observability/      metrics, tracing, structured logs
  workers/
    import_worker/      executes import stages
    cleanup_worker/     TTL deletion for raw artifacts
```

## 6. Core contracts

### 6.1 Import job state machine

`ImportJob` is the backbone of the ingestion flow. A job can only move forward or to a terminal failure state; users may retry by creating a new job that references the same source.

Valid statuses:

```text
queued
fetching_source
extracting_text
transcribing_audio
resolving_source_page
structuring_recipe
normalizing_recipe
awaiting_user_review
completed
failed
canceled
```

Rules:

- Jobs become `awaiting_user_review` only if a draft object exists and passed minimum schema validation.
- A recipe record is created only when the user explicitly saves or when a manual-draft path creates a saved draft locally and syncs later.
- `failed` jobs must include `failure_code`, `failure_stage`, and `user_message`.
- Raw artifacts are retained for a bounded TTL, default 24 hours, unless needed longer for an unresolved job.

### 6.2 Import request and draft contract

```ts
type ImportInputType = "url" | "share" | "screenshot" | "text" | "manual";

type ImportCreateRequest = {
  inputType: ImportInputType;
  sourceUrl?: string;
  sharedText?: string;
  uploadedAssetIds?: string[];
  locale?: string;
  timezone?: string;
};

type ImportEvidence = {
  captionText?: string;
  sourceTitle?: string;
  sourceDescription?: string;
  transcriptText?: string;
  sourcePageText?: string;
  thumbnailUrl?: string;
  creatorName?: string;
  creatorHandle?: string;
  platform?: "instagram" | "tiktok" | "youtube" | "facebook" | "pinterest" | "web" | "unknown";
  warnings: string[];
};

type NormalizedIngredientDraft = {
  originalText: string;
  quantityText?: string;
  quantityMin?: string;
  quantityMax?: string;
  unitRaw?: string;
  unitCanonical?: string;
  foodName: string;
  preparationNote?: string;
  optional: boolean;
  sectionTitle?: string;
  confidence: number;
};

type RecipeDraft = {
  title?: string;
  description?: string;
  servingsText?: string;
  prepMinutes?: number;
  cookMinutes?: number;
  totalMinutes?: number;
  cuisine?: string;
  mealType?: string;
  ingredients: NormalizedIngredientDraft[];
  steps: { position: number; instruction: string; durationSeconds?: number; confidence: number }[];
  notes: string[];
  tags: string[];
  source: ImportEvidence;
  parseWarnings: string[];
  confidence: number;
  confidenceBand: "high" | "medium" | "low";
  requiresReview: boolean;
};
```

### 6.3 Ingredient normalization contract

Normalization rules:

- Always preserve `originalText`.
- Never coerce an unknown unit into a known unit.
- Quantities use decimal strings in storage, not floats, to avoid rounding drift.
- Ranges like `1-2 tbsp` split into `quantityMin` and `quantityMax`.
- Fractions like `1 1/2` normalize to decimal string `1.5` while preserving original display text.
- `foodName` excludes preparation notes where possible.
- Preparation fragments such as `diced`, `softened`, `room temperature`, `divided`, `for garnish` move into `preparationNote` or `optional`.
- If parser confidence is below threshold, keep the line as free text rather than inventing structure.

Canonical units for MVP:

- Volume: `tsp`, `tbsp`, `cup`, `ml`, `l`
- Weight: `g`, `kg`, `oz`, `lb`
- Count-ish: `clove`, `can`, `package`, `slice`, `whole`

Out of scope for MVP:

- Density-aware conversions across mass and volume.
- Region-specific shopping equivalence like `1 bunch cilantro` to grams.

### 6.4 Sync contract

Mobile sync is offline-capable but not fully collaborative. The only editor is the signed-in user across their own devices.

Rules:

- Reads come from local SQLite immediately.
- Mutations write locally first as pending operations, then sync upstream when authenticated and connected.
- Each recipe has `revision` and `updatedAt`.
- Server accepts mutation only if client revision matches or client sets `force=true`.
- On revision conflict, server returns `409` with current server record. Client shows "updated elsewhere" resolution UI.
- Shopping-list item check/uncheck uses last-write-wins because the stakes are low.
- Import jobs themselves are server-authored; client can cache them but cannot author state transitions except cancel/retry.

## 7. Extraction pipeline architecture

### 7.1 Stage flow

1. Client submits import request and receives `ImportJob`.
2. API validates auth, quota, file size, URL scheme, and domain allow/deny rules.
3. API stores source metadata and enqueues `import.requested`.
4. Worker fetches accessible source evidence:
   - URL metadata and page HTML where applicable
   - shared text payload
   - screenshot text from OCR
   - media metadata and thumbnail
5. Worker decides whether transcript is needed:
   - skip if caption/page text already passes minimum evidence threshold
   - run STT only if text evidence is insufficient and media is retrievable
6. Worker tries source-page resolution:
   - if caption or page references an external recipe URL, fetch that page and extract readable text
7. Worker assembles evidence bundle and calls recipe structuring model.
8. Worker runs deterministic validators and ingredient normalization.
9. Worker computes confidence band and warnings.
10. API exposes draft to client in `awaiting_user_review`.
11. User edits and saves draft as final recipe.

### 7.2 Minimum evidence threshold

Do not run the expensive path when unnecessary.

Evidence is "sufficient for first-pass structuring" if any of the following is true:

- caption/shared text contains at least 3 candidate ingredient lines and 2 candidate step lines
- fetched page contains a machine-readable recipe block
- manual text import exceeds 300 characters and matches recipe heuristics

If the threshold is not met:

- try transcript if media duration is within limit
- try external source-page fetch if a link is found
- otherwise fail early with `insufficient_source_content`

### 7.3 Import limits

MVP limits to protect cost and abuse:

- Maximum media duration for STT: 5 minutes
- Maximum uploaded screenshot count per import: 5
- Maximum image size per screenshot: 8 MB
- Maximum raw artifact retention: 24 hours
- Maximum concurrent imports per user: 3
- Default monthly free-tier import quota: product decision, but pipeline should enforce a configurable per-user counter

### 7.4 Confidence and review gating

Confidence is not a vanity score; it drives UX.

Bands:

- `high` >= 0.85: review screen can default to collapsed warnings
- `medium` 0.60 to 0.84: review screen highlights fields needing inspection
- `low` < 0.60: save action is still allowed, but UI defaults to expanded warnings and missing-field prompts

The app should never auto-publish or silently finalize recipes from low-confidence drafts.

## 8. Auth model

### 8.1 Provider choice

Use Firebase Authentication for:

- email/password
- Apple Sign-In
- Google Sign-In

Anonymous trial mode is deferred until post-MVP because it complicates sync ownership, quota enforcement, and conversion flows.

### 8.2 Backend auth flow

1. Mobile authenticates with Firebase.
2. Mobile sends Firebase ID token as Bearer token to backend.
3. Backend verifies token signature and extracts `firebase_uid`.
4. Backend upserts a local `users` row keyed by `firebase_uid`.
5. All recipe/library data is keyed to the local user ID; Firebase remains the identity provider, not the domain database.

### 8.3 Session requirements

- All import creation and recipe mutation endpoints require auth.
- Uploaded artifacts use short-lived signed upload URLs.
- Backend never stores provider refresh tokens.

## 9. Offline and sync semantics

### 9.1 Supported offline behaviors

- Browse previously synced recipes, collections, and shopping lists offline.
- Edit an existing recipe offline.
- Create manual text recipes offline.
- Queue screenshot/text imports for later upload if assets are available locally.

### 9.2 Not supported offline in MVP

- URL import processing while disconnected.
- Share-sheet import finalization without any network access.
- Multi-device merge beyond explicit conflict handling.

### 9.3 Conflict policy

Use coarse-grained revision control:

- Recipe title/metadata and ingredient/step arrays are versioned at the recipe level.
- On conflict, keep both versions visible to the user; do not silently merge step lists.
- Shopping-list checkbox state uses last-write-wins.

## 10. UI behavior for uncertainty and errors

### 10.1 Import progress states

Client-visible stages:

- Fetching source
- Reading text
- Listening to audio
- Structuring recipe
- Ready for review

Each stage must map to one or more backend job statuses so the UI never fakes progress.

### 10.2 Review screen uncertainty surfacing

Required cues:

- warning banner when fields are incomplete or low confidence
- per-ingredient confidence hints only when confidence is low enough to matter
- "Source evidence" section showing the origin of the draft: caption, transcript, source page, screenshot text
- missing-field prompts for title, ingredients, or steps
- source attribution link/button if a source URL exists

Avoid:

- showing opaque numeric model scores without explanation
- silently dropping contradictory inputs between caption and transcript

### 10.3 Error taxonomy

User-facing error codes:

- `unsupported_source`
- `private_or_unreachable_content`
- `download_failed`
- `ocr_failed`
- `transcription_failed`
- `insufficient_source_content`
- `parse_failed`
- `quota_exceeded`
- `temporary_server_error`

Every terminal error response must include:

- stable machine-readable code
- short user message
- retryability boolean

## 11. Search behavior

MVP search requirements:

- title prefix and full-text match
- ingredient name match on normalized `foodName`
- creator/source match
- collection filter
- cuisine and meal-type filters

Implementation:

- Postgres `tsvector` for recipe text fields
- trigram index for fuzzy title search
- ingredient search via join on normalized ingredient rows

Semantic search is deferred until the library size and user demand justify it.

## 12. Observability and ops

### 12.1 Logs

Use structured JSON logs with these keys at minimum:

- `request_id`
- `user_id`
- `import_job_id`
- `stage`
- `provider`
- `latency_ms`
- `cost_estimate_usd`
- `result`

### 12.2 Metrics

Track at least:

- import jobs created per day
- import success rate by source type
- stage failure rate
- p50/p95 import latency
- mean and p95 LLM tokens per import
- mean and p95 STT minutes per import
- average import cost by source type
- percentage of recipes saved after review
- percentage of drafts abandoned in review

### 12.3 Tracing

Trace one import across API, queue, worker, OCR/STT/LLM calls, and save-finalization. A single `import_job_id` must link all spans and logs.

### 12.4 Alerts

Page-worthy alerts for MVP:

- DLQ depth above threshold
- import failure rate above 20% for 15 minutes
- mean import cost exceeds configured ceiling
- RDS or S3 auth/configuration failures

## 13. Cost model

### 13.1 Cost objectives

Target blended direct processing cost per completed import:

- median <= $0.05
- p95 <= $0.20
- hard kill-switch threshold at $0.25 per import

This excludes fixed hosting cost and app-store fees.

### 13.2 Expected cost profile by path

| Import path | Main cost drivers | Expected relative cost |
| --- | --- | --- |
| Text/manual | LLM structuring only | Lowest |
| URL with usable caption/page | Fetch + LLM structuring | Low |
| Screenshot OCR | OCR + LLM structuring | Medium |
| Video with STT | Download + STT + LLM structuring | Highest |

### 13.3 Cost controls

- Do not run STT if caption/page evidence is already sufficient.
- Limit media duration.
- Enforce per-user quotas.
- Use one structuring pass by default; second pass only for recoverable validation failures.
- Store cost estimates per import for later policy tuning.

## 14. Data model sketch

The schema below is intentionally relational. Recipes, ingredients, steps, collections, and shopping lists need strong ownership and queryability.

```sql
create table users (
  id uuid primary key,
  firebase_uid text not null unique,
  email text unique,
  display_name text,
  locale text not null default 'en',
  timezone text not null default 'UTC',
  subscription_status text not null default 'free',
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);

create table sources (
  id uuid primary key,
  user_id uuid not null references users(id) on delete cascade,
  platform text not null,
  source_url text,
  canonical_url text,
  creator_name text,
  creator_handle text,
  source_title text,
  thumbnail_url text,
  caption_text text,
  transcript_text text,
  source_page_text text,
  attribution_required boolean not null default true,
  created_at timestamptz not null default now()
);

create table import_jobs (
  id uuid primary key,
  user_id uuid not null references users(id) on delete cascade,
  source_id uuid references sources(id) on delete set null,
  input_type text not null,
  status text not null,
  failure_code text,
  failure_stage text,
  retryable boolean not null default false,
  confidence numeric(4,3),
  confidence_band text,
  raw_artifact_prefix text,
  cost_estimate_usd numeric(8,4),
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  completed_at timestamptz
);

create table recipes (
  id uuid primary key,
  user_id uuid not null references users(id) on delete cascade,
  source_id uuid references sources(id) on delete set null,
  import_job_id uuid references import_jobs(id) on delete set null,
  title text not null,
  description text,
  servings_text text,
  prep_minutes integer,
  cook_minutes integer,
  total_minutes integer,
  cuisine text,
  meal_type text,
  notes text,
  parse_confidence numeric(4,3),
  confidence_band text,
  revision integer not null default 1,
  is_deleted boolean not null default false,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);

create table ingredients (
  id uuid primary key,
  recipe_id uuid not null references recipes(id) on delete cascade,
  position integer not null,
  section_title text,
  original_text text not null,
  quantity_text text,
  quantity_min numeric(10,3),
  quantity_max numeric(10,3),
  unit_raw text,
  unit_canonical text,
  food_name text not null,
  preparation_note text,
  optional boolean not null default false,
  confidence numeric(4,3),
  created_at timestamptz not null default now()
);

create table steps (
  id uuid primary key,
  recipe_id uuid not null references recipes(id) on delete cascade,
  position integer not null,
  instruction text not null,
  duration_seconds integer,
  confidence numeric(4,3),
  created_at timestamptz not null default now()
);

create table collections (
  id uuid primary key,
  user_id uuid not null references users(id) on delete cascade,
  name text not null,
  color text,
  icon text,
  is_default boolean not null default false,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  unique (user_id, name)
);

create table recipe_collections (
  recipe_id uuid not null references recipes(id) on delete cascade,
  collection_id uuid not null references collections(id) on delete cascade,
  created_at timestamptz not null default now(),
  primary key (recipe_id, collection_id)
);

create table shopping_lists (
  id uuid primary key,
  user_id uuid not null references users(id) on delete cascade,
  title text not null,
  status text not null default 'active',
  created_from_recipe_ids jsonb not null default '[]'::jsonb,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);

create table shopping_list_items (
  id uuid primary key,
  shopping_list_id uuid not null references shopping_lists(id) on delete cascade,
  position integer not null,
  ingredient_name text not null,
  quantity_text text,
  quantity numeric(10,3),
  unit_canonical text,
  aisle_category text,
  checked boolean not null default false,
  source_recipe_id uuid references recipes(id) on delete set null,
  source_ingredient_id uuid references ingredients(id) on delete set null,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);
```

## 15. API surface

Use REST for MVP. The client is mobile-only and the domain is resource-oriented enough that REST keeps integration simpler than introducing RPC.

All endpoints are prefixed with `/v1` and require Bearer auth unless marked otherwise.

### 15.1 Session bootstrap

| Method | Path | Purpose |
| --- | --- | --- |
| `POST` | `/session/bootstrap` | Verify Firebase token, create/update local user row, return app profile and feature flags |

Response:

```json
{
  "user": {
    "id": "uuid",
    "email": "user@example.com",
    "displayName": "Name",
    "locale": "en",
    "timezone": "UTC"
  },
  "features": {
    "screenshotImport": true,
    "shareImport": true
  }
}
```

### 15.2 Uploads

| Method | Path | Purpose |
| --- | --- | --- |
| `POST` | `/uploads` | Request signed upload URLs for screenshots or other assets |

### 15.3 Imports

| Method | Path | Purpose |
| --- | --- | --- |
| `POST` | `/import-jobs` | Create import job |
| `GET` | `/import-jobs/{id}` | Poll job status and draft |
| `POST` | `/import-jobs/{id}/cancel` | Cancel queued/in-progress job |
| `POST` | `/import-jobs/{id}/retry` | Create a retry job with same source input |
| `POST` | `/import-jobs/{id}/finalize` | Save edited draft as recipe |

`POST /import-jobs` request:

```json
{
  "inputType": "url",
  "sourceUrl": "https://example.com/post/123",
  "locale": "en-US",
  "timezone": "America/Chicago"
}
```

`GET /import-jobs/{id}` response shape:

```json
{
  "id": "uuid",
  "status": "awaiting_user_review",
  "failureCode": null,
  "progressLabel": "Ready for review",
  "draft": {
    "title": "Creamy Lemon Pasta",
    "ingredients": [],
    "steps": [],
    "parseWarnings": ["Transcript and caption disagree on oven temperature."],
    "confidence": 0.74,
    "confidenceBand": "medium",
    "requiresReview": true
  }
}
```

`POST /import-jobs/{id}/finalize` request:

```json
{
  "recipe": {
    "title": "Creamy Lemon Pasta",
    "description": "Edited by user",
    "servingsText": "4 servings",
    "prepMinutes": 10,
    "cookMinutes": 20,
    "ingredients": [],
    "steps": [],
    "notes": [],
    "collectionIds": ["uuid"]
  }
}
```

### 15.4 Recipes

| Method | Path | Purpose |
| --- | --- | --- |
| `GET` | `/recipes` | Paginated library list with filters |
| `GET` | `/recipes/{id}` | Fetch one recipe |
| `POST` | `/recipes` | Create manual recipe |
| `PATCH` | `/recipes/{id}` | Update recipe by revision |
| `DELETE` | `/recipes/{id}` | Soft-delete recipe |

Recommended query params on `GET /recipes`:

- `q`
- `collectionId`
- `cuisine`
- `mealType`
- `sourcePlatform`
- `cursor`
- `limit`

### 15.5 Collections

| Method | Path | Purpose |
| --- | --- | --- |
| `GET` | `/collections` | List collections |
| `POST` | `/collections` | Create collection |
| `PATCH` | `/collections/{id}` | Rename/update collection |
| `DELETE` | `/collections/{id}` | Delete collection |
| `POST` | `/collections/{id}/recipes` | Add recipe to collection |
| `DELETE` | `/collections/{id}/recipes/{recipeId}` | Remove recipe from collection |

### 15.6 Shopping lists

| Method | Path | Purpose |
| --- | --- | --- |
| `GET` | `/shopping-lists` | List shopping lists |
| `POST` | `/shopping-lists` | Create from recipe IDs or manual items |
| `GET` | `/shopping-lists/{id}` | Fetch one shopping list |
| `PATCH` | `/shopping-lists/{id}` | Rename/archive list |
| `PATCH` | `/shopping-lists/{id}/items/{itemId}` | Check/uncheck or edit item |
| `POST` | `/shopping-lists/{id}/items` | Add manual item |

### 15.7 Sync

| Method | Path | Purpose |
| --- | --- | --- |
| `GET` | `/sync/changes` | Fetch deltas since a cursor |
| `POST` | `/sync/mutations` | Submit queued offline mutations in order |

## 16. Acceptance criteria

### 16.1 Import success

- For supported public-link imports, the system returns either a reviewable draft or a terminal error with a retryable code.
- The system never saves a recipe without user review in MVP.
- Every saved recipe retains source URL or a reason why source attribution is unavailable.

### 16.2 Data quality

- Every ingredient row preserves original text.
- Steps are stored in explicit order.
- Contradictory evidence creates warnings, not silent overwrites.

### 16.3 Offline behavior

- Previously synced recipes load without network.
- Offline edits survive app restart and sync after reconnect.
- Conflicts produce a user-visible resolution path rather than silent data loss.

### 16.4 Ops

- Every import job can be traced across API and worker logs.
- Mean cost per import is queryable from stored metrics.
- DLQ and quota breaches are observable within the admin/ops toolchain.

## 17. Risks and open questions

### 17.1 Legal and platform risk

- Social-platform ToS may restrict downloading, scraping, or storing media beyond user-submitted link handling.
- `yt-dlp` is pragmatic but increases legal and operational exposure if used indiscriminately.
- Takedown handling is required even for a private-library product if copyrighted material is stored.

Current recommendation:

- Position the product as a personal organization tool for user-submitted content.
- Prefer metadata/text extraction first.
- Store raw media only temporarily.
- Build a takedown/delete path from day one.

### 17.2 Rate limits and source fragility

- Social pages change markup frequently.
- Anti-bot and rate-limit defenses can reduce fetch success.
- Some platforms or posts will always be private or inaccessible.

Recommendation:

- Treat source adapters as best-effort.
- Make error states honest.
- Maintain a support matrix by source type instead of implying universal coverage.

### 17.3 Hallucinated ingredients and unsafe instructions

- LLM structuring can invent missing ingredients or normalize incorrectly.
- Transcript noise can change temperatures, quantities, or timing.

Recommendation:

- Preserve evidence.
- Require review.
- Flag low-confidence ingredient lines and contradictory temperatures or quantities.

### 17.4 Abuse and cost amplification

- Users can spam expensive video imports.
- Malicious inputs can target long videos, huge uploads, or repeated retries.

Recommendation:

- Per-user quotas.
- Concurrent import caps.
- Media duration and size limits.
- Retry throttles.

### 17.5 Content moderation

- Even private imports may include non-food or disallowed content.
- If public sharing is added later, moderation scope increases sharply.

Recommendation:

- MVP keeps content private by default.
- Log abuse signals but avoid building a full moderation stack before public features exist.

### 17.6 Open questions

- Which source platforms are officially in-scope at launch versus "best effort"?
- Is screenshot OCR good enough if performed on-device first, or do we want server-side OCR consistency for all users?
- Should the first public release include grocery lists, or can they slip if import quality needs more work?
- What exact free-tier quota and cost ceiling are acceptable for launch?

## 18. Milestone delivery plan

Sizes are rough solo-dev estimates:

- `S`: 1 to 2 focused days
- `M`: 3 to 5 focused days
- `L`: 1 to 2 focused weeks

| Milestone | Task | Size | Depends on | Exit criteria |
| --- | --- | --- | --- | --- |
| M0 Foundation | Choose infra accounts, secrets strategy, repo layout, env config | S | None | Local and staging environments can boot |
| M0 Foundation | Firebase auth bootstrap in Flutter + backend token verification | M | None | User can sign in and call authenticated endpoint |
| M0 Foundation | Postgres schema + Alembic baseline + Drift schema scaffold | M | M0 infra | Mobile and backend share stable core entities |
| M1 Tracer Bullet | `POST /import-jobs` + polling + worker queue loop | M | Auth, schema | Import jobs can be created and transition states |
| M1 Tracer Bullet | Caption/page fetcher for one public-link path | M | Queue loop | At least one supported source yields text evidence |
| M1 Tracer Bullet | First-pass LLM structuring + draft review UI + save recipe | L | Fetcher, auth, schema | End-to-end URL import to saved recipe works |
| M2 Core Library | Recipe list/detail/edit screens with local cache | M | Tracer bullet | Saved recipes are browsable and editable |
| M2 Core Library | Search and filters via Postgres FTS + mobile query UI | M | Recipe schema | User can find recipes by title and ingredient |
| M3 Import Hardening | Screenshot upload + OCR path | M | Uploads, storage | Screenshot import yields reviewable draft |
| M3 Import Hardening | Share-sheet entry flow on iOS and Android | M | Tracer bullet | Shared URLs open directly into import flow |
| M3 Import Hardening | STT adapter + media-duration guardrails | M | Storage, fetchers | Video imports can fall back to transcript path |
| M3 Import Hardening | Confidence scoring, warnings, failure taxonomy | M | Structuring path | Review UI surfaces uncertainty consistently |
| M4 Organization | Collections CRUD + recipe membership | S | Recipe CRUD | Recipes can belong to multiple collections |
| M4 Organization | Shopping list generation + item toggles | M | Ingredient normalization | One or more recipes create a merged checklist |
| M5 Sync and Ops | Drift mutation queue + sync endpoints + conflict UI | L | Core library | Offline edits reconcile after reconnect |
| M5 Sync and Ops | Sentry, PostHog, cost metrics, DLQ alerts | M | Core API/worker | Import health and cost are observable |
| M6 Launch Prep | Source support matrix, policy pages, takedown flow, QA | M | All core flows | Product is honest, supportable, and releasable |

### Dependency shape

```text
Foundation
  -> Auth + schema
    -> Import job backbone
      -> Tracer-bullet URL import
        -> Library/search
        -> Screenshot/share/STT hardening
          -> Collections/shopping lists
            -> Offline sync/ops
              -> Launch prep
```

## 19. Recommendation

Build PantryClip as a Flutter client backed by a Python/FastAPI + Postgres + S3 + SQS system, with Drift-based local persistence and a draft-first import pipeline. This is the simplest architecture that still respects the hard parts of the product: async ingestion, uncertain extraction quality, offline library access, and future cost control.
