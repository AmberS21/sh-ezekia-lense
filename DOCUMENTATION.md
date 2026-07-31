# SH Ezekia Lens — Technical Documentation

_Last updated: 31 July 2026_

## 1. Overview

SH Ezekia Lens ('the tool') is an internal Sheffield Haworth productivity layer that sits on top of Ezekia (the firm's executive search / CRM platform). It gives consultants a fast, spreadsheet-like view of a project's candidate pipeline, with inline editing, tagging, filtering, bulk actions, AI-assisted summaries, and export — all without leaving the browser or waiting on Ezekia's native UI.

It is not a replacement for Ezekia. It reads candidate data from Ezekia's API (via a small proxy), lets a consultant work quickly across many records, and writes selected edits (progress notes, pipeline tag, gender, fit-to-profile, person tag) back to Ezekia when the consultant chooses to 'push' them.

The tool also contains a second, independent mode called Bulk Finder, which matches a pasted or uploaded list of names against people already in an Ezekia project and helps push matches into that project.

## 2. Architecture

The entire application is a single self-contained HTML file. There is no build step, no framework (no React/Vue/Angular), and no backend of its own — everything is vanilla HTML, CSS and JavaScript in one file, loaded directly by the browser.

Rough composition of the production file (sh_ezekia_lense.html):

- ~5,190 lines total
- One `<style>` block (~64.5 KB of CSS) — layout, theme, components
- One `<script>` block (~170 KB of JS) — all application logic, roughly 155 functions
- Two external dependencies: SheetJS/xlsx (loaded from cdnjs) for Excel export, and a direct call to the Anthropic API for AI Insights. Real Ezekia data calls go through a small serverless proxy (see Section 5)

Because it's one file with no bundler, 'deploying' a change simply means editing that file and committing it — there is no compile/build/package step.

## 3. Environments: Staging vs Production, and Deployment

The repository AmberS21/sh-ezekia-lense (branch: main) hosts two copies of the tool:

- Production: `/sh_ezekia_lense.html` — the live tool consultants use
- Staging: `/staging/sh_ezekia_lense.html` — an identical copy used to build and test changes before they reach Production

Both are published automatically by GitHub Pages from the main branch:

- Production: https://ambers21.github.io/sh-ezekia-lense/sh_ezekia_lense.html
- Staging: https://ambers21.github.io/sh-ezekia-lense/staging/sh_ezekia_lense.html

Working practice has been: build and fully test a change on Staging first; only copy it into Production once it is confirmed working and the change owner has explicitly said to go ahead. This avoids ever breaking the live tool while iterating on a feature.

There is no separate 'deploy' step — committing a change to either file on the main branch is the deployment. GitHub Pages typically republishes within well under a minute. Because browsers aggressively cache static files, a hard refresh or a cache-busting query string (e.g. adding `?v=2` to the URL) is often needed to see a change immediately after a commit.

## 4. Data Model

Every candidate row is a plain JavaScript object. The columns available in the table are defined centrally in one array (`COLUMNS`, near the top of the script), which drives the column manager, table rendering, filters, export and Advance Search all from one source of truth. The current columns are: Name, Pipeline Tag (status), Stage, Current Title, Current Company, Location, Nationality, Gender, Compensation, Currency, Type, Consultant, Rating, Source, Availability, Interview Date, Added Date, Last Updated, Progress Notes, Summary, Industry, Fit to Profile, and Person Tag.

Each column definition can carry flags: `always` (cannot be hidden — used for Name), `badge` (rendered as a coloured pill — used for Pipeline Tag, Stage, Person Tag), and `editable` (can be edited inline and pushed back to Ezekia — used for Pipeline Tag, Gender, Progress Notes, Summary, Fit to Profile, Person Tag).

Several arrays hold global application state, the most important being: `ALL` (every loaded candidate row), `FILTERED` (the subset currently passing all active filters/search — this is what actually gets rendered/paginated/exported), `VISIBLE` (the Set of column keys currently shown), `COLUMN_ORDER` (current left-to-right order of columns), `PENDING` (a Map of unsaved inline edits waiting to be pushed to Ezekia), and `PERSON_TAGS` / `_GLOBAL_TAGS` (the master lists of tag definitions and colours).

## 5. Connecting to Live Ezekia Data

A consultant pastes an Ezekia project/assignment/opportunity/list URL into the connection box. `parseEzekiaUrl()` extracts the entity type (project, assignment, opportunity or list) and its numeric ID from that URL using a regular expression. `connectFromUrl()` then drives the load: it fetches the list of people on that entity and, for each candidate, calls `enrichOne()` to pull the full person record and any additional-info fields (progress notes, fit-to-profile, etc.) from Ezekia, batching requests two at a time with a short delay between batches to stay within API limits. Results are cached in IndexedDB (via `_openCacheDB`/`cacheGet`/`cachePut`/`cacheDelete`) so re-opening the same project is fast and can work from cache if the network is slow.

All calls to Ezekia go through a fixed proxy base URL (a Val Town serverless endpoint) rather than hitting Ezekia's API directly from the browser; the app automatically attaches an access-key header to any request sent to that proxy via a small wrapper around `window.fetch`. This proxy exists to handle CORS and centralise the Ezekia credentials — it is not something a consultant needs to configure per-use.

If no project URL is supplied, the tool falls back to `generateDemo()`/`loadDemo()`, which fabricates realistic-looking sample candidates so the UI can be explored, demoed, or used for training without touching a real project.

Editable fields are never written to Ezekia automatically. Editing a cell marks the row 'dirty' (`markRowDirty`/`updateDirtyBadge`) and queues the change in `PENDING`; the consultant reviews pending changes and confirms them via the push modal (`openPushModal` → `executePush`), which sends the appropriate requests to Ezekia's API per field (progress notes and fit-to-profile use the additional-info endpoint; pipeline tag, gender and person tag use their own respective endpoints).

## 6. Core Features

### 6.1 Table, Sorting, Pagination, Columns
The main table (`renderTable`/`renderPagination`) supports click-to-sort per column (`setSort`), configurable rows-per-page (`changePerPage`), and a column manager (`buildColToggles`) that lets a consultant show/hide and reorder columns; the chosen layout is remembered per saved view.

### 6.2 Inline Editing and Pushing Changes
Editable cells (`onEditFocus`/`onEditInput`/`onEditKeydown`/`onEditBlur`) can be typed into directly in the grid. Edited rows are flagged dirty and tracked in `PENDING` until explicitly pushed to Ezekia through the push modal, with per-row undo available (`undoChange`) before a push.

### 6.3 Tagging
Pipeline Tag (status), Person Tag, Gender and Fit-to-Profile are all colour-coded pill/badge fields with their own dropdown selectors (`toggleTagDropdown`/`selectPipelineTag`, `togglePersonTagDropdown`/`addPersonTag`/`removePersonTag`, `toggleGenderDropdown`/`selectGender`, `toggleFitDropdown`/`selectFitToProfile`). Colours are assigned consistently per tag value via `tagColor`/`personTagColor`/`fitColor`.

### 6.4 Filtering
There are three complementary filtering mechanisms:

- Per-column filter dropdowns (`openColFilter`/`buildFilterOptions`/`filterColDropSearch`/`clearColFilter`) — click a column header's filter icon to tick/untick specific values, with a small search box to narrow long value lists.
- Multi-select quick filters (`toggleMsDropdown`/`toggleMsOption`/`removeMsFilter`) shown as removable tags above the table.
- Advance Search — a small draggable panel (`toggleAdvSearchPanel`) reachable via the 'Advance Search' button beside the main search bar. It combines free-text fields (Name, Location, Current Title, Current Company, Progress Notes, Summary, Industry) with dropdown pickers for Pipeline Tag, Gender and Person Tag (`populateAdvSearchDropdowns`). Applying it (`applyAdvSearch`) narrows `FILTERED` via `applyFilters`, which is exact-match for status/gender/person_tag and substring-match for the free-text fields. When any Advance Search condition is active, the button turns into an 'on' state with a small badge (`updateAdvSearchIndicator`) so it's obvious the visible list is being narrowed; loading a new project or demo data automatically clears Advance Search (`clearAdvSearch`) so stale conditions from a previous project can't silently hide candidates.

All three filtering layers, plus the plain text search box, combine together (`clearFilters` resets all of them at once).

### 6.5 Saved Views and Recent Projects
A consultant can save the current combination of filters, column layout, sort and search as a named view (`saveCurrentView`/`persistSavedViews`, stored in `localStorage`), reapply it later (`applySavedView`), or delete it (`deleteSavedView`). The tool also remembers the last few projects opened (`saveRecentProject`/`renderRecentProjects`) for one-click reopening, and auto-restores the last view/session on reload (`autoSaveLastView`/`restoreLastView`).

### 6.6 Bulk Edit
Selecting multiple rows (`toggleSelectAll`/`updateSelection`) and opening Bulk Edit (`openBulkEditModal`) lets a consultant set Pipeline Tag, Person Tags or Industry codes across every selected candidate at once (`applyBulkEdit`, with supporting pickers `bulkToggleField`/`bulkToggleSetMember`/`_bulkLoadIndustryOptions`/`bulkFilterIndustryList`).

### 6.7 AI Insights
The 'AI Insights' modal (`openAI` for a single candidate, `openBulkAI` for the current filtered list) builds a prompt from the relevant candidate data and sends it to Anthropic's Claude API (`sendAI`, calling `api.anthropic.com/v1/messages` with model `claude-sonnet-4-20250514`) to produce a natural-language summary or answer a free-text question about the pipeline. Note: the request is currently sent without an API key attached, so this call will fail in a live browser as-is — the in-app fallback message says this deliberately ('In a deployed version this uses the Anthropic API key configured in Admin'). Wiring up a working key/proxy for this feature is a known outstanding item, not a bug to fix by editing the prompt logic.

### 6.8 Export
The Export modal (`openExport`/`selectExport`/`runExport`) writes the currently filtered (or all) rows, restricted to the currently visible columns and in their current order, to either an `.xlsx` file (via the SheetJS library) or CSV.

### 6.9 Usage Log and Admin Panel
Every meaningful action can be recorded to an in-memory usage log (`logUsage`/`renderUsageLogTable`), viewable and exportable from the Admin panel (`openAdmin`/`loadUsageDashboard`/`exportUsageLog`/`clearUsageLog`). The Admin panel also stores a display name per user, and a couple of low-level config values (proxy URL, environment) in `localStorage` under `sh.ezekia.config` / `sh.ezekia.user` (`saveAdminConfig`/`getDisplayName`/`ensureUserIdentified`).

### 6.10 Bulk Finder
Bulk Finder is a self-contained secondary mode of the same page (all its functions and variables are prefixed `BF_` and are intentionally kept independent from the main table's code, so changes to the main tool should never break it and vice versa). A consultant pastes or uploads (drag-and-drop or file picker) a list of candidate names (`BF_onDrop`/`BF_onFileSelect`/`BF_loadFile`/`BF_parseInput`), the tool fuzzy-matches each name against the people already in a connected Ezekia project (`BF_fuzzyScore`/`BF_score`/`BF_runMatch`/`BF_nameLadder`), shows a confidence/recommended-action per row (`BF_statusFromConf`/`BF_recommendedAction`) in its own results table (`BF_renderTable`/`BF_sortBy`/`BF_openColFilter` and related functions), and can push confirmed matches into the project (`BF_pushToProject`) or export the match results (`BF_exportExcel`).

### 6.11 Theming
A light/dark theme toggle (`toggleTheme`/`setTheme`/`initTheme`) switches a `data-theme` attribute on the page; all colours are defined once as CSS custom properties under `:root`, so most visual changes only require editing those variables rather than hunting through individual rules. The Sheffield Haworth brand palette (navy background, #0e83df sky-blue accent) is the default/dark theme.

### 6.12 Caching
Loaded project data is cached in the browser's IndexedDB (a small wrapper: `_openCacheDB`/`cacheGet`/`cachePut`/`cacheDelete`, with a small badge shown via `_showCacheBadge`) so reopening a recently viewed project is fast and has an offline fallback if Ezekia's API is briefly unreachable. Cache entries are capped (`CACHE_MAX`) and keyed by project URL.

## 7. Where Things Live in the Code

Reading top to bottom, the file is organised as: `<style>` block (theme variables first, then component styles) → HTML body, itself commented into clear sections (BF:LANDING, SIDEBAR, CONNECTION, RECENT PROJECTS, SAVED VIEWS, STATS, COLUMN MANAGER, ADMIN FOOTER, MAIN, TOPBAR, ACTIVE FILTER TAGS, TABLE, FOOTER/PAGINATION, API/PROXY CONFIG, MS 365 IDENTITY placeholder, USAGE LOG, AI MODAL, BULK EDIT MODAL, EXPORT MODAL) → one `<script>` block containing all application logic. Searching the file for these HTML comment markers, or for a specific function name, is the fastest way to find the right place to edit.

## 8. How to Modify, Fix, Update or Upgrade This Tool

### 8.1 Recommended workflow
Always change Staging first (`/staging/sh_ezekia_lense.html`), reload the Staging URL (with a cache-busting `?v=` query string if needed) and test the change against both demo data and a real connected project. Only once it behaves correctly should the same change be copied into Production (`/sh_ezekia_lense.html`) and re-tested there — Production-only regressions can occur even when Staging looks fine, simply because the surrounding code isn't byte-identical between the two files, so always re-verify after copying rather than assuming success.

### 8.2 Editing mechanics
Both files are edited directly through GitHub's web file editor (or any Git workflow) — there's no local dev server required, though one can be run trivially by opening the HTML file directly in a browser or serving the folder with any static file server. Because the tool is a single file, most edits touch up to three places for a given feature: the CSS in `<style>`, the HTML markup in `<body>`, and one or more functions in `<script>`.

### 8.3 Common pitfalls learned from past fixes
When copying an HTML block from one file to another, always count opening and closing tags (`<div>` vs `</div>`) in the copied snippet before inserting it — a missing closing tag produces no JavaScript error at all, but silently reparents everything after it in the DOM, breaking layout in ways that are only visible by looking at the rendered page (not by checking console errors or filtered-data counts). When adding a new filter or comparison on a tag/status/gender-style field, use exact matching rather than substring matching — a naive substring check on gender would incorrectly match 'male' inside 'female'. When editing Bulk Finder or the main table, double-check the other one still works afterwards, since both share the same page and script block even though their functions are separately namespaced (`BF_` prefix for Bulk Finder).

### 8.4 Before calling a change 'done'
It's worth re-checking, after any edit: the browser console for new errors, that the page's overall layout/structure still looks correct (not just that data loads), that Bulk Finder still works independently of whatever was changed in the main tool (and vice versa), and that a hard refresh / cache-busted URL is used so the newly deployed version is actually what's being viewed, not a cached copy.

## 9. External Services and Dependencies

- Ezekia API — the source of all real candidate data, accessed only through the Val Town proxy described in Section 5, not called directly from the browser.
- Val Town proxy (the `BASE` constant) — handles the Ezekia connection and access-key header injection.
- Anthropic Claude API — used for AI Insights; currently called without an attached key (see 6.7).
- SheetJS (`xlsx.full.min.js`, loaded from cdnjs) — used only for `.xlsx` export.
- Browser storage — `localStorage` for theme, saved views, recent projects, user identity and small config; IndexedDB for cached project data.

## 10. Known Notes / Open Items

- AI Insights (6.7) needs a working Anthropic API key/proxy wired in before it will return real responses; right now it shows a fallback 'could not reach AI' message.
- The Admin panel's proxy URL/environment fields are present in the UI, but the app is currently hard-wired to the single Val Town `BASE` URL for all Ezekia calls.
- Staging and Production are not byte-identical files (they have diverged in minor ways over time even though they are functionally equivalent for shipped features), so line numbers for the 'same' code will differ between the two — always search by function/comment name, not by line number, when working across both.
