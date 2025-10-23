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

**Common Error Examples:**
```json
// Authentication errors
{ "success": false, "error": "Access token required" }
{ "success": false, "error": "Invalid token" }
{ "success": false, "error": "Token expired" }
{ "success": false, "error": "Invalid password" }

// Validation errors
{ "success": false, "error": "Missing required fields: name, description" }
{ "success": false, "error": "Invalid project ID format" }

// Not found errors
{ "success": false, "error": "Project not found" }
{ "success": false, "error": "Article not found" }
```

### Authentication
JWT token-based authentication with a single app password.

**Login Flow:**
1. User enters password on login page
2. Frontend sends to `POST /auth/login` with `{ password }`
3. Backend returns JWT token valid for 24 hours
4. Frontend stores token in localStorage or cookie
5. Frontend includes token in all API requests

**Example Login:**
```typescript
const response = await fetch('http://localhost:8080/auth/login', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ password: 'your_password' })
});
const { data } = await response.json();
// Response format: { token, expiresIn, expiresAt }
localStorage.setItem('authToken', data.token);
localStorage.setItem('tokenExpiry', data.expiresAt);
```

**Example Authenticated Request:**
```typescript
const token = localStorage.getItem('authToken');
const response = await fetch('http://localhost:8080/projects', {
  headers: {
    'Authorization': `Bearer ${token}`
  }
});
```

**401/403 Handling:**
- 401 (Unauthorized): No token provided - redirect to login
- 403 (Forbidden): Invalid/expired token - redirect to login

**Token Verification:**
```typescript
// Check if token is still valid
const response = await fetch('http://localhost:8080/auth/verify', {
  headers: { 'Authorization': `Bearer ${token}` }
});
const { data } = await response.json();
// Response: { valid: true, user: {...} }
```

**Token Management Best Practices:**
```typescript
// 1. Check token expiry before making requests
const tokenExpiry = localStorage.getItem('tokenExpiry');
if (tokenExpiry && new Date() > new Date(tokenExpiry)) {
  // Token expired, redirect to login
  localStorage.removeItem('authToken');
  localStorage.removeItem('tokenExpiry');
  window.location.href = '/login';
}

// 2. Create API client with automatic token handling
class ApiClient {
  private baseURL = 'http://localhost:8080';
  
  private getAuthHeaders() {
    const token = localStorage.getItem('authToken');
    return token ? { 'Authorization': `Bearer ${token}` } : {};
  }
  
  async request(endpoint: string, options: RequestInit = {}) {
    const response = await fetch(`${this.baseURL}${endpoint}`, {
      ...options,
      headers: {
        'Content-Type': 'application/json',
        ...this.getAuthHeaders(),
        ...options.headers
      }
    });
    
    if (response.status === 401 || response.status === 403) {
      // Token invalid/expired, redirect to login
      localStorage.removeItem('authToken');
      localStorage.removeItem('tokenExpiry');
      window.location.href = '/login';
      return;
    }
    
    return response.json();
  }
}
```

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
→ Returns array of projects with article counts and archived status
```

**Get Project Details:**
```
GET /projects/:id
→ Returns project with related articles
```

**Archive Project:**
```
PUT /projects/:id/archive
→ Sets archived = true
```

**Unarchive Project:**
```
PUT /projects/:id/unarchive
→ Sets archived = false
```

**Bulk Archive Projects:**
```
POST /projects/bulk-archive
Body: { "projectIds": ["uuid1", "uuid2", ...] }
→ Archives multiple projects
```

**Bulk Unarchive Projects:**
```
POST /projects/bulk-unarchive
Body: { "projectIds": ["uuid1", "uuid2", ...] }
→ Unarchives multiple projects
```

**Delete Project:**
```
DELETE /projects/:id
→ Removes project and all articles/quotes (only if archived)
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

### 5. Category Management

Categories are now stored in the database and fully editable through the API.

#### Get All Categories
```
GET /categories
→ Returns array of categories ordered by sortOrder
```

Query params:
- `includeInactive=true` - Include deactivated categories

#### Create Category
```
POST /categories
Body: {
  "name": "New Category",
  "definition": "Description of what belongs here",
  "keywords": ["keyword1", "keyword2", "keyword3"],
  "sortOrder": 99  // optional, auto-calculated if omitted
}
→ Returns: created category
```

**UI Pattern:** Form with name (required), definition (required), keywords (array input), sortOrder (optional).

#### Update Category
```
PUT /categories/:id
Body: {
  "name": "Updated Name",           // optional
  "definition": "Updated definition", // optional
  "keywords": ["new", "keywords"],   // optional
  "sortOrder": 5,                    // optional
  "isActive": true                   // optional
}
→ Returns: updated category
```

**UI Pattern:** Edit form, all fields optional (only send changed fields).

#### Delete Category
```
DELETE /categories/:id
→ Soft deletes (sets isActive = false)
```

**UI Pattern:** Confirmation dialog before deletion.

#### Reorder Categories
```
PUT /categories/reorder
Body: {
  "categoryOrders": [
    { "id": "uuid-1", "sortOrder": 0 },
    { "id": "uuid-2", "sortOrder": 1 },
    { "id": "uuid-3", "sortOrder": 2 }
  ]
}
→ Returns: success confirmation
```

**UI Pattern:** Drag-and-drop interface, batch update on completion.

**Important:** All category modifications automatically clear the backend cache, so changes are reflected immediately in analysis.

---

### 6. Export

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
1. Dashboard page with project list (showing archived status)
2. Create project modal
3. Archive/unarchive project functionality
4. Bulk archive/unarchive operations
5. Delete project with confirmation (only if archived)
6. Navigate to project page

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

### Phase 5: Category Management (Settings)
1. Settings page with Categories tab
2. Display category list with sortable table
3. Create category form (name, definition, keywords)
4. Edit category inline or in modal
5. Delete category with confirmation
6. Drag-and-drop reordering with batch update

### Phase 6: Export (Output)
1. Export button
2. Loading state
3. Open Google Sheet link

### Phase 7: Polish
1. Error handling and user feedback (toasts)
2. Empty states and loading skeletons
3. Responsive design (mobile/tablet)
4. Dark/light mode toggle

---

## 📚 API Reference Quick Sheet

### Projects
| Method | Endpoint | Body | Returns |
|--------|----------|------|---------|
| GET | `/projects` | — | Array of projects (includes `archived` field) |
| GET | `/projects/:id` | — | Single project with articles |
| POST | `/projects` | `{ name, description }` | Created project |
| PUT | `/projects/:id` | `{ name, description }` | Updated project |
| PUT | `/projects/:id/archive` | — | Archived project |
| PUT | `/projects/:id/unarchive` | — | Unarchived project |
| POST | `/projects/bulk-archive` | `{ projectIds: string[] }` | Bulk archive confirmation |
| POST | `/projects/bulk-unarchive` | `{ projectIds: string[] }` | Bulk unarchive confirmation |
| DELETE | `/projects/:id` | — | Success confirmation (only if archived) |

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

### Categories
| Method | Endpoint | Body | Returns |
|--------|----------|------|---------|
| GET | `/categories` | — (query: `?includeInactive=true`) | Array of categories |
| GET | `/categories/:id` | — | Single category |
| POST | `/categories` | `{ name, definition, keywords[], sortOrder? }` | Created category |
| PUT | `/categories/:id` | `{ name?, definition?, keywords?, sortOrder?, isActive? }` | Updated category |
| DELETE | `/categories/:id` | — | Success confirmation (soft delete) |
| PUT | `/categories/reorder` | `{ categoryOrders: [{id, sortOrder}] }` | Success confirmation |

**Note:** All category modifications automatically clear the Gemini cache to ensure fresh data.

### Export
| Method | Endpoint | Body | Returns |
|--------|----------|------|---------|
| POST | `/export/:projectId` | — | Google Sheet URL |

### Authentication
| Method | Endpoint | Body | Returns |
|--------|----------|------|---------|
| POST | `/auth/login` | `{ password: string }` | `{ token, expiresIn, expiresAt }` |
| GET | `/auth/verify` | — (requires Authorization header) | `{ valid: true, user: {...} }` |

---

## 🧪 Testing the Backend

Before building UI, test endpoints using:
- **Postman** or **Thunder Client** (VS Code extension)
- **cURL** commands
- Browser for GET requests

Example test:
```bash
# 1. Test login (get token)
curl -X POST http://localhost:8080/auth/login \
  -H "Content-Type: application/json" \
  -d '{"password":"your_app_password"}'

# 2. Test protected endpoint (use token from step 1)
curl -X POST http://localhost:8080/projects \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN_HERE" \
  -d '{"name":"Test Project","description":"Testing API"}'

# 3. List projects (requires token)
curl http://localhost:8080/projects \
  -H "Authorization: Bearer YOUR_TOKEN_HERE"

# 4. Test token verification
curl http://localhost:8080/auth/verify \
  -H "Authorization: Bearer YOUR_TOKEN_HERE"
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
- [ ] Implement login page with password input
- [ ] Store JWT token in localStorage/cookies
- [ ] Add Authorization header to all API requests
- [ ] Handle 401/403 errors with redirect to login
- [ ] Build Dashboard with project list (including archived status)
- [ ] Implement project creation modal
- [ ] Implement archive/unarchive project functionality
- [ ] Implement bulk archive/unarchive operations
- [ ] Build Project page with article table
- [ ] Implement import workflow with preview + progress
- [ ] Implement analysis workflow with batch creation + progress
- [ ] Add quotes tab
- [ ] Add category management page (Settings → Categories tab)
- [ ] Implement category CRUD (create, edit, delete, reorder)
- [ ] Add export button
- [ ] Polish UI (loading states, errors, toasts)
- [ ] Test all workflows end-to-end

---

**Welcome to NewsHub frontend development. Build something great! 🎉**
