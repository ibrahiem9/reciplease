# Reciplease Plan Draft v2

This document evolves [plan-draftv1.md](/home/ec2-user/workspace/other/reciplease/docs/roadmap/plan-draftv1.md) by resolving the open assumptions through a structured Q&A. Decisions made during that grilling are recorded in §0 and folded into the relevant sections below.

## 0. Resolved decisions (Q&A log)

| # | Decision | Rationale |
| --- | --- | --- |
| Q1 | **Distribution: private alpha → public launch.** Solo dev + ~5 invited friends in MVP via TestFlight + Play Internal Testing, then public App Store / Play Store within ~3 months of starting. | De-risks extraction quality on real diverse content before taking on public-launch concerns (ToS scrutiny, scaling, billing). Architecture stays public-ready from day one. |
| Q2 | **Source matrix for alpha:** Instagram (public reels/posts), TikTok, YouTube (incl. Shorts), generic web pages with `schema.org/Recipe` JSON-LD, screenshot OCR, manual text. Facebook + Pinterest deferred to post-launch. | Covers the three platforms users actually save from. JSON-LD becomes a fast-path that bypasses the LLM entirely for cooperating recipe sites. FB is mostly private (low fetch success), Pinterest is mostly already-structured (low value-add). |
| Q3 | **Architecture pivot: FaaS + Firestore, no separate backend service.** Drop FastAPI, Postgres, Cloud Tasks, the Drift mutation queue, and the worker pool. The import pipeline runs as a single function; everything else (recipes, collections, shopping lists, sync, offline) is Firestore + Firebase SDKs called directly from the mobile client. | The v1 plan over-built. ~5 alpha users do not need a multi-service backend. Firestore's native offline persistence replaces ~60% of the v1 sync code. Server-side execution is still required for yt-dlp, LLM key protection, and quota enforcement, but those fit in one function. Migration trigger to a real backend is documented in §4.2 below. |
| Q4 | **FaaS platform: GCP Cloud Functions Gen 2** (container image, 60-min timeout, integrates natively with Firebase Auth + Firestore triggers). | Built on Cloud Run; supports the yt-dlp + ffmpeg container layer. Firebase + Firestore + Functions in one Google project means no cross-cloud token plumbing. `gcloud` is already configured on the dev machine. |
| Q5 | **LLM: OpenAI `gpt-4o-mini`** for recipe structuring, called with structured-outputs (`response_format: json_schema`) to enforce the `RecipeDraft` shape. Wrapped behind a thin `LlmStructurer` adapter so it can be swapped later. | Mature structured-outputs feature; cost is ~$0.0005 per typical structuring call (well under the §13.1 ceilings); latency p50 ~1-2s. Claude Haiku 4.5 stays as the documented swap-in if cost or latency degrades. |
| Q6 | **STT: OpenAI `whisper-1` API** with `response_format=verbose_json` for word-level timestamps. Wrapped behind `TranscriptionAdapter`. | Same vendor as the LLM (one fewer API key to rotate), $0.006/min, multilingual, word timestamps help the LLM disambiguate measurements ("two cups" vs "two cubes"). YouTube's official transcript still bypasses this when available, per the §7.4 source matrix. |
| Q7 | **Quotas. Alpha:** no import-count quota; per-user **$5/month soft alert** + **$20/month hard cutoff** + per-import **$0.25 kill-switch**. **Public launch:** **50 imports/user/month** free-tier hard cap (calendar-month reset), same $0.25 per-import kill-switch, no paid tier in MVP. | Alpha cost is bounded by people you know; quota plumbing built but counter is set high enough not to bother testers. Public launch needs a real cap because you don't know who's signing up. Stripe is deferred until >X% of users hit the cap (instrumented in PostHog). |
| Q8 | **App name: Reciplease.** Replaces "PantryClip" working title throughout the spec. Matches repo and bundle ID `com.ibrahiem.reciplease`. | Bundle IDs cannot be renamed once submitted to TestFlight. Locking in the name now avoids a rename pass later. |
| Q9 | **Auth scope. Alpha:** email/password + Google Sign-In only. **M6 (Alpha hardening):** add Apple Sign-In before public submission. **Anonymous accounts:** not used (alpha is invite-only). | Email is zero-config baseline. Google covers most testers. Apple Sign-In is required by Apple if any other 3P SSO is offered, but only enforced on public App Store submission — TestFlight tolerates its absence. Pulling Apple plumbing into M6 keeps M0 small. |
| Q10 | **Parsing: LLM-only structuring + rule-based confidence post-hoc.** Confidence is *not* a self-report from the LLM. It is computed deterministically from validation checks (canonical unit, parseable quantity, food-name sanity, **substring check against the source evidence to catch hallucinations**). See §6.3.1 for the exact rubric. | LLM confidence scores are unreliable. Rule-based confidence is auditable, fixable, and detects hallucinations the LLM would happily report as 0.95 confidence. JSON-LD fast-path skips this entirely (deterministic by construction). |
| Q11 | **Shopping lists cut from alpha.** Ships in M5 (just before public launch), not M4 as v1 had it. Alpha library = recipes + collections + search only. | Shopping lists amplify import-quality bugs, conflating two bug surfaces during the alpha review window. Better to nail import quality first; lists are straightforward to layer on once ingredient normalization is solid. |
| Q12 | **Retention: zero-media, zero-source-text.** No raw audio/video/screenshots persisted past the function invocation. **Captions and transcripts are also discarded** once the user saves the recipe. The saved recipe stores only parsed `ingredients[]`, `steps[]`, `notes`, `sourceUrl`, `creatorName`, `creatorHandle`, `sourcePlatform`. The `sources` subcollection from the v1 data model is removed. Failed/abandoned import jobs are TTL'd after 30 days. Account deletion cascades to all subcollections + GCS objects. | Maximally clean privacy + legal posture. We never store copyrighted media, never store derivative works. "View original" is just a URL link out. Storage bill approaches zero. The import-job doc holds evidence *only* while the user is reviewing, and is deleted as soon as the user finalizes the save. |
| Q13 | **Testing: TDD on the import function** (`functions/tests/` covers fetchers, normalization, confidence rubric, quota checks). **Flutter widget tests** for screens with branching logic (review screen, conflict UI, quota-exceeded state). **Manual E2E** for the social-import paths during alpha. No CI tests against live IG/TikTok/YT pages. | Import function is where regressions are silent — TDD catches normalization drift and rubric changes. Flutter UI is mostly Firestore-bound lists; widget tests focus on the branchy parts. E2E against live social pages is too flaky to run in CI; alpha testers and dev provide that coverage manually. |

## 1. Goal and constraints

Reciplease is a mobile-first recipe saver that converts user-submitted social posts, links, screenshots, and text into structured recipes that can be edited, saved, searched, and reused. The design must optimize for fast time-to-first-value, bounded operating cost, legal caution around third-party content, and an implementation path that a solo developer can ship incrementally.

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

### 3.2 Week 6 Alpha (TestFlight + Play Internal Testing, ~5 invited friends)

Distribution: TestFlight build for iOS, Play Console internal-testing track for Android. No App Store listing yet. Invitees are people the dev knows personally; feedback channel is direct (text/Slack/Discord), not an in-app loop.

Included:

- Email/password, Apple, and Google sign-in through one auth provider.
- Paste-link import, share-sheet import, screenshot import, and manual text import.
- Extraction pipeline with caption, OCR, STT, and source-page fallback.
- Draft review/edit flow with confidence and warnings.
- Saved recipe library with search, collections, and recipe detail.
- ~~Shopping list generation from one or more recipes.~~ **Cut from alpha (Q11). Ships in M5.**
- Multi-device sync with offline read/write support.
- Basic quotas, observability, and admin diagnostics for imports.

Not required by week 6:

- Meal planner.
- Subscription billing.
- Browser extension.
- Semantic/vector search.
- Public sharing and moderation workflows for community content.

### 3.3 Public launch (~month 3)

Gate to flip from alpha to public store listing:

- Median import success rate ≥ 70% across the supported source matrix during alpha.
- Zero P0 data-loss bugs open for 14 consecutive days.
- Cost-per-import p95 staying under the §13.1 ceilings on real alpha traffic.
- Privacy policy, ToS, support email, and takedown form live.
- App Store and Play Store listing assets, age rating, and review-build approval complete.
- Crash-free sessions ≥ 99% on both platforms over the prior 7 days.

What changes at public launch:

- Open registration (no invite required).
- Quotas enforced for the free tier (Q4 will set the exact numbers).
- Sentry + PostHog wired to production project (separate from alpha).
- Public support surface (email + simple help page).

### 3.4 Post-MVP (after public launch)

- Meal planning and calendar views.
- Subscription/paywall (only if free-tier cost trends justify it; see §13).
- Nutrition enrichment.
- Unit conversion and scaling intelligence.
- Browser extension and desktop/web app.
- Semantic search and recommendations.
- Shared/family collections.

## 4. Concrete tech choices

### 4.1 Stack (post Q3+Q4 pivot)

| Area | Choice | Rationale |
| --- | --- | --- |
| Mobile app | Flutter | Already scaffolded; one mobile codebase is the right trade-off for a solo developer. |
| Flutter state management | Riverpod with `flutter_riverpod` + code generation | Clean dependency graph; less ceremony than Bloc. |
| Navigation | `go_router` | Deep-linking and share-sheet entry points are easier with declarative routing. |
| HTTP client | `dio` (only for the one import function call) | Most data ops use Firebase SDKs; `dio` is for the import HTTPS function only. |
| Local storage / offline | **Firestore native offline persistence + Hive for ephemeral UI state** | Firestore SDK caches reads and queues writes automatically across app restarts. Hive holds non-synced UI prefs (last-viewed collection, etc.). Drift is deleted from the plan. |
| Auth | Firebase Authentication | Email/password, Apple, Google. |
| Cloud data store | **Firestore (Native mode)** | Replaces Postgres for alpha. Document shapes in §14.2. Migration trigger in §4.2. |
| Object storage | **Google Cloud Storage** | Replaces S3. Used for raw media, screenshots, thumbnails. Mobile uploads via Firebase Storage SDK with signed-URL semantics. |
| Compute | **GCP Cloud Functions Gen 2 (container image)** | Single function for the import pipeline + one scheduled function for artifact cleanup. No long-running services. |
| Function language | Python 3.12 | Best ecosystem for `yt-dlp`, OCR, STT, recipe parsing. |
| Container base | `python:3.12-slim` + `ffmpeg` + `yt-dlp` pinned | Container image to keep yt-dlp/ffmpeg out of cold-start surprises and let us pin versions. |
| LLM provider | OpenAI `gpt-4o-mini` with structured outputs (`response_format: json_schema`) for `RecipeDraft` enforcement | Mature schema-locked JSON; cheap; fast. Wrapped behind `LlmStructurer` so we can swap to Claude Haiku 4.5 later if cost or latency degrades. |
| STT | OpenAI `whisper-1` API with `verbose_json` (word-level timestamps) | $0.006/min; word timestamps help disambiguate noisy measurements; same vendor as LLM. Wrapped behind `TranscriptionAdapter`. |
| Video/audio download | `yt-dlp` inside the import function container | Only path that handles IG/TikTok/YT consistently. |
| OCR | On-device ML Kit (Flutter) for screenshots, Cloud Vision OCR fallback inside the import function | On-device covers ~95% of English content for free; server-side handles edge cases. |
| Search | Client-side filter + Firestore composite indexes for alpha; revisit at >1k recipes per user | Postgres FTS is the migration target if Firestore becomes insufficient. |
| Vector store | None | Not needed for MVP. Defer until semantic search is a documented user complaint. |
| Secrets | Google Secret Manager | LLM/STT API keys live here; function reads them at cold-start. |
| Analytics | PostHog (client-side from Flutter; later add server-side from the import function for cost metrics) | Solo-dev friendly. |
| Error tracking | Sentry (Flutter SDK + Sentry inside the import function) | One product, both surfaces. |
| CI/CD | GitHub Actions | Flutter tests, function deploys, container build/push to Artifact Registry. |
| IaC | Terraform (single small module) for Firebase project, Firestore indexes, GCS buckets, Cloud Function, Secret Manager bindings, IAM | Keeps the project reproducible; one folder, one `terraform apply`. |

### 4.2 When to migrate off Firestore (documented exit criteria)

Stay on Firestore until **any** of these is true. Then port to Postgres + a real API service.

- A single user's recipe count exceeds 5,000 (Firestore queries get expensive).
- Search complaints become a top-3 user issue and client-side filtering is no longer adequate.
- Monthly Firestore + Cloud Functions cost exceeds ~$200 and the import function isn't the dominant line item.
- A multi-user feature ships that needs server-side joins (e.g., shared family collections with permission queries that Firestore security rules can't express cleanly).

Migration playbook (sketch): export Firestore → seed Postgres via a one-shot script using the data model in §14.1; introduce a thin FastAPI in front of it; clients keep using the same `/import` HTTPS function unchanged; cut over by feature-flag.

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

### 5.2 Backend (post Q3 pivot — single Cloud Functions package)

```text
functions/
  pyproject.toml          one Python project; one container image for both functions
  Dockerfile              python:3.12-slim + ffmpeg + yt-dlp + app code
  src/pantryclip/
    main.py               function entrypoints (import_handler, cleanup_handler)
    config.py             env + Secret Manager loader
    domain/
      jobs.py             ImportJob state machine, evidence assembly, confidence
      parsing.py          RecipeDraft validation
      normalization.py    ingredient/unit normalization
    fetchers/
      jsonld.py           schema.org/Recipe extractor (fast-path)
      instagram.py        yt-dlp + caption scrape
      tiktok.py           yt-dlp + caption scrape
      youtube.py          yt-dlp + transcript-first
      generic_web.py      readability-style fallback
    media/
      ocr.py              Cloud Vision OCR adapter
      transcription.py    STT adapter
    llm/
      structurer.py       LlmStructurer adapter (OpenAI default)
    storage/
      gcs.py              signed URLs, artifact lifecycle
      firestore.py        ImportJob + RecipeDraft writes
    quotas.py             atomic Firestore counter checks
    observability.py      Sentry + structured logging
  tests/                  unit tests per module
```

Two function entrypoints from `main.py`:

- `import_handler` — HTTPS Callable, invoked by mobile client with the `ImportCreateRequest` payload. Runs the full pipeline synchronously and writes draft + status updates into Firestore as it goes (mobile listens via `onSnapshot`).
- `cleanup_handler` — Cloud Scheduler-triggered (daily). Deletes GCS objects older than the artifact TTL.

No long-running API service. No worker pool. No queue. No DB migrations.

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

// Per Q12: ImportEvidence is *ephemeral*. It exists in memory during the
// function invocation and lives in `users/{uid}/import_jobs/{jobId}` only
// while the user is reviewing the draft. The moment the user finalizes the
// save, the import-job doc is deleted along with all evidence fields.
// Captions, transcripts, and source-page text are NEVER persisted as part
// of the saved recipe. The recipe holds only `sourceUrl` + creator metadata.
type ImportEvidence = {
  captionText?: string;
  sourceTitle?: string;
  sourceDescription?: string;
  transcriptText?: string;
  sourcePageText?: string;
  thumbnailUrl?: string;
  creatorName?: string;
  creatorHandle?: string;
  platform?: "instagram" | "tiktok" | "youtube" | "web" | "unknown"; // facebook + pinterest reserved for post-launch (Q2)
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

### 6.3.1 Confidence scoring rubric (post Q10)

The LLM emits structured ingredient JSON via OpenAI structured outputs but does **not** emit a confidence field. Confidence is computed deterministically by the import function after the LLM returns.

**Per-ingredient confidence** (sum, then clamp to [0, 1]):

| Check | Weight |
| --- | --- |
| `unit_canonical` is non-null and in the canonical set (§6.3) | +0.30 |
| `quantity_min` parses as a positive number, OR `optional` is true | +0.30 |
| `food_name` is ≥3 chars, has at least one alphabetic char, is not a stopword | +0.20 |
| `original_text` (lowercased, whitespace-normalized) appears as a substring of the source evidence (caption + transcript + page text) | +0.20 |
| Hallucination penalty: above check fails AND `food_name` also doesn't appear anywhere in the source evidence | −0.30 |

**Recipe-level confidence:**

```
recipe_conf = mean(ingredient_conf)
  × (1.0 if title present else 0.7)
  × (0.5 if no steps OR no ingredients else 1.0)
  × (0.7 if step count < 2 else 1.0)
```

**Confidence band thresholds** (drives §7.5 review-screen behavior):

- `high` ≥ 0.85
- `medium` 0.60–0.84
- `low` < 0.60

**Hallucination flag:** any ingredient whose hallucination penalty fired gets a per-row `parseWarnings` entry: `"This ingredient was not found in the source. Verify before saving."` The review UI surfaces these inline.

JSON-LD fast-path skips this rubric entirely; confidence defaults to 1.0 because the data is structured by construction.

### 6.4 Sync contract (post Q3 — Firestore native)

Mobile sync is offline-capable but not collaborative. The only editor is the signed-in user across their own devices. Firestore's mobile SDK provides this for free.

Rules:

- Reads come from the Firestore SDK's local cache immediately and update via `onSnapshot` when the document changes server-side.
- Writes call the Firestore SDK directly. The SDK queues mutations while offline and replays them on reconnect, in order, idempotently.
- Each recipe document has a server-managed `updatedAt` (Firestore `serverTimestamp()`). Cross-device "stale write" defense relies on Firestore transactions for read-modify-write paths (e.g., reordering steps, checking shopping-list items).
- Shopping-list checkbox state uses last-write-wins via Firestore transactions; conflict UI is unnecessary.
- For full recipe edits across devices (rare in the alpha), we use a `revision` field inside a Firestore transaction: the write fails if the local revision doesn't match the server revision; the app shows "updated on another device — view changes?" UI, identical in spirit to the v1 plan but enforced by Firestore transactions instead of a server endpoint.
- Import jobs are written by the import function and surfaced as Firestore documents under `users/{uid}/import_jobs/{id}`. The mobile client only reads these (rules deny client writes); cancel/retry are HTTPS callables on the import function.

## 7. Extraction pipeline architecture

### 7.1 Stage flow (post Q3 — single Cloud Function invocation)

1. Mobile calls `import_handler` (HTTPS Callable). Firebase Auth wraps the call; `firebase_uid` is in the verified token.
2. Function checks quota counter in Firestore (`users/{uid}/quota`). If exceeded → return `quota_exceeded` immediately.
3. Function creates `users/{uid}/import_jobs/{jobId}` doc with `status: "queued"`. Mobile is already listening on this doc via `onSnapshot`.
4. Function progresses status field through the §6.1 state machine, writing to Firestore at each transition. Mobile UI updates live.
5. Function fetches source evidence per the §7.4 source matrix:
   - JSON-LD fast-path → emit `RecipeDraft` directly, skip LLM (status jumps `fetching_source` → `awaiting_user_review`).
   - Otherwise: caption/page text + (transcript via STT if needed) + (source-page resolution if linked).
6. Function calls `LlmStructurer.structure(evidence)` for non-JSON-LD paths.
7. Function runs deterministic ingredient/unit normalization and confidence scoring.
8. Function writes the final `RecipeDraft` into the import_jobs doc and sets `status: "awaiting_user_review"`. Mobile sees the change via the listener.
9. User edits the draft in the review screen. On save, mobile (a) writes the recipe document to `users/{uid}/recipes/{recipeId}` containing **only parsed fields + `sourceUrl` + creator attribution** (no caption, no transcript, no page text — per Q12), then (b) **deletes** `users/{uid}/import_jobs/{jobId}` so the captured evidence is gone.

The function ends after step 8. There is no long-running worker — the function ran for ~10-60 seconds (cold start + fetch + STT + LLM) and is gone.

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

MVP limits to protect cost and abuse (post Q7):

- Maximum media duration for STT: 5 minutes
- Maximum uploaded screenshot count per import: 5
- Maximum image size per screenshot: 8 MB
- Maximum raw artifact retention: 24 hours (cleanup function runs daily)
- Maximum concurrent imports per user: 3
- **Per-import dollar kill-switch:** $0.25 (function aborts and writes `failed/quota_exceeded` if estimated cost crosses this mid-flight)
- **Alpha:** no monthly import-count cap; per-user $5 soft alert + $20 hard cutoff per calendar month
- **Public launch:** 50 imports/user/calendar-month, enforced by atomic counter at `users/{uid}/quota/current`

### 7.4 Source support matrix (alpha, per Q2)

| Source | Fetch path | Fast-path | LLM structuring | Notes |
| --- | --- | --- | --- | --- |
| Web page with `schema.org/Recipe` JSON-LD | HTTP fetch + HTML parse + JSON-LD extract | **Yes** — emit `RecipeDraft` directly from JSON-LD; skip LLM | Skipped on JSON-LD hit; used as fallback if JSON-LD is malformed | Cheapest path. Sites: NYT Cooking, Serious Eats, AllRecipes, Bon Appétit, etc. |
| Instagram (public reel/post) | `yt-dlp` for media, scrape caption + creator handle from public oEmbed/page | No | Yes, on caption + transcript | Often the highest-value path; captions are short so STT is usually required. |
| TikTok (public video) | `yt-dlp` for media + caption | No | Yes, on caption + transcript | Captions are typically rich; STT still useful for measurements. |
| YouTube (incl. Shorts) | `yt-dlp` + YouTube oEmbed for title/description; prefer official transcript when available, else STT | When official transcript exists, skip STT | Yes, on transcript + description | Transcripts are often free; biggest cost saver of the three video paths. |
| Screenshot upload | Mobile uploads → S3 → on-device OCR first, server-side OCR fallback | No | Yes | Up to 5 images per import (§7.3). |
| Manual text paste | Direct API submit | No | Yes | Cheapest non-fast-path. |

Out of scope for alpha: Facebook, Pinterest, handwritten/printed cookbook pages, audio-only podcasts. Adapter interfaces are stable enough to add them later without schema changes.

### 7.5 Confidence and review gating

Confidence is not a vanity score; it drives UX.

Bands:

- `high` >= 0.85: review screen can default to collapsed warnings
- `medium` 0.60 to 0.84: review screen highlights fields needing inspection
- `low` < 0.60: save action is still allowed, but UI defaults to expanded warnings and missing-field prompts

The app should never auto-publish or silently finalize recipes from low-confidence drafts.

## 8. Auth model

### 8.1 Provider choice (post Q9)

Use Firebase Authentication. Sign-in methods enabled by milestone:

| Milestone | Methods enabled |
| --- | --- |
| M0 → M5 (alpha) | Email/password, Google Sign-In |
| M6 (alpha hardening) | + Apple Sign-In (required for public App Store review) |

Anonymous trial mode is deferred indefinitely because it complicates sync ownership, quota enforcement, and conversion flows; the alpha is invite-only so anonymous onboarding has no benefit anyway.

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

## 9. Offline and sync semantics (post Q3 — Firestore-backed)

### 9.1 Supported offline behaviors

- Browse previously synced recipes, collections, shopping lists offline (Firestore SDK serves from disk cache).
- Edit an existing recipe offline (Firestore SDK queues writes; replays on reconnect).
- Create manual text recipes offline.
- Queue screenshot/text imports if assets are available locally — though the *processing* of those imports requires the function to be reachable, so the import will move from `queued` to actively running only after reconnect.

### 9.2 Not supported offline in alpha

- URL/social-platform import processing while disconnected (function is online-only).
- Share-sheet import finalization without any network access.
- Real-time collaborative editing (out of scope; product is single-user across own devices).

### 9.3 Conflict policy

Use Firestore transactions plus a recipe-level `revision` integer:

- For multi-field edits to a recipe (title, ingredients, steps), the mobile client reads → mutates → writes inside a Firestore transaction that asserts the server `revision` equals the local `revision`. Mismatch aborts the write and the app shows the "updated on another device" UI.
- Shopping-list item check/uncheck uses last-write-wins via plain Firestore writes (no transaction).
- Import job documents are server-written only; clients cannot author state transitions. Cancel/retry are HTTPS callables, not Firestore writes.

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

### 13.1 Cost objectives (post Q5/Q6/Q7 with concrete pricing)

Target blended direct processing cost per completed import:

- median <= $0.01 (most imports are caption + LLM, ~$0.005)
- p95 <= $0.05 (5-min video + STT + LLM, ~$0.030)
- hard per-import kill-switch threshold at $0.25 (function aborts mid-flight if estimate exceeds)
- per-user monthly $-cap: alpha $20, public free tier effectively bounded by 50-import count cap

Per-user expected monthly cost at light usage (~10 imports): ~$0.05.
Per-user expected monthly cost at heavy alpha usage (~100 imports): ~$0.50-3.00 depending on video share.
5-user alpha total expected monthly cost: ~$5-15.

Excludes fixed hosting cost (Cloud Functions baseline + Firestore reads + GCS storage — together <$5/month for alpha) and app-store developer-account fees.

### 13.2 Expected cost profile by path

| Import path | Main cost drivers | Expected relative cost |
| --- | --- | --- |
| Web page with JSON-LD recipe | Fetch + JSON-LD parse only | **Effectively zero** (no LLM) |
| Text/manual | LLM structuring only | Lowest |
| URL with usable caption/page (no JSON-LD) | Fetch + LLM structuring | Low |
| YouTube with official transcript | Fetch + LLM structuring (no STT) | Low |
| Screenshot OCR | OCR + LLM structuring | Medium |
| Video with STT (IG / TikTok / YT no-transcript) | Download + STT + LLM structuring | Highest |

### 13.3 Cost controls

- Do not run STT if caption/page evidence is already sufficient.
- Limit media duration.
- Enforce per-user quotas.
- Use one structuring pass by default; second pass only for recoverable validation failures.
- Store cost estimates per import for later policy tuning.

## 14. Data model sketch

Two views below: §14.1 is the **Firestore document layout** we ship in alpha. §14.2 is the SQL DDL preserved as the migration target if/when we trip the §4.2 exit criteria.

### 14.1 Firestore document layout (alpha)

All paths are rooted at `users/{uid}/...`. Security rules deny cross-user reads/writes; only authenticated users matching `{uid}` can access their subtree (with the exception of `import_jobs/{jobId}` which is server-written only).

Per Q12, the `sources/{sourceId}` subcollection from earlier drafts is **removed** — there is no separate provenance record. The recipe doc holds the minimal source attribution fields directly.

```text
users/{uid}                          User profile (display_name, locale, timezone, plan)
users/{uid}/quota/current            { monthlyImports, monthlyImportLimit, periodStart }
users/{uid}/recipes/{recipeId}       Recipe doc:
                                       title, description, servings_text,
                                       prep_minutes, cook_minutes, total_minutes,
                                       cuisine, meal_type, notes, tags[],
                                       ingredients[]: [{ originalText, quantityMin, quantityMax,
                                                         unitCanonical, foodName, preparationNote,
                                                         optional, sectionTitle, confidence }],
                                       steps[]:       [{ position, instruction, durationSeconds, confidence }],
                                       sourceUrl, creatorName, creatorHandle, sourcePlatform,
                                       parseConfidence, confidenceBand,
                                       revision, collectionIds[], isDeleted, createdAt, updatedAt
users/{uid}/collections/{cid}        { name, color, icon, isDefault, recipeIds[] }
users/{uid}/shopping_lists/{lid}     { title, status, createdFromRecipeIds[], items[] }     // M5, not alpha
users/{uid}/import_jobs/{jobId}      Server-only writes. Mirrors §6.1 status enum + ephemeral draft + ephemeral evidence.
                                     DELETED by the mobile client immediately upon `finalize` save.
                                     TTL'd after 30 days for failed/abandoned jobs.
```

Denormalization choices:

- `ingredients[]` and `steps[]` are arrays inside the recipe doc rather than subcollections — avoids N+1 reads on the most common operation (fetch one recipe). Cost: writes are whole-document; acceptable for recipes (small).
- `recipeIds[]` is duplicated in both the collection doc and `recipes/{id}.collectionIds[]` for two-way query support; both are written inside one Firestore transaction.
- `import_jobs` are scoped per user (not a global collection) so security rules are simple.

Required composite indexes (declared in `firestore.indexes.json`):

- `recipes`: `(isDeleted asc, updatedAt desc)` for library list
- `recipes`: `(isDeleted asc, cuisine asc, updatedAt desc)` for cuisine filter
- `recipes`: `(isDeleted asc, mealType asc, updatedAt desc)` for meal-type filter
- `recipes`: `(isDeleted asc, sourcePlatform asc, updatedAt desc)` for "from Instagram" view
- `import_jobs`: `(status asc, createdAt desc)` for active-import view

### 14.2 Migration-target SQL DDL (post Q3 — preserved for the future, not used in alpha)

The schema below is **not deployed in alpha**. It is kept as the explicit migration target if any of the §4.2 exit criteria fires. Recipes, ingredients, steps, collections, and shopping lists need strong ownership and queryability when we outgrow Firestore.

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

## 15. API surface (post Q3 — Firestore + 2 HTTPS callables)

The v1 plan's REST surface is replaced by **direct Firestore SDK calls** for almost all CRUD, plus **two HTTPS Callable functions** for operations that must run server-side.

### 15.1 HTTPS Callable functions

| Name | Purpose |
| --- | --- |
| `import_handler` | Verify Firebase token, check quota, create import job doc, run the full extraction pipeline, write `awaiting_user_review` draft to Firestore. Synchronous (returns when draft is ready or terminal failure). Mobile may also fire-and-forget and rely on the Firestore listener. |
| `cancel_or_retry_import` | Cancels an in-flight job (best-effort; long STT/LLM calls are not interruptible) or creates a new retry job that references the same source. |

`import_handler` request payload:

```json
{
  "inputType": "url",
  "sourceUrl": "https://example.com/post/123",
  "uploadedAssetGcsPaths": [],
  "locale": "en-US",
  "timezone": "America/Chicago"
}
```

`import_handler` response is just `{ "jobId": "..." }`. All status, progress, and draft fields are written to `users/{uid}/import_jobs/{jobId}` and consumed via Firestore `onSnapshot`.

### 15.2 Direct Firestore operations from the client

All paths are subject to security rules that scope reads/writes to `request.auth.uid == {uid}`.

| Operation | Firestore call |
| --- | --- |
| Library list with filters | `where('isDeleted','==',false).orderBy('updatedAt','desc')` (+ composite index per §14.1) |
| Fetch one recipe | doc read on `users/{uid}/recipes/{recipeId}` |
| Create manual recipe | doc write |
| Edit recipe | transaction on the doc that asserts client `revision` matches server `revision` |
| Soft-delete recipe | partial update `{ isDeleted: true }` |
| Collections CRUD | doc reads/writes on `users/{uid}/collections/{cid}` |
| Add/remove recipe ↔ collection | transaction updating both `collections/{cid}.recipeIds[]` and `recipes/{id}.collectionIds[]` |
| Shopping lists CRUD | doc reads/writes on `users/{uid}/shopping_lists/{lid}` |
| Check/uncheck shopping-list item | direct field write — last-write-wins |
| Upload screenshot | Firebase Storage SDK `putFile` to `gs://.../uploads/{uid}/{jobId}/{n}.jpg`; mobile then passes the GCS path into `import_handler` |
| Sync (multi-device) | nothing — Firestore SDK handles offline + replay automatically |

Search for alpha is client-side: page through `users/{uid}/recipes` (paginated by `updatedAt`) and apply `q` filtering in Dart. Acceptable up to ~1k recipes per user; revisit per §4.2.

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

### 17.1 Legal and platform risk (significantly reduced post Q12)

- Social-platform ToS may restrict downloading, scraping, or storing media beyond user-submitted link handling.
- `yt-dlp` is pragmatic but increases legal and operational exposure if used indiscriminately.
- Takedown handling is still relevant for any product that processes third-party content, even if nothing is persisted.

Posture (post Q12):

- Personal organization tool for user-submitted URLs.
- **Zero media retention.** Raw audio/video is fetched, transcribed, and discarded within the function invocation.
- **Zero source-text retention.** Captions, transcripts, and source-page text are dropped the moment the user finalizes the recipe save. Saved recipes contain only parsed ingredients/steps + the source URL.
- The recipe is the user's interpretation of public content; the source URL is preserved for "view original" attribution.
- Takedown email is still set up in M6 even though we hold no copyrighted material — App Stores expect one.

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

- ~~Which source platforms are officially in-scope at launch versus "best effort"?~~ **Resolved (Q2):** see §7.4.
- ~~Is screenshot OCR good enough if performed on-device first, or do we want server-side OCR consistency for all users?~~ **Resolved (Q4 stack):** on-device ML Kit OCR first, Cloud Vision OCR fallback inside the import function. See §4.1.
- Should the first public release include grocery lists, or can they slip if import quality needs more work?
- ~~What exact free-tier quota and cost ceiling are acceptable for launch?~~ **Resolved (Q7):** 50 imports/user/calendar-month free-tier cap; $0.25 per-import kill-switch. See §7.3 and §13.1.

## 18. Milestone delivery plan

Sizes are rough solo-dev estimates:

- `S`: 1 to 2 focused days
- `M`: 3 to 5 focused days
- `L`: 1 to 2 focused weeks

| Milestone | Task | Size | Depends on | Exit criteria |
| --- | --- | --- | --- | --- |
| M0 Foundation | GCP project, Firebase init, Firestore in Native mode, GCS bucket, Secret Manager, Terraform skeleton, GitHub Actions baseline | S | None | `terraform apply` creates a clean dev project from scratch |
| M0 Foundation | Firebase Auth in Flutter (email/password + Google) + Firestore security rules baseline | M | M0 infra | User can sign in and read/write their own subtree only |
| M0 Foundation | Firestore document schema + composite indexes + Riverpod data layer wrappers | S | M0 infra | Recipe/collection docs round-trip through the app |
| M1 Tracer Bullet | `import_handler` Cloud Function skeleton + container build + deploy pipeline | M | M0 infra | Function is callable from the mobile app and writes a stub `import_jobs` doc |
| M1 Tracer Bullet | JSON-LD fast-path fetcher + ingredient/step assembly | S | Function skeleton | A NYT Cooking URL produces a `RecipeDraft` with no LLM call |
| M1 Tracer Bullet | LLM structuring path + draft review UI + save recipe via Firestore | L | Function skeleton | End-to-end Instagram URL → reviewable draft → saved recipe works |
| M2 Core Library | Recipe list/detail/edit screens reading from Firestore (offline-aware) | M | Tracer bullet | Saved recipes are browsable, editable, and survive airplane mode |
| M2 Core Library | Client-side search + filters (collection/cuisine/meal-type) | S | Recipe schema | User can find recipes by title fragment and ingredient name |
| M3 Import Hardening | Screenshot upload via Firebase Storage + on-device OCR + Cloud Vision fallback | M | Uploads, function | Screenshot import yields reviewable draft |
| M3 Import Hardening | Share-sheet entry flow on iOS and Android | M | Tracer bullet | Shared URLs open directly into import flow |
| M3 Import Hardening | STT adapter (Whisper) + media-duration guardrails | M | Function, GCS | Video imports fall back to transcript path |
| M3 Import Hardening | Confidence scoring, warnings, failure taxonomy | M | Structuring path | Review UI surfaces uncertainty consistently |
| M4 Organization | Collections CRUD + recipe membership (Firestore transactions) | S | Recipe CRUD | Recipes can belong to multiple collections |
| M5 Sync and Ops | Multi-device conflict UI (Firestore transaction-based revision check) + manual smoke tests across two devices | M | Core library | Offline edits reconcile after reconnect; conflicting edits show the resolution UI |
| M5 Sync and Ops | Sentry + PostHog + per-import cost logging + Cloud Monitoring alerts on function errors and per-user quota breaches | M | Function | Import health and cost are observable |
| M5 Sync and Ops | **Shopping list generation + item toggles** (post Q11 — ships here, not M4) | M | Ingredient normalization solid in M3 | One or more recipes create a merged checklist |
| M6 Alpha hardening | Source support matrix, takedown flow, internal QA pass, alpha invite onboarding, **Apple Sign-In wired up** | M | All core flows | Product is honest and supportable for ~5 invited testers on TestFlight + Play Internal; iOS auth supports Apple SSO for public-launch readiness |
| M7 Public launch | Privacy policy, ToS, store listing assets, age rating, review-build submission, free-tier quota enforcement, separate prod Sentry/PostHog projects | M | M6, gate criteria in §3.3 met | Public App Store + Play Store listings are live and pass review |

### Dependency shape

```text
GCP project + Firebase + Firestore + Auth (M0)
  -> import_handler Cloud Function skeleton (M1)
    -> JSON-LD fast-path + LLM structuring + draft review UI (tracer bullet, M1)
      -> Library/search (M2)
      -> Screenshot/share/STT hardening (M3)
        -> Collections/shopping lists (M4)
          -> Multi-device conflict UI + observability (M5)
            -> Alpha hardening (M6)
              -> Public launch (M7)
```

## 19. Recommendation (post Q1–Q13)

Build Reciplease as a Flutter client talking directly to **Firebase (Auth + Firestore + Cloud Storage)** for all CRUD, with a single **GCP Cloud Functions Gen 2** container running the import pipeline (`yt-dlp` + OCR + STT + OpenAI `gpt-4o-mini` for structuring, `whisper-1` for transcription). No long-running backend service, no Postgres in alpha, no separate worker pool, no DB migrations. Recipes are stored as **text-only** — captions, transcripts, and raw media are processed in-memory and discarded the moment the user finalizes the save. Confidence is rule-based, not LLM-self-reported, and includes an explicit hallucination check. The architecture is dramatically smaller than v1, ships sooner, and has a documented migration path to a real backend when the §4.2 exit criteria fire.

## 20. Open follow-ups (smaller decisions, can be answered during execution)

These are real questions but not load-bearing — they can be resolved without architectural impact during the relevant milestone:

- **Domain registration** (M6): pick `reciplease.app` or similar; needed for privacy policy / ToS / takedown email.
- **Privacy policy + ToS text** (M6): boilerplate generators are fine for alpha; review with a lawyer pre-public-launch.
- **Alpha tester recruitment**: which 5 friends, when, what feedback channel.
- **Mobile minimum OS versions**: defaults are iOS 13 / Android API 21 from the Flutter scaffold; revisit if dependencies force it up.
- **Internationalization**: alpha is English-only; revisit before public launch if user base is multilingual.
- **Accessibility**: dynamic type + screen reader labels in the review screen (mandatory for App Store review).
- **PostHog event taxonomy**: define before M5 so analytics aren't retroactive.
- **Cost-trigger to add Stripe paid tier**: instrument "% of free-tier users hitting the 50/month cap" in PostHog. When >20% of weekly actives hit it, build the paid tier.
