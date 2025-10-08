# 🗂️ NewsHub Documentation Index

Welcome to the NewsHub project documentation. This index provides a comprehensive overview of all documentation files and their purposes.

## 📋 Core Documentation

| File | Purpose |
|------|---------|
| [`definitions.md`](./definitions.md) | Global terminology and data model definitions for Projects, Articles, Quotes, and core processes |
| [`manifest.md`](./manifest.md) | Repository structure map and component purpose guide |
| [`instructions.md`](./instructions.md) | System architecture, development conventions, and phase tracking |

## 🔌 Integration Documentation

| File | Purpose |
|------|---------|
| [`NewsAPI.ai schema and examples.md`](./NewsAPI.ai%20schema%20and%20examples.md) | Schema reference and example payloads for NewsAPI.ai integration |

## 🎨 Frontend Documentation

| File | Purpose |
|------|---------|
| [`style-guidelines.md`](./style-guidelines.md) | UI styling and design conventions using TailwindCSS and Shadcn/UI |

## 📁 Documentation Folders

### `/prompts/` - Gemini AI Prompts
| File | Purpose |
|------|---------|
| [`article-analysis.md`](./prompts/article-analysis.md) | Master prompt for article analysis (categories, sentiment, summaries) |
| [`category-definitions.md`](./prompts/category-definitions.md) | Category definitions and keywords for content classification |
| [`quote-analysis.md`](./prompts/quote-analysis.md) | Master prompt for stakeholder and quote extraction |

### `/UI/` - User Interface Specifications
| File | Purpose |
|------|---------|
| [`dashboard.md`](./UI/dashboard.md) | Dashboard page layout, components, and API interactions |
| [`project.md`](./UI/project.md) | Project page layout with Articles, Quotes, and Settings tabs |
| [`settings.md`](./UI/settings.md) | Global settings page with Integrations, Gemini, Exports, and UI/Preferences tabs |

## 🚀 Quick Navigation

### For Developers
- **Getting Started**: [`instructions.md`](./instructions.md) - System architecture and development setup
- **Data Models**: [`definitions.md`](./definitions.md) - Core terminology and data structures
- **File Structure**: [`manifest.md`](./manifest.md) - Repository organization

### For Frontend Development
- **Design System**: [`style-guidelines.md`](./style-guidelines.md) - UI styling and component patterns
- **Page Layouts**: [`/UI/`](./UI/) - Detailed specifications for all major pages
- **Component Architecture**: [`/UI/dashboard.md`](./UI/dashboard.md), [`/UI/project.md`](./UI/project.md), [`/UI/settings.md`](./UI/settings.md)

### For AI Integration
- **Analysis Prompts**: [`/prompts/`](./prompts/) - Gemini AI prompt templates
- **API Integration**: [`NewsAPI.ai schema and examples.md`](./NewsAPI.ai%20schema%20and%20examples.md) - External API documentation

## 📊 Project Status

Current development phase and completion status can be found in [`instructions.md`](./instructions.md).

**Latest Updates:**
- ✅ Phase 1: Express + Prisma base
- ✅ Phase 2: CRUD endpoints  
- ✅ Phase 3: NewsAPI integration
- ✅ Phase 4: Gemini analysis + batching queue
- 🔜 Phase 5: Google Sheets export

---

*This index is automatically maintained. Please update when adding new documentation files.*
