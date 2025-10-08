🧭 NewsHub — Project Instructions (v0.1)

Comprehensive system architecture and development guide for the NewsHub web application.
This file defines scope, structure, and functional logic for all modules within the Express-based server environment.
It is designed for use within Cursor to maintain global project context and ensure consistent code generation.

🧱 Project Summary

Goal:
A server-based application that retrieves, analyses, and organises news articles from external APIs, producing structured insights about sentiment, categories, and stakeholder quotes.

Core Capabilities:

Search — Retrieve and catalogue news articles via NewsAPI.

Analysis — Run AI-powered analysis through Gemini to extract summaries, sentiment, categories, and quotes.

Export — Output analysed datasets to Google Sheets for further study.

Stack:

Backend Framework: Express (TypeScript)

Database: PostgreSQL (Neon)

ORM: Prisma

Hosting: Render

Integrations: NewsAPI, Gemini API, Google Sheets API

🧩 Core Architecture
src/
  index.ts                 → Entry point / server setup
  routes/                  → Express routes
    projects.ts            → Project CRUD endpoints
    articles.ts            → Article CRUD endpoints
    quotes.ts              → Quote CRUD endpoints
    import.ts              → Import preview, start, session management
    analysis.ts            → Analysis batch creation and management
    export.ts              → Google Sheets export endpoints
    settings.ts            → Settings endpoints (placeholder)
  controllers/             → Business logic for routes
    projectController.ts   → Project CRUD operations
    articleController.ts   → Article CRUD and project queries
    quoteController.ts     → Quote CRUD operations
    importController.ts    → Import workflow coordination
    analysisController.ts  → Analysis batch orchestration
    exportController.ts    → Export to Google Sheets
    settingsController.ts  → Settings management (stub)
  lib/
    db.ts                  → Prisma client instance
    newsapi.ts             → NewsAPI.ai client and data normalization
    importService.ts       → Import workflow orchestration
    importSession.ts       → Async import session management
    analysisBatch.ts       → Analysis batch processing service
    gemini.ts              → Gemini API integration (analysis + quotes)
    sheets.ts              → Google Sheets export helper
  utils/
    constants.ts           → Shared constants and enums
    errorHandler.ts        → Express error middleware
    formatters.ts          → Data cleaning and transformation
    validation.ts          → Input validation helpers

⚙️ Server Responsibilities
Express Application

Initialise middleware (cors, json, dotenv)

Register all route modules

Provide basic health endpoint (/)

Handle global errors gracefully

Serve as API only (no frontend rendering)

Database (PostgreSQL via Prisma)

Stores Projects, Articles, Quotes

Manages relations and cascade deletions

Tracks analysis progress via analysedAt field

Enforces enums for inputMethod and sentimentGemini

APIs
Function	Route	Description
Project Management	/projects	CRUD for projects
Project Details	/projects/:id	Get specific project with metadata
Article CRUD	/articles	Get all articles (with query params)
Article by Project	/articles/project/:projectId	Get all articles for a project
Article Details	/articles/:id	Get/update/delete specific article
Import Preview	/import/preview	Preview estimated article count before importing
Start Import	/import/start	Creates ImportSession and begins async import
Import Session Status	/import/session/:sessionId	Get status of ongoing/completed import
Cancel Import	/import/session/:sessionId/cancel	Cancel a running import session
Project Import Sessions	/import/project/:projectId/sessions	Get all import sessions for a project
Project Stats	/import/project/:projectId/stats	Get import statistics for a project
Get Sources	/import/sources	Get all available news sources
Get Countries	/import/countries	Get list of available countries
Get Languages	/import/languages	Get list of available languages
Create Analysis Batch	/analysis/batch	Create new batch for up to 10 articles
Start Analysis	/analysis/batch/:batchId/start	Begin processing an analysis batch
Batch Status	/analysis/batch/:batchId	Get status of specific analysis batch
Cancel Batch	/analysis/batch/:batchId/cancel	Cancel a running analysis batch
Project Batches	/analysis/project/:projectId/batches	Get all analysis batches for a project
Quote CRUD	/quotes	Get all quotes (with query params)
Quote Details	/quotes/:id	Get/update/delete specific quote
Export to Sheets	/export/:projectId	Builds and sends data to Google Sheets
Export Status	/export/status/:projectId	Get export status for a project
Download Export	/export/download/:projectId	Download export file
Settings	/settings	Get application settings (placeholder)
Prompts	/settings/prompts	Get/update Gemini prompts (placeholder)
Categories (Legacy)	/settings/categories	Get/update category definitions (placeholder - deprecated)
Get Categories	/categories	Get all categories (active by default)
Get Category	/categories/:id	Get specific category by ID
Create Category	/categories	Create new category with name, definition, keywords
Update Category	/categories/:id	Update category fields
Delete Category	/categories/:id	Soft delete category (set isActive=false)
Reorder Categories	/categories/reorder	Batch update sortOrder for multiple categories
🧠 Data Flow Overview
1. Creating a Project

User hits /projects → POST with { name, description }

Database creates entry with timestamps

Returns project object with ID for later references

2. Importing Articles (Session-based Workflow)

**Step 1: Preview**

Client posts search parameters to /import/preview
```json
{
  "projectId": "uuid",
  "searchTerms": ["climate", "COP30"],
  "sourceIds": ["bbc.co.uk", "reuters.com"],
  "startDate": "2025-01-01",
  "endDate": "2025-01-31"
}
```

Backend queries SearchSource database and estimates article count

Returns preview data: estimated count, selected sources, date range

**Step 2: Start Import**

Client confirms and posts same parameters to /import/start

Backend creates ImportSession record with status "running"

Returns session ID immediately for monitoring

**Step 3: Async Processing**

ImportSession.processImportSession() runs in background

Fetches articles from NewsAPI.ai in batches

Saves articles to database with importSessionId link

Updates session: articlesFound, articlesImported

**Step 4: Monitor Progress**

Frontend polls /import/session/:sessionId

Backend returns current status, counts, errors

When complete, status becomes "completed" or "failed"

3. Running Analysis (Batch-based Workflow)

**Step 1: Create Batch**

User selects up to 10 unanalyzed articles

Client posts to /analysis/batch
```json
{
  "projectId": "uuid",
  "articleIds": ["article-uuid-1", "article-uuid-2", ...]
}
```

Backend creates AnalysisBatch record with status "pending"

Returns batch ID for monitoring

**Step 2: Start Processing**

Client posts to /analysis/batch/:batchId/start

Backend updates status to "running"

Fetches article content from database

**Step 3: Gemini Processing**

Sends batch to analyzeArticles() - returns summaries, categories, sentiment

Sends batch to extractQuotes() - returns stakeholder quotes

Updates Article records with analysis results

Creates Quote records linked to articles

Updates AnalysisBatch: processedArticles++, results stored

**Step 4: Monitor Progress**

Frontend polls /analysis/batch/:batchId

Backend returns status, totalArticles, processedArticles

When complete, status becomes "completed", "failed", or "cancelled"

**For Large Projects**: Frontend can create multiple batches of 10 articles each

4. Viewing Results

Client fetches /articles/project/:projectId for all articles

Each article now has summaryGemini, categoryGemini, sentimentGemini populated

Client fetches /quotes?projectId=X to see extracted stakeholder quotes

Quotes linked back to source articles via articleId

5. Exporting Data

User triggers /export/:projectId

Server gathers all Articles + Quotes for project

Formats data and sends to Google Sheets API

Creates spreadsheet with two tabs: Articles, Quotes

Returns shareable link to the created sheet

Note: Export implementation requires verification

🧩 Module Responsibilities
/lib/db.ts

Exports a single Prisma client instance shared across all controllers:

```typescript
import { PrismaClient } from "@prisma/client";
const prisma = new PrismaClient();
export default prisma;
```

/lib/newsapi.ts

Provides NewsAPIClient class for interacting with NewsAPI.ai

Builds search requests with filters (keywords, sources, dates)

Fetches articles from NewsAPI.ai REST API

Normalizes response data into Article schema format

Handles pagination and error responses

/lib/importService.ts

Orchestrates the article import workflow

Implements ImportService class with preview and start methods

Validates search parameters and source selections

Creates and manages ImportSession records

Coordinates between NewsAPIClient and ImportSessionManager

/lib/importSession.ts

Implements ImportSessionManager class for async import processing

Creates ImportSession database records

Processes imports asynchronously in background

Fetches articles in batches from NewsAPI.ai

Saves articles to database with deduplication logic

Updates session status and progress counters

/lib/gemini.ts

Handles all Gemini API communication

Loads prompt templates from database (PromptTemplate table) with 5-minute cache

Implements analyzeArticles() - sends up to 10 articles, returns analysis

Implements extractQuotes() - extracts stakeholder quotes

Loads category definitions from database (Category table) with 5-minute cache

Exports clearPromptCache() - called when categories are updated via API

Configurable timeout via GEMINI_TIMEOUT_MS environment variable (default: 300000ms / 5 minutes)

Progress indicators: logs start time, article count, timeout value, completion time, and results summary

Parses Gemini JSON responses and validates structure

Handles API errors and retries

Note: Prompts and categories cached for 5 minutes to optimize performance. Cache automatically cleared on category modifications.

/lib/analysisBatch.ts

Implements AnalysisBatchService class for batch analysis workflow

Creates AnalysisBatch records in database

Orchestrates dual analysis: articles + quotes

Manages batch lifecycle: pending → running → completed/failed

Updates Article records with Gemini results

Creates Quote records from extracted quotes

Tracks progress: processedArticles / totalArticles

Supports batch cancellation

/lib/sheets.ts

Manages Google Sheets export functionality

Handles Google OAuth authentication

Accepts project ID → fetches Articles + Quotes

Formats data for spreadsheet export

Creates Google Sheet with two tabs (Articles, Quotes)

Returns shareable URL

Note: Implementation exists but requires verification

🖥️ UI-Independent Design Philosophy

Although the backend currently has no UI, all responses are formatted to be frontend-agnostic, ready for future integration with a React dashboard.

Response structure convention:

{
  "success": true,
  "data": {...},
  "error": null
}


Error example:

{
  "success": false,
  "error": "Failed to fetch from NewsAPI"
}

🧰 Development Conventions

All TypeScript files use ES2020 syntax.

Each route file defines only routes; logic lives in controllers.

Environment variables stored in .env:

DATABASE_URL=
NEWS_API_KEY=
NEWSAPI_MAX_TOTAL_ARTICLES=100  # Max articles per NewsAPI search (default: 100)
GEMINI_API_KEY=
GEMINI_TIMEOUT_MS=300000  # Gemini API timeout in milliseconds (default: 300000 / 5 minutes)
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
GOOGLE_REFRESH_TOKEN=


Use try/catch and async/await consistently.

Use relative imports (@/lib/db etc.) when possible.

🧱 Scalability Notes

Database: PostgreSQL on Neon (free tier scalable).

Job Queue: In early stages, runs as a simple table-driven loop.
Later upgrade path → Redis or a lightweight queue like BullMQ.

Worker Separation: Optional future Render service for independent processing.

Authentication: Currently single-user; OAuth planned for future expansion.

🚧 Development Phases - Backend
Phase	Description	Status
1	Express + Prisma base, database migration	✅ Complete
2	CRUD endpoints for Projects, Articles, Quotes	✅ Complete
3	NewsAPI.ai integration + session-based import	✅ Complete
4	Gemini analysis + batch processing system	✅ Complete
5	Google Sheets export	🔧 Routes Implemented (needs verification)
6	Settings + editable prompts	🔧 Stub Only (returns placeholders)

🚧 Development Phases - Frontend
Phase	Description	Status
1	Projects (Foundation) - Dashboard with project CRUD	✅ Complete
2	Import (Data Entry) - Article import workflow with progress	✅ Complete
3	Analysis (Core Value) - Batch analysis with Gemini	✅ Complete
4	Quotes (Secondary Value) - Quote display and management	🔜 Pending
5	Export (Output) - Google Sheets export	🔜 Pending
6	Polish - Error handling, responsive design, loading states	🔜 Pending

**Note:** Mark phases with ✅ as they are completed.

## Implemented Features

### Import System ✅
- Session-based import with preview functionality
- Source filtering by country, language, region
- Progress tracking with ImportSession records
- Async processing with status monitoring
- Boolean query support for advanced searches

### Analysis System ✅
- Batch-based processing (up to 10 articles per batch)
- Dual analysis: article summaries + quote extraction
- Progress tracking with AnalysisBatch records
- Cancellation support
- Category definitions loaded from prompts/category-definitions.md

### Data Model ✅
- Full NewsAPI.ai integration with extended fields
- Import session history tracking
- Analysis batch history tracking
- Source management database (SearchSource)

## Still To Do

### High Priority:
- **Settings Implementation**: Replace placeholder endpoints with actual settings storage
  - Store API keys in database (encrypted)
  - Allow category editing through API
  - Enable prompt customization
- **Export Verification**: Test and verify Google Sheets export functionality
- **Frontend UI**: Build React dashboard to consume all these APIs

### For NewsAPI Import:
- Add UI for real-time article preview during import
- Enhanced boolean query builder interface
- Additional NewsAPI.ai filter parameters (sentiment range, source ranking)

### For Gemini Analysis:
- **Multi-batch Orchestration**: Auto-split large article sets into multiple batches of 10
- **Translation Detection**: Enhance prompts to better detect non-English articles
- **Category Database Storage**: Move categories from markdown file to database for editing

### Nice to Have:
- **Stakeholder Tracking**: Aggregate quotes by stakeholder across projects
- **Report Generation**: Generate summary reports from analyzed articles with PDF export
- **Retry Logic**: Auto-retry failed batches with exponential backoff
- **Webhook Notifications**: Notify when imports/analysis complete
🧩 Coding Style Reference

Functions use camelCase.

Database tables mirror schema field names exactly.

Keep controllers pure (no side effects outside DB + API).

Always validate external input.

Prefer modular exports: one function per responsibility.

Every route should have an accompanying controller test file.

💾 Testing Strategy (future)

Unit tests for controllers (using Jest).

Mocked API responses for NewsAPI and Gemini.

Local test DB via Prisma with sqlite for CI.

Manual API tests through Postman or Thunder Client.

🧩 Core Design Goals

Reliability: handle API rate limits + batching gracefully.

Transparency: clear logs and progress tracking for all jobs.

Extensibility: every module can be replaced (e.g., swap Gemini for another LLM).

Simplicity: human-readable data models, predictable structure.

Low cost: all free-tier compatible.

🚀 Quick Setup Summary

Clone repo → install deps → connect Neon.

npx prisma migrate dev

npm run dev → local API server

Deploy to Render → link DATABASE_URL

Confirm / endpoint → “NewsHub API running!”

🧠 Cursor Context Meta

These lines guide Cursor’s internal context weighting and retrieval behaviour during code generation, refactoring, and commentary.

---

# 🧠 Cursor Context Meta (for internal guidance)

**Primary Context Files:**
- `docs/definitions.md` — contains project lexicon and terminology
- `docs/instructions.md` — contains architecture, workflow, and development conventions

**Cursor Behaviour Guidelines:**

1. **Global Awareness**
   - Always read both context files (`definitions.md` and `instructions.md`) before writing or modifying code.
   - Treat these as the single source of truth for:
     - Field names and database schema
     - API endpoint purposes
     - Relationships between Project → Articles → Quotes
     - The Gemini analysis flow
     - Development style conventions

2. **Priority Hierarchy**
   - When writing or editing code:
     1. Match the **data model** and **API behaviours** described here.
     2. Follow naming and structure conventions exactly.
     3. Preserve schema field names from `definitions.md`.
     4. Implement new logic modularly (no duplication across controllers or utils).

3. **Code Generation Scope**
   - Code should always:
     - Be written in **TypeScript** for Express.
     - Use **Prisma** for all database access.
     - Respect the architecture in `/routes`, `/controllers`, `/lib`, and `/jobs`.
   - Each function should have a clearly named export.
   - Keep API responses structured:
       ```ts
       { success: true, data, error: null }
       ```
       or
       ```ts
       { success: false, error: "message" }
       ```

4. **When Building New Components**
   - Reference these docs to determine which folder the new file belongs in.
   - If unsure of naming, default to patterns already defined in `instructions.md`.
   - Never create frontend files; backend only.

5. **When Unsure**
   - Cursor should check both docs for matching definitions before guessing field names or inventing schema.
   - Prefer function stubs with TODO comments rather than hallucinating logic.

6. **Tone & Style**
   - Keep code clean, modular, and well-annotated.
   - Comments should clarify logic, not narrate steps.
   - Avoid over-abstracting; favour clarity and maintainability.

---

*End of Context Meta*