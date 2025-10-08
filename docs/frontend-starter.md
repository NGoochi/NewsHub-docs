# 🚀 NewsHub Frontend — Starter Guide

## 📌 Project Overview

**NewsHub** is a web application for journalists, researchers, and media analysts to:
- Search and import news articles from NewsAPI.ai
- Analyze articles using Gemini AI (summaries, categories, sentiment)
- Extract stakeholder quotes automatically
- Export results to Google Sheets for further analysis

The **backend is complete** and provides a full REST API. Your task is to build the React-based frontend that consumes this API and provides an intuitive user interface.

---

## 🎯 What You're Building

A dashboard-style web application with three main pages:

1. **Dashboard** — View all projects, create new ones, see import/analysis status
2. **Project Page** — View articles, run imports, trigger analysis, view quotes
3. **Settings Page** — Configure API keys, categories, and preferences (future)

---

## 🔌 Backend Connection

### Base URL
Development: `http://localhost:8080`  
Production: `https://newshub-api.onrender.com` (update as needed)

### Response Format
All API endpoints follow this structure:

**Success:**
```json
{
  "success": true,
  "data": { ... },
  "error": null
}
```

**Error:**
```json
{
  "success": false,
  "error": "Error message description"
}
```

### Authentication
Currently **none** — this is a single-user application. OAuth may be added in the future.

---

## 🧭 Core Workflows

### 1. Project Management

**Create Project:**
```
POST /projects
Body: { "name": "COP30 Coverage", "description": "..." }
→ Returns project object with ID
```

**List Projects:**
```
GET /projects
→ Returns array of projects with article counts
```

**Get Project Details:**
```
GET /projects/:id
→ Returns project with related articles
```

**Delete Project:**
```
DELETE /projects/:id
→ Removes project and all articles/quotes
```

---

### 2. Article Import (Session-based Workflow)

This is a **multi-step async process**:

#### Step 1: Preview Import
```
POST /import/preview
Body: {
  "projectId": "uuid",
  "searchTerms": ["climate change", "COP30"],
  "sourceIds": ["bbc.co.uk", "reuters.com"],  // optional
  "startDate": "2025-01-01",
  "endDate": "2025-01-31"
}
→ Returns: { estimatedArticles, sources, dateRange }
```

**UI Pattern:** Show user estimated count and confirm before importing.

#### Step 2: Start Import
```
POST /import/start
Body: { same as preview }
→ Returns: { sessionId, status: "running" }
```

**UI Pattern:** Immediately return session ID, show loading indicator.

#### Step 3: Poll Session Status
```
GET /import/session/:sessionId
→ Returns: {
  "sessionId": "uuid",
  "status": "running" | "completed" | "failed",
  "articlesFound": 150,
  "articlesImported": 120,
  "error": null
}
```

**UI Pattern:** Poll every 2-3 seconds. Update progress bar: `articlesImported / articlesFound`. Stop when status is "completed" or "failed".

#### Get Available Sources
```
GET /import/sources
→ Returns array of { id, title, country, language, sourceUri }
```

#### Get Available Countries/Languages
```
GET /import/countries  → Returns ["United States", "United Kingdom", ...]
GET /import/languages  → Returns ["eng", "deu", "spa", ...]
```

---

### 3. Article Analysis (Batch-based Workflow)

Also a **multi-step async process**:

#### Step 1: Create Analysis Batch
```
POST /analysis/batch
Body: {
  "projectId": "uuid",
  "articleIds": ["article-uuid-1", "article-uuid-2", ...]  // max 10
}
→ Returns: { batchId, status: "pending", totalArticles: 10 }
```

**UI Pattern:** User selects articles (checkboxes), max 10 at a time. For large projects, create multiple batches.

#### Step 2: Start Analysis
```
POST /analysis/batch/:batchId/start
→ Returns: { batchId, status: "running" }
```

#### Step 3: Poll Batch Status
```
GET /analysis/batch/:batchId
→ Returns: {
  "batchId": "uuid",
  "status": "running" | "completed" | "failed" | "cancelled",
  "totalArticles": 10,
  "processedArticles": 7,
  "results": { ... }
}
```

**UI Pattern:** Poll every 2-3 seconds. Progress: `processedArticles / totalArticles`. When completed, refresh article list to show analysis results.

#### Cancel Batch
```
POST /analysis/batch/:batchId/cancel
→ Stops processing
```

---

### 4. Viewing Results

#### Get Articles for Project
```
GET /articles/project/:projectId
→ Returns array of articles with:
  - Basic metadata (title, author, date, url)
  - Gemini analysis (summaryGemini, categoryGemini, sentimentGemini)
  - analysedAt timestamp (null if not yet analyzed)
```

**UI Pattern:** Show table with expandable rows. Unexpanded: title, author, date, category badge, sentiment icon. Expanded: full summary, body text, quotes.

#### Get Quotes
```
GET /quotes?articleId=:articleId  → Quotes for specific article
GET /quotes?projectId=:projectId  → All quotes in project
→ Returns array of quotes with stakeholder name, affiliation, quote text
```

**UI Pattern:** Quotes tab showing stakeholder name + affiliation + quote. Group by stakeholder or by article.

---

### 5. Export

```
POST /export/:projectId
→ Creates Google Sheet with Articles and Quotes tabs
→ Returns: { url: "https://docs.google.com/spreadsheets/..." }
```

**UI Pattern:** Show loading spinner, then open sheet URL in new tab or display link.

---

## 🎨 Design System

### Tech Stack
- **TailwindCSS** for utility-first styling
- **Shadcn/UI** for component patterns (buttons, cards, dialogs, tables, tabs)
- **Lucide Icons** for clean vector icons
- Typography: Sans-serif (Inter or similar)

### Visual Identity
Theme: Calm, precise, editorial — like a tool journalists actually use.

**Colors:**
- Background: `bg-slate-950` (dark mode) or `bg-slate-50` (light mode)
- Cards: `bg-slate-900/40` with soft shadows
- Accent: `bg-blue-600` for primary actions
- Success: `text-green-500`
- Warning: `text-yellow-500`
- Error: `text-red-500`

**Components:**
- Rounded corners: `rounded-2xl` for cards, `rounded-lg` for buttons
- Shadows: `shadow-md` for elevation
- Transitions: Subtle hover states with `transition`

See [`style-guidelines.md`](./style-guidelines.md) for complete design reference.

---

## 🧩 Key UI Patterns

### 1. Real-time Status Updates
Use **polling** for long-running operations:
- Import sessions: poll every 2-3 seconds until status is "completed" or "failed"
- Analysis batches: poll every 2-3 seconds until "completed", "failed", or "cancelled"

**Recommended:** Use `useEffect` with `setInterval` or React Query with `refetchInterval`.

### 2. Progress Indicators
Always show visual feedback:
- Linear progress bar: `(current / total) * 100`
- Status badges: color-coded by state (running = blue, completed = green, failed = red)
- Spinners for loading states

### 3. Async Operation Pattern
```
1. User action (click "Start Import")
2. Immediately show loading UI
3. API call returns session/batch ID
4. Start polling for status
5. Update UI with progress
6. On completion: stop polling, show success message, refresh data
```

### 4. Expandable Rows
Articles list uses accordion pattern:
- Compact row: title, author, date, category, sentiment
- Expanded: summary, full body text, quotes list, action buttons

### 5. Multi-select for Batch Operations
Allow users to select multiple articles (checkboxes) for analysis.
Show count of selected items. Enforce max of 10 per batch.
For larger selections, auto-create multiple batches.

---

## 🧠 State Management

### Server State (API Data)
**Recommended: TanStack Query (React Query)**

Benefits:
- Built-in caching and refetching
- Loading and error states
- Easy polling with `refetchInterval`
- Optimistic updates

Example:
```typescript
const { data: projects, isLoading } = useQuery({
  queryKey: ['projects'],
  queryFn: async () => {
    const res = await fetch('http://localhost:8080/projects');
    const json = await res.json();
    return json.data;
  }
});
```

### Client State (UI State)
**Recommended: Zustand or React Context**

Track:
- Selected articles (for batch analysis)
- Active project ID
- Modal open/closed states
- Notification toasts

---

## 🛠️ Suggested Tech Stack

### Core
- **React 18+** with TypeScript
- **Next.js 14+** (App Router) for SSR and routing
- **TailwindCSS** for styling
- **Shadcn/UI** for components

### Data Fetching
- **TanStack Query** for server state
- **Axios** or native `fetch` for HTTP requests

### Forms
- **React Hook Form** for form validation
- **Zod** for schema validation

### UI Components
- **Shadcn/UI**: Button, Card, Dialog, Tabs, Table, Badge, Progress
- **Lucide React** for icons
- **Framer Motion** (optional) for animations

### Dev Tools
- **ESLint** + **Prettier** for code quality
- **TypeScript** for type safety

---

## 📋 Development Priorities

Build in order of user workflow:

### Phase 1: Projects (Foundation)
1. Dashboard page with project list
2. Create project modal
3. Delete project with confirmation
4. Navigate to project page

### Phase 2: Import (Data Entry)
1. Import form with preview
2. Source selection (checkboxes with search/filter)
3. Start import and show progress
4. Session status polling
5. Display imported articles in table

### Phase 3: Analysis (Core Value)
1. Article selection UI (checkboxes)
2. Create analysis batch
3. Start batch and show progress
4. Status polling
5. Display analysis results (summary, category, sentiment badges)

### Phase 4: Quotes (Secondary Value)
1. Quotes tab on project page
2. Display quotes grouped by stakeholder
3. Link quotes back to source articles

### Phase 5: Export (Output)
1. Export button
2. Loading state
3. Open Google Sheet link

### Phase 6: Polish
1. Error handling and user feedback (toasts)
2. Empty states and loading skeletons
3. Responsive design (mobile/tablet)
4. Dark/light mode toggle

---

## 📚 API Reference Quick Sheet

### Projects
| Method | Endpoint | Body | Returns |
|--------|----------|------|---------|
| GET | `/projects` | — | Array of projects |
| GET | `/projects/:id` | — | Single project with articles |
| POST | `/projects` | `{ name, description }` | Created project |
| PUT | `/projects/:id` | `{ name, description }` | Updated project |
| DELETE | `/projects/:id` | — | Success confirmation |

### Import
| Method | Endpoint | Body | Returns |
|--------|----------|------|---------|
| POST | `/import/preview` | Search params | Estimated count + sources |
| POST | `/import/start` | Search params | Session ID |
| GET | `/import/session/:sessionId` | — | Session status + progress |
| POST | `/import/session/:sessionId/cancel` | — | Cancelled confirmation |
| GET | `/import/sources` | — | Array of available sources |
| GET | `/import/countries` | — | Array of countries |
| GET | `/import/languages` | — | Array of languages |

### Analysis
| Method | Endpoint | Body | Returns |
|--------|----------|------|---------|
| POST | `/analysis/batch` | `{ projectId, articleIds }` | Batch ID |
| POST | `/analysis/batch/:batchId/start` | — | Started confirmation |
| GET | `/analysis/batch/:batchId` | — | Batch status + progress |
| POST | `/analysis/batch/:batchId/cancel` | — | Cancelled confirmation |
| GET | `/analysis/project/:projectId/batches` | — | All batches for project |

### Articles
| Method | Endpoint | Body | Returns |
|--------|----------|------|---------|
| GET | `/articles/project/:projectId` | — | All articles in project |
| GET | `/articles/:id` | — | Single article |
| PUT | `/articles/:id` | Updated fields | Updated article |
| DELETE | `/articles/:id` | — | Success confirmation |

### Quotes
| Method | Endpoint | Query Params | Returns |
|--------|----------|--------------|---------|
| GET | `/quotes` | `?projectId=X` or `?articleId=Y` | Array of quotes |
| GET | `/quotes/:id` | — | Single quote |
| DELETE | `/quotes/:id` | — | Success confirmation |

### Export
| Method | Endpoint | Body | Returns |
|--------|----------|------|---------|
| POST | `/export/:projectId` | — | Google Sheet URL |

---

## 🧪 Testing the Backend

Before building UI, test endpoints using:
- **Postman** or **Thunder Client** (VS Code extension)
- **cURL** commands
- Browser for GET requests

Example test:
```bash
# Create a project
curl -X POST http://localhost:8080/projects \
  -H "Content-Type: application/json" \
  -d '{"name":"Test Project","description":"Testing API"}'

# List projects
curl http://localhost:8080/projects
```

---

## 🚨 Common Pitfalls to Avoid

1. **Don't poll too aggressively** — 2-3 second intervals are sufficient. Faster = API overload.
2. **Handle loading states** — Always show spinners/skeletons when fetching data.
3. **Handle errors gracefully** — Display user-friendly error messages, not raw API errors.
4. **Validate max batch size** — Enforce 10 article limit for analysis batches in UI.
5. **Stop polling on unmount** — Clean up intervals in `useEffect` return functions.
6. **Optimize table rendering** — Use virtualization for large article lists (react-window or tanstack-virtual).
7. **Cache API responses** — Use TanStack Query cache to reduce redundant requests.

---

## 🎯 Success Criteria

Your frontend is ready when:
- ✅ User can create projects and see them listed
- ✅ User can import articles with progress tracking
- ✅ User can select articles and run analysis with progress tracking
- ✅ User can view analysis results (summaries, categories, sentiment, quotes)
- ✅ User can export projects to Google Sheets
- ✅ All async operations show clear loading/progress indicators
- ✅ Errors are caught and displayed with helpful messages
- ✅ UI is responsive and visually polished
- ✅ Code follows TypeScript best practices

---

## 📖 Additional Resources

- **Backend Docs:** [`instructions.md`](./instructions.md) — Full API architecture
- **Data Models:** [`definitions.md`](./definitions.md) — Entity schemas and relationships
- **Design System:** [`style-guidelines.md`](./style-guidelines.md) — UI styling conventions
- **UI Specs:** [`/UI/`](./UI/) — Detailed page layouts and component specs

---

## 🚀 Getting Started Checklist

- [ ] Set up Next.js + TypeScript project
- [ ] Install dependencies: TailwindCSS, Shadcn/UI, TanStack Query, Axios
- [ ] Configure Tailwind with design tokens
- [ ] Create API client utility (base URL, error handling)
- [ ] Build Dashboard with project list
- [ ] Implement project creation modal
- [ ] Build Project page with article table
- [ ] Implement import workflow with preview + progress
- [ ] Implement analysis workflow with batch creation + progress
- [ ] Add quotes tab
- [ ] Add export button
- [ ] Polish UI (loading states, errors, toasts)
- [ ] Test all workflows end-to-end

---

**Welcome to NewsHub frontend development. Build something great! 🎉**
