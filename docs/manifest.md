🗺️ NewsHub — Project Manifest (v0.1)

A concise file map and component purpose guide.
Used by Cursor and developers to quickly understand where specific functionality lives within the NewsHub repository.
This file should stay updated as new routes, controllers, or utilities are added.

📁 Root Directory
/newshub-express
│
├── prisma/                  → Database schema + migrations
│   └── schema.prisma
│
├── src/                     → Application source
│   ├── index.ts             → Main Express entry point
│   ├── routes/              → Route definitions
│   ├── controllers/         → Business logic
│   ├── lib/                 → External API helpers (NewsAPI, Gemini, Sheets)
│   ├── jobs/                → Background queue + worker logic
│   ├── utils/               → Helper modules (validation, formatting, etc.)
│   └── tests/               → (Optional) automated tests
│
├── docs/                    → Documentation + context files
│   ├── definitions.md       → Project lexicon and terminology
│   ├── instructions.md      → Architecture + development blueprint
│   ├── manifest.md          → This file
│
├── .env                     → Environment variables (API keys, DB URL)
├── package.json             → Scripts, dependencies, metadata
└── tsconfig.json            → TypeScript configuration

🧩 Folder Breakdown
/src/routes

Express routers grouped by domain.
Each file only defines HTTP endpoints — all logic is imported from controllers.

File	Description
projects.ts	CRUD routes for Projects
articles.ts	Article CRUD, get by project, manual creation
quotes.ts	Quote CRUD operations
import.ts	Import preview, start, session status, source management
analysis.ts	Analysis batch creation, start, status, cancellation
export.ts	Export project data to Google Sheets
settings.ts	App settings and configuration (placeholder implementation)
/src/controllers

Implements logic for each route.
Each controller focuses on database operations and external API calls, returning formatted responses.

File	Responsibilities	Status
projectController.ts	Create, read, update, delete Projects	✅ Implemented
articleController.ts	Insert, fetch, edit, delete Articles; query by project	✅ Implemented
quoteController.ts	CRUD for Quotes	✅ Implemented
importController.ts	Preview imports, start sessions, monitor progress, source management	✅ Implemented
analysisController.ts	Create batches, start processing, monitor status, cancel batches	✅ Implemented
exportController.ts	Generate and send Google Sheets exports	🔧 Routes exist, needs verification
settingsController.ts	Manage settings, prompts, categories	⚠️ Stub only (returns placeholders)
/src/lib

Houses helper modules for third-party APIs and reusable utilities.

File	Description	Status
db.ts	Exports a singleton Prisma client	✅ Implemented
newsapi.ts	NewsAPIClient class: builds requests, fetches articles, normalizes data	✅ Implemented
importService.ts	ImportService class: orchestrates preview and import workflows	✅ Implemented
importSession.ts	ImportSessionManager class: handles async import processing	✅ Implemented
analysisBatch.ts	AnalysisBatchService class: manages batch lifecycle and Gemini calls	✅ Implemented
gemini.ts	Gemini API integration: analyzeArticles(), extractQuotes()	✅ Implemented
sheets.ts	Handles Google Sheets creation and export routines	🔧 Exists, needs verification
/src/jobs

Note: Original job queue system has been replaced with AnalysisBatch model.
Analysis is now managed through the AnalysisBatchService class in /lib/analysisBatch.ts.
Import processing is handled through ImportSessionManager in /lib/importSession.ts.

Previous files (if they exist) are deprecated in favor of the batch-based approach.
/src/utils

General helpers and validation tools.

File	Description
validation.ts	Schema validation for incoming data
formatters.ts	Cleans and formats API or database results
errorHandler.ts	Express error middleware
constants.ts	Shared config (e.g., sentiment enums, status codes)
/prisma

Prisma ORM setup.

File	Description
schema.prisma	Database schema: Project, Article, Quote, ImportSession, AnalysisBatch, AnalysisJob, SearchSource
/migrations/	Auto-generated SQL migration history

Key Models:
- **Project**: Root entity containing articles and tracking relationships
- **Article**: Extended with NewsAPI.ai fields (sourceUri, concepts, categories, sentiment, imageUrl, location)
- **Quote**: Stakeholder quotes extracted by Gemini
- **ImportSession**: Tracks article import operations with progress
- **AnalysisBatch**: Groups articles for batch Gemini processing
- **SearchSource**: Reference data for available news sources
- **AnalysisJob**: Individual job records (used alongside batches)
/docs

Contains all internal documentation and AI context files.

File	Purpose
definitions.md	Glossary of entities, terminology, and data model
instructions.md	System architecture, logic flow, and conventions
manifest.md	File reference map (this document)
⚙️ Developer Notes

New features must include:

Route definition in /routes

Controller implementation in /controllers

Optional util or lib helper

Avoid placing logic directly inside routes.

Keep all Prisma calls inside controllers or /lib/*.ts service classes.

Each new module must be referenced here when added.

Key Patterns:

- **Session-based workflows**: Long-running operations (import, analysis) use session/batch records for progress tracking
- **Async processing**: Operations return immediately with session ID, process in background
- **Service classes**: Complex logic lives in /lib service classes (ImportService, AnalysisBatchService)
- **Controller simplicity**: Controllers validate inputs and delegate to services

Implementation Status Legend:
- ✅ Implemented: Fully working, tested
- 🔧 Exists: Code present but needs verification/testing
- ⚠️ Stub: Placeholder implementation only

🧠 Cursor Context Meta (lightweight reference)
This manifest provides Cursor with a structural overview.
When generating new code, match new files to the correct directory
based on their described purpose. Always cross-reference with:

- `definitions.md` for data naming consistency
- `instructions.md` for logic and architectural patterns