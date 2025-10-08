🧾 NewsHub — Definitions (v0.1)

A living lexicon describing the structural, conceptual, and functional elements of the NewsHub project.
This file ensures consistent terminology across code, prompts, and UI development.

🔧 Project Overview

Project Name: NewsHub
Purpose: A webapp that searches, catalogues, and analyses news articles using external APIs (NewsAPI, Gemini) and stores results in a local database (PostgreSQL via Prisma).
Primary Use Case: Provide structured insights into media coverage by summarising and classifying articles, identifying key stakeholders, and exporting data for external analysis (e.g., Google Sheets).

📚 Core Data Structure
Project

A self-contained workspace containing a collection of Articles and their associated Quotes.

Fields:

id – unique UUID identifier

name – project name

description – optional description

createdAt, updatedAt – timestamps

Relationships: has many Articles

Article

A single news item imported from NewsAPI or added manually.
Holds both the raw metadata and the analysed data returned by Gemini.

Core Fields:

id – unique UUID

projectId – foreign key to Project

newsOutlet – publication or source

title – headline of the article

authors – array of author names

url – original article link

fullBodyText – full article content

dateWritten – date of publication

inputMethod – enum: newsapi, manual, csv

Gemini Analysis Fields:

summaryGemini – short summarisation by Gemini

categoryGemini – thematic category assigned by Gemini

sentimentGemini – sentiment classification (enum: positive, neutral, negative)

translatedGemini – boolean; whether translation occurred

analysedAt – timestamp marking analysis completion

NewsAPI.ai Extended Fields:

sourceUri – NewsAPI.ai source identifier

concepts – JSON array of AI-extracted concepts from NewsAPI.ai

categories – JSON array of article categories from NewsAPI.ai

sentiment – numeric sentiment score (-1 to 1) from NewsAPI.ai

imageUrl – article's featured image URL

location – JSON object with geographic/dateline data

importSessionId – foreign key linking to ImportSession

Relationships: has many Quotes, belongs to ImportSession

Quote

A statement extracted from an Article by Gemini, attributed to an identified stakeholder.

Fields:

id – unique UUID

articleId – foreign key to Article

stakeholderNameGemini – person or entity name

stakeholderAffiliationGemini – organisational or contextual link

quoteGemini – extracted statement text

ImportSession

Tracks a single article import operation from NewsAPI.ai, allowing users to monitor progress and review import history.

Fields:

id – unique UUID

projectId – foreign key to Project

searchTerms – array of keywords used in search

sources – array of source URIs selected for this import

startDate – beginning of date range filter

endDate – end of date range filter

articlesFound – total count returned by NewsAPI.ai

articlesImported – count successfully saved to database

status – enum: running, completed, failed

createdAt – timestamp when import started

completedAt – timestamp when import finished (nullable)

Relationships: belongs to Project, has many Articles

AnalysisBatch

Groups articles for batch processing through Gemini API, enabling efficient analysis of multiple articles simultaneously.

Fields:

id – unique UUID

projectId – foreign key to Project

status – enum: pending, running, completed, failed, cancelled

articleIds – array of article UUIDs being analyzed

totalArticles – count of articles in batch

processedArticles – count of articles successfully analyzed

createdAt – timestamp when batch was created

startedAt – timestamp when processing began (nullable)

completedAt – timestamp when processing finished (nullable)

error – error message if batch failed (nullable)

results – JSON object storing analysis results

Relationships: belongs to Project

SearchSource

Represents a news source available for article import from NewsAPI.ai, with regional and language metadata.

Fields:

id – unique UUID

title – display name of the news source

region – geographic region (e.g., "North America")

country – country code or name

language – ISO language code

sourceUri – NewsAPI.ai identifier (e.g., "bbc.co.uk")

isActive – boolean; whether source is available for selection

createdAt – timestamp when added to database

updatedAt – timestamp of last modification

Relationships: none (reference data)

🧠 Core Processes
Article Import (Session-based)

A multi-step workflow for importing news articles from NewsAPI.ai with progress tracking.

Steps:

1. **Preview**: User submits search parameters (keywords, sources, date range). Backend queries NewsAPI.ai to estimate article count and validate sources.

2. **Start Import**: Creates an ImportSession record with status "running". Process executes asynchronously, fetching articles in batches and saving to database.

3. **Monitor Progress**: Frontend polls session status endpoint to display real-time progress (articlesFound, articlesImported).

4. **Completion**: Session status updates to "completed" or "failed". Articles are linked to both Project and ImportSession for traceability.

Input: search terms, source URIs, date range, optional boolean query

Output: ImportSession ID, article counts, status updates

Source Management

A system for filtering and selecting news sources for imports.

Behaviour:

SearchSource table stores available news sources with metadata (region, country, language)

Frontend can query available sources, countries, and languages

Users select specific sources or use all active sources

Selected source URIs are passed to import requests

Gemini Analysis (Batch-based)

A structured process where articles are grouped into AnalysisBatch records and processed through Gemini API for summarisation, categorisation, sentiment, and quote extraction.

Workflow:

1. **Create Batch**: User selects up to 10 articles. System creates AnalysisBatch with status "pending".

2. **Start Processing**: Backend updates status to "running", fetches article content, sends to Gemini.

3. **Dual Analysis**: Each batch performs both article analysis (summary, category, sentiment) and quote extraction in sequence.

4. **Database Update**: Results are parsed and saved to Article records (summaryGemini, categoryGemini, sentimentGemini) and Quote records.

5. **Completion**: Batch status updates to "completed". Progress tracked via processedArticles / totalArticles.

Input: project ID, array of article IDs (max 10)

Output: AnalysisBatch ID, status updates, analysis results

Behaviour:

Runs on user request for selected articles

Executes asynchronously with progress tracking

Can be cancelled mid-process

Multiple batches can be created for large projects

Export

Converts the full Project dataset (Articles + Quotes) into a Google Sheets file.
Creates two tabs: Articles and Quotes.
Handles authentication via Google OAuth.

Note: Implementation exists but requires verification of Google Sheets API integration.

🖥️ User Interface Definitions
Dashboard

Landing screen displaying all Projects.

Create / delete Projects

Open an existing Project

Project Page

Workspace showing all articles belonging to a Project.

Displays article title, author, date, sentiment, and category

Expandable rows reveal full body text and summary

Multi-select to run Gemini analysis

Tabs for Articles and Quotes

Option to export to Google Sheets

Settings Dialogue

Allows users to edit:

Custom Gemini prompt fragments (for analysis style)

Category definitions

Export preferences
Admin-only fields:

Gemini system formatting schema (“System I/O Contract”)

Database field mapping

⚙️ Supporting Concepts

Input Method
Describes how an article entered the system. Can be newsapi, manual, or csv.

Analysis Status
Whether an article has been analysed (analysedAt not null).

Import Session
A tracked workflow instance representing a single article import operation from NewsAPI.ai, allowing users to monitor progress and review import history.

Analysis Batch
A group of up to 10 articles processed together through Gemini API for efficiency. Tracks progress and stores results.

Source URI
A unique identifier for a news source in NewsAPI.ai format (e.g., "bbc.co.uk", "nytimes.com").

Boolean Query
Advanced search syntax for NewsAPI.ai allowing complex keyword combinations (AND, OR, NOT operators).

Session-based Import
Import workflow pattern where operations are tracked via ImportSession records, enabling async processing with progress monitoring.

Batch Processing
Analysis workflow where multiple articles (max 10) are sent to Gemini in a single API request for efficiency and cost optimization.

System I/O Contract
An admin-only specification defining the strict JSON schema Gemini must follow when returning analysed data for insertion into the database. Defined in prompt templates (article-analysis.md, quote-analysis.md).

🧩 Planned Integrations

NewsAPI: Retrieve article metadata and body text.

Gemini API: Run structured NLP analysis (summaries, sentiment, stakeholder extraction).

Google Sheets API: Export completed project data to Sheets with two tabs (Articles, Quotes).

🧱 Stack Overview

Server: Express (TypeScript)

Database: PostgreSQL (Neon)

ORM: Prisma

Hosting: Render

Analysis: Gemini API

Search: NewsAPI

Export: Google Sheets API

🚧 Future Reserved Terms

(to be defined later)

“Batch Job” — internal process of sequential Gemini calls

“Prompt Schema” — editable portion of Gemini request

“Category Set” — controlled list of article topics, user-editable in settings