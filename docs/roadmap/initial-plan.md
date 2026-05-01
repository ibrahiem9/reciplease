# Recipe Saver App Spec
**Working title:** PantryClip  
**Document type:** Product + technical design spec  
**Target stack:** Flutter mobile app + backend services  
**Primary goal:** Let users save recipe videos/posts from social media and turn them into structured, searchable recipes.

## 1. Background

ReciMe publicly positions itself as an app that lets users save recipes from sources including Instagram, TikTok, Facebook, YouTube, Pinterest, screenshots, handwritten notes, and websites, then organize those recipes, plan meals, and generate grocery lists. Its help content also describes a multi-step import approach: first use the post caption when available, then try audio from the video, and if needed attempt to find the original recipe website.  [oai_citation:0‡ReciMe](https://www.recime.app/?utm_source=chatgpt.com)

This spec describes how to build an **original app inspired by that workflow**, not a trademark/content clone.

## 2. Product Vision

A mobile-first app where a user can:
- share a social media link, reel, or post into the app
- have the app extract a structured recipe
- save it into a personal recipe library
- organize recipes into collections
- convert ingredients into shopping lists
- optionally plan meals and export recipes later

## 3. Problem Statement

Recipes on social platforms are scattered, inconsistent, and often trapped in short-form video or caption text. Users want a single place to:
- save recipes quickly
- recover ingredients and steps from messy social content
- search by ingredient, cuisine, creator, or meal type
- avoid losing recipes when posts are buried or deleted

## 4. Product Goals

### Primary goals
- Make saving a recipe take under 30 seconds.
- Convert unstructured social content into a clean recipe record.
- Make saved recipes searchable and reusable.
- Support both Android and iOS.

### Secondary goals
- Enable grocery list generation.
- Enable collections and meal planning.
- Build a subscription-ready product.

### Non-goals for MVP
- full social network inside the app
- creator marketplace
- in-app grocery ordering
- OCR perfection across all handwriting styles
- guaranteed parsing of every private or protected platform post

## 5. Target Users

### Core users
- home cooks who save recipe reels/posts frequently
- busy parents and meal planners
- users who collect recipes but rarely revisit them because organization is poor

### Secondary users
- fitness/macro-focused cooks
- beginner cooks who need simplified step-by-step instructions
- users migrating from Notes, screenshots, bookmarks, or paper recipes

## 6. Key User Stories

1. As a user, I can paste a recipe video URL into the app and get a structured recipe.
2. As a user, I can use the native share sheet from another app to send a link into this app.
3. As a user, I can review and edit extracted ingredients, steps, servings, and metadata before saving.
4. As a user, I can save recipes into folders/collections.
5. As a user, I can search my saved recipes later.
6. As a user, I can generate a grocery list from one or more recipes.
7. As a user, I can import from screenshots or typed text if a social post import fails.
8. As a user, I can access my saved recipes from another device after signing in.

## 7. Competitive Anchor: ReciMe-Inspired Capabilities

The product should be informed by ReciMe’s visible feature set, including:
- importing recipes from multiple social/web sources
- grocery list generation
- meal planning
- web app / extension support
- export/PDF and measurement conversion features as later-stage extensions.  [oai_citation:1‡ReciMe](https://www.recime.app/?utm_source=chatgpt.com)

This app should differentiate through:
- faster import feedback
- better edit/review UX
- stronger search and organization
- cleaner ingredient normalization
- optional Islamic-friendly food preference filters if desired later

## 8. MVP Scope

### Included in MVP
- account creation and login
- recipe import by URL paste
- iOS/Android share-to-app import
- recipe extraction pipeline
- recipe detail editing
- save recipe to user library
- collections/folders
- search and filtering
- grocery list generation from selected recipes
- screenshot OCR import
- plain text/manual import
- sync across devices

### Deferred to Phase 2+
- calendar-style meal planner
- Chrome extension / browser extension
- desktop/web app
- PDF export
- handwritten recipe import tuned beyond baseline OCR
- creator following/sharing
- smart recommendations
- pantry/inventory features

## 9. Functional Requirements

## 9.1 Authentication
- Email/password
- Apple Sign-In
- Google Sign-In
- Anonymous trial mode optional
- User profile stores dietary preferences, default serving units, timezone, locale

## 9.2 Recipe Import
Users can import by:
- pasted URL
- system share sheet from supported apps
- pasted caption/text
- screenshot upload
- manual entry

### Import pipeline behavior
Based on ReciMe’s documented public approach, the import pipeline should follow a fallback chain:
1. extract useful caption/text metadata first
2. if insufficient, run speech-to-text on the video audio
3. if still insufficient, attempt to identify and fetch the original linked/source recipe page
4. if extraction confidence is low, present an assisted edit screen rather than silently saving junk data.  [oai_citation:2‡ReciMe](https://recime.app/help/en/articles/11624978-import-from-facebook?utm_source=chatgpt.com)

### Import output schema
Each parsed recipe should attempt to populate:
- title
- author/creator handle
- source platform
- source URL
- hero image / thumbnail
- prep time
- cook time
- total time
- servings
- ingredients[]
- steps[]
- notes
- tags
- cuisine
- meal type
- nutrition placeholder
- confidence score
- parse warnings

## 9.3 Recipe Detail Screen
- hero image
- title and metadata
- ingredients with quantity normalization
- numbered steps
- notes/tips
- save/edit/delete
- add to grocery list
- add to collection
- mark as cooked
- source attribution

## 9.4 Collections
- create/edit/delete collections
- default collections: Favorites, To Try, Dinner, Desserts
- recipe may belong to multiple collections
- sort by newest, recently cooked, alphabetic

## 9.5 Search
Search by:
- title
- ingredient
- creator/source
- cuisine
- collection
- dietary tag
- prep time
- meal type

## 9.6 Grocery List
- generate from one recipe or multiple recipes
- merge duplicate ingredients
- manual check/uncheck
- aisle/category grouping if available
- unit conversion support in later phase

## 9.7 Admin / Moderation
- abuse reporting for shared/public content if social features are added
- import failure analytics
- parser confidence dashboards
- rate-limits for video/audio processing abuse

## 10. UX Flows

## 10.1 Paste Link Flow
1. User opens app
2. Taps “Import”
3. Pastes recipe link
4. Backend starts extraction
5. UI shows progress states: fetching -> extracting -> structuring
6. User reviews parsed recipe
7. User edits as needed
8. User taps Save
9. Recipe lands in library and optionally a chosen collection

## 10.2 Share Sheet Flow
1. User sees recipe post in another app
2. Taps native Share
3. Chooses PantryClip
4. App opens lightweight import screen
5. Import starts with shared URL/content
6. User reviews and saves

## 10.3 Screenshot Flow
1. User uploads screenshot(s)
2. OCR extracts text
3. Parser detects ingredients/steps
4. User fixes parsing issues
5. User saves recipe

## 10.4 Grocery List Flow
1. User opens saved recipe
2. Taps Add to Grocery List
3. App adds all ingredients
4. User can deselect pantry items
5. Grocery list groups and deduplicates items

## 11. Suggested Mobile Screens

- Onboarding
- Auth
- Home / recipe library
- Import modal
- Import progress
- Recipe review/edit
- Recipe detail
- Collections list
- Collection detail
- Search/results
- Grocery list
- Settings/profile
- Subscription/paywall

## 12. Technical Architecture

## 12.1 Client
**Framework:** Flutter

Rationale:
- single shared codebase for Android and iOS
- fast UI iteration
- strong fit for content-heavy mobile UI
- Linux-friendly day-to-day development, while iOS device build/release still requires Xcode on macOS. Flutter’s current docs state that Xcode is required to build and release iOS apps and that macOS is required for that workflow.  [oai_citation:3‡Flutter Docs](https://docs.flutter.dev/platform-integration/ios/setup?utm_source=chatgpt.com)

### Suggested Flutter architecture
- `presentation/`
- `application/`
- `domain/`
- `data/`
- `core/`

Recommended patterns:
- Riverpod or Bloc for state management
- Dio/HTTP client
- Freezed + json_serializable for models
- GoRouter for navigation
- Hive/SQLite for local cache

## 12.2 Backend
Recommended:
- API layer: FastAPI or NestJS
- DB: PostgreSQL
- Object storage: S3
- Queue: SQS / RabbitMQ
- Workers: Python for parsing / OCR / transcription orchestration
- Search: Postgres FTS initially, OpenSearch later
- Auth: Firebase Auth, Cognito, or custom JWT service
- Analytics: PostHog / Amplitude
- Error tracking: Sentry

## 12.3 Import Processing Pipeline
### Services
- Import API
- Content fetcher
- Metadata extractor
- OCR service
- Speech-to-text adapter
- Recipe parser / normalizer
- Confidence scorer
- Source resolution service
- Notification/webhook service

### Pipeline
1. Accept import request
2. Validate URL and user quota
3. Fetch accessible metadata/content
4. Extract caption/title/description
5. If video/audio available, run transcription
6. If source page can be identified, fetch original page content
7. Send combined evidence to recipe-structuring service
8. Return parsed object + confidence
9. Store raw artifacts and structured output
10. Present editable review screen to user

## 13. Data Model

## 13.1 User
- id
- email
- auth_provider
- display_name
- locale
- timezone
- subscription_status
- created_at

## 13.2 Recipe
- id
- user_id
- title
- description
- source_platform
- source_url
- source_creator_name
- source_creator_handle
- hero_image_url
- servings
- prep_minutes
- cook_minutes
- total_minutes
- cuisine
- meal_type
- notes
- parse_confidence
- import_status
- created_at
- updated_at

## 13.3 Ingredient
- id
- recipe_id
- original_text
- quantity
- unit
- item_name
- preparation_note
- sort_order

## 13.4 Step
- id
- recipe_id
- step_number
- instruction
- duration_seconds nullable
- sort_order

## 13.5 Collection
- id
- user_id
- name
- color
- icon
- created_at

## 13.6 RecipeCollection
- recipe_id
- collection_id

## 13.7 GroceryList
- id
- user_id
- title
- status
- created_at

## 13.8 GroceryItem
- id
- grocery_list_id
- ingredient_name
- quantity
- unit
- aisle_category
- checked

## 13.9 ImportJob
- id
- user_id
- input_type
- input_url
- raw_payload_ref
- status
- confidence
- failure_reason
- created_at
- completed_at

## 14. API Sketch

## 14.1 Public/mobile endpoints
- `POST /auth/signup`
- `POST /auth/login`
- `POST /imports`
- `GET /imports/{id}`
- `POST /recipes`
- `GET /recipes`
- `GET /recipes/{id}`
- `PATCH /recipes/{id}`
- `DELETE /recipes/{id}`
- `POST /collections`
- `GET /collections`
- `POST /recipes/{id}/collections`
- `POST /grocery-lists`
- `POST /grocery-lists/{id}/items`
- `GET /search?q=`

## 14.2 Internal worker endpoints/events
- `import.requested`
- `import.metadata_extracted`
- `import.transcription_completed`
- `import.source_resolved`
- `import.parsed`
- `import.failed`

## 15. Parsing and AI Strategy

## 15.1 Parsing layers
- rule-based extraction first
- LLM-assisted structuring second
- deterministic normalization third

### Why this matters
Pure LLM extraction will be expensive and inconsistent. The better design is:
- use deterministic scraping/OCR/transcription where possible
- use LLMs to convert messy content into structured recipe fields
- run normalization and validation after the LLM step

## 15.2 Confidence model
Confidence score should consider:
- count of detected ingredients
- count of detected steps
- source agreement between caption/audio/page
- presence of title and servings
- contradiction checks
- unit/quantity validation

### Confidence thresholds
- 0.85+ auto-save draft ready for quick review
- 0.60–0.84 require user review
- below 0.60 present assisted reconstruction UI

## 16. Platform Constraints and Compliance

## 16.1 iOS / macOS handoff
Development can begin on Amazon Linux for core Flutter, backend, tests, and Android work. For iOS simulator/device testing, signing, and release, the project must move onto a macOS machine with Xcode. Flutter’s official docs explicitly require Xcode and macOS for building and releasing iOS apps.  [oai_citation:4‡Flutter Docs](https://docs.flutter.dev/platform-integration/ios/setup?utm_source=chatgpt.com)

## 16.2 Source platform/legal constraints
Important:
- do not imply official affiliation with TikTok, Instagram, Pinterest, Facebook, or YouTube
- do not copy ReciMe branding, logos, or proprietary content
- preserve source attribution where relevant
- comply with applicable platform terms and copyright rules
- design for import of user-provided links/content, not large-scale unauthorized scraping

ReciMe’s own help content discusses recipe saving and copyright in the context of personal organization, which is a sensible line to stay near in product framing.  [oai_citation:5‡ReciMe](https://www.recime.app/?utm_source=chatgpt.com)

## 17. Security and Privacy
- encrypt tokens and sensitive secrets
- signed URLs for uploaded media
- least-privilege worker roles
- redact imported raw data after retention window when possible
- let users delete account and recipes
- clear privacy policy about uploaded content, OCR, and transcription processing

## 18. Monetization
### Freemium suggestion
**Free tier**
- limited imports per month
- basic collections
- basic grocery list

**Premium tier**
- unlimited imports
- advanced search filters
- meal planner
- PDF export
- web access
- smart grocery list grouping
- unit conversion
- family/shared collections

## 19. Analytics / KPIs

### Activation
- sign-up to first import completion
- first import success rate
- first-day save count

### Engagement
- weekly saved recipes per active user
- searches per active user
- grocery list generation rate
- recipes cooked per month

### Retention
- D7 / D30 retention
- number of collections created
- repeat import rate

### Quality
- parse confidence distribution
- import failure rate by source platform
- average manual edits per import
- OCR/transcription fallback frequency

## 20. Risks and Mitigations

## 20.1 Import fragility
**Risk:** social platforms change markup/metadata patterns  
**Mitigation:** isolate source adapters; rely on user-supplied links and share intents; keep fallback import modes

## 20.2 Poor parsing quality
**Risk:** messy captions/video audio create bad recipes  
**Mitigation:** confidence scoring, review screen, edit-first UX

## 20.3 Cost blowout
**Risk:** OCR/transcription/LLM costs rise fast  
**Mitigation:** staged pipeline, cache artifacts, quota system, cheaper rule-based pass first

## 20.4 iOS-specific delays
**Risk:** share extension, signing, and native integration issues surface late  
**Mitigation:** smoke test the iOS flow early on macOS/Xcode, not just at release time. Flutter’s iOS setup docs recommend simulator setup and also testing on a physical device.  [oai_citation:6‡Flutter Docs](https://docs.flutter.dev/platform-integration/ios/setup?utm_source=chatgpt.com)

## 21. Phased Delivery Plan

## Phase 0: Foundation
- auth
- recipe CRUD
- basic Flutter app shell
- backend + DB + storage
- manual text import

## Phase 1: MVP
- URL import
- share sheet import
- extraction pipeline v1
- editable recipe review
- collections
- search
- grocery list

## Phase 2
- screenshot OCR
- better transcription
- premium subscription
- meal planning
- smarter normalization

## Phase 3
- web companion app
- browser extension
- PDF export
- collaborative collections
- recommendation engine

## 22. Recommended Engineering Plan for Your Setup

Given your intended workflow:
- develop the Flutter app and backend primarily on Amazon Linux
- keep Android builds and shared tests there
- use Git-based handoff to a Mac for iOS-targeted testing
- once the first share/import flow works, move to macOS and validate:
  - iOS simulator launch
  - native share sheet
  - login
  - recipe review/edit
  - persistence
- continue iterating mostly on Linux, but regularly re-test the iOS lane on macOS

This matches Flutter’s current documented requirement for macOS/Xcode on the iOS path.  [oai_citation:7‡Flutter Docs](https://docs.flutter.dev/platform-integration/ios/setup?utm_source=chatgpt.com)

## 23. Open Questions
- Which social sources are in MVP versus later?
- Should the first release require login, or allow guest saving?
- Is nutrition estimation part of MVP?
- Is subscription required at launch?
- Is the product strictly personal, or will users be able to share collections?
- Should import jobs be synchronous for small text inputs and async for video inputs?
- Which transcription/OCR vendors fit your cost target?

## 24. Recommendation

Build an **original recipe-import organizer** inspired by the import convenience and organization model ReciMe demonstrates publicly, but keep the first version narrow:
- import from pasted URLs and share sheet
- review/edit extracted recipe
- save to library
- organize with collections
- generate grocery list

That is enough to test the core value proposition without overbuilding.