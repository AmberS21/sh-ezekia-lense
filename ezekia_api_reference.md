# Ezekia API Reference — Sheffield Haworth DocGen

## Setup
- **Base URL:** `https://ezekia.com/api`
- **Proxy URL:** `/api/proxy/ezekia` (Azure Functions backend)
- **Auth:** API key passed via proxy — never exposed to frontend
- **Frontend calls proxy:** `fetch('/api/proxy/ezekia/v4/...')`

---

## Key Endpoints Used

### 1. Fetch Assignment / Project Details
```
GET /v4/projects/{id}
```
Returns: project name, client, lead consultant, researcher, status, practice, geography, date created

**URL pattern:** `https://ezekia.com/api/v4/projects/{id}`

---

### 2. Fetch Candidates for a Project
```
GET /v4/projects/{id}/candidates?fields[]=meta.candidate
```
Returns array of candidates with:
- `id` — person ID (use for /v4/people/{id} call)
- `name`, `firstName`, `lastName`, `fullName`
- `meta.candidate.pipelineTags[]` — pipeline stage tags
- `meta.candidate.addedAt`, `meta.candidate.addedBy`
- `owner.tags` — fallback tags

**Pipeline Tag extraction (CRITICAL):**
```javascript
// Store on _bc immediately before any async calls
_bc._resolvedPipelineTag = '';
if (_bc.meta && _bc.meta.candidate && _bc.meta.candidate.pipelineTags) {
  var tags = _bc.meta.candidate.pipelineTags.slice().sort(function(a,b){
    return new Date(b.addedAt||0) - new Date(a.addedAt||0);
  });
  if (tags.length > 0) _bc._resolvedPipelineTag = tags[0].text || '';
}
// Later in .map() use: cand._resolvedPipelineTag (NOT _bc — stale loop var!)
```

---

### 3. Fetch Full Person Profile
```
GET /v4/people/{id}?fieldsWithCandidate[]=manager.tags&fieldsWithCandidate[]=manager.customValues&fieldsWithCandidate[]=manager.researchNotes&fields[]=meta.candidate
```
Returns: full profile including positions, addresses, emails, phones

Key fields:
- `profile.positions[]` — work history (company, title, dates)
- `addresses[]` — city, country, state
- `fullName`, `firstName`, `lastName`

---

### 4. Fetch Additional Info (Gender, Notes)
```
GET /people/{id}/additional-info
```
Returns custom field values including:
- Gender: field hashId `boYK`
- Progress notes: field hashId `bmwO`

```javascript
var genderField = additionalInfo.find(function(f){ return f.hashId === 'boYK'; });
var notesField  = additionalInfo.find(function(f){ return f.hashId === 'bmwO'; });
```

---

### 5. Fetch Opportunities
```
POST /v4/opportunities/search
Body: { "ids": [id] }
```
Different from assignments — uses POST not GET.

### 6. Fetch Lists
```
GET /v4/lists/{id}/candidates
```

---

## URL Parsing (from Ezekia browser URL)
```javascript
// Ezekia URLs look like:
// https://app.ezekia.com/assignments/12345/pipeline
// https://app.ezekia.com/projects/67890/overview

var match = url.match(/\/(assignments?|projects?|opportunities?|lists?)\/(\d+)/i);
var entityType = match[1].toLowerCase(); // 'assignment', 'project', etc.
var entityId   = match[2];              // '12345'
```

---

## Proxy Setup (backend/server.js)
```javascript
// All Ezekia requests go through /api/proxy/ezekia
app.all('/api/proxy/ezekia/*', async (req, res) => {
  const ezekiaPath = req.params[0];
  const ezekiaUrl  = `https://ezekia.com/api/${ezekiaPath}`;
  // Adds API key header, forwards request, returns response
});
```

**Frontend usage:**
```javascript
const BASE = '/api/proxy/ezekia';
const resp = await fetch(`${BASE}/v4/projects/${id}/candidates?fields[]=meta.candidate`);
const data = await resp.json();
const candidates = data.data || [];
```

---

## Entity Types & Supported Endpoints
| Entity | Endpoint | Status |
|---|---|---|
| Assignment | GET /v4/projects/{id} + /candidates | ✅ Working |
| Project | GET /v4/projects/{id} + /candidates | ✅ Working |
| Opportunity | POST /v4/opportunities/search | ⚠️ Phase 2 |
| List | GET /v4/lists/{id}/candidates | ⚠️ Phase 2 |

---

## Rate Limiting & Error Handling
- Retry logic: 3 attempts with backoff on 429
- Concurrent users: template caching (`_tplCache`) prevents duplicate fetches
- Always check `data.data || []` — response wraps array in `data` key

---

## Notes
- API key stored as Azure env var `EZEKIA_API_KEY` — never in frontend
- All requests proxied through `/api/proxy/ezekia` to avoid CORS
- `fields[]=meta.candidate` must be separate from `fieldsWithCandidate[]` params
- Pipeline tags sorted by `addedAt` DESC to get most recently applied tag
