# WF5: Enrichment + Social Scoring — Detailed Workflow Architecture

## 1. Purpose

This workflow takes vetted, tiered prospects from Tab 5 and produces fully enriched dossiers in Tab 6. For each prospect, it resolves the right contact person, finds their verified email, discovers their X handle, pulls social activity from up to three sources, scores the best personalization point, and (for Guest Post prospects) gathers publication context.

This is the most complex workflow in the system. It orchestrates 6 different APIs and adjusts its behavior based on the prospect's tier (A, B, or C) and workflow type (Linkable or Guest Post). Consider splitting it into sub-workflows in n8n for maintainability.

---

## 2. The 6 APIs Used

| API | Purpose | When Called | Cost Per Call |
|-----|---------|------------|---------------|
| **Exa** | Editor identification, recent web articles by contact, X handle fallback | Always | ~$0.007-0.022 |
| **Prospeo** | Primary email verification | Always | 1 credit (~$0.025) per match |
| **EnrichLayer** | Fallback email, X handle discovery, LinkedIn URL lookup | When Prospeo fails or X handle needed | 1-5 credits (~$0.017-0.084) |
| **Grok API (xAI)** | Recent X/Twitter posts by contact | When X handle is available | Token-based (fractions of a cent) |
| **LinkUpAPI** | LinkedIn activity data | Tier A only, when Sources 1+2 are thin | 1 credit (~$0.058) |
| **Linkup.so** | Publication context for Guest Post prospects | Guest Post workflow only | €0.005-0.05 |

---

## 3. Inputs & Outputs

### Inputs
- **Tab 5 (Vetted Prospects):** Rows where Status = "Approved for Enrichment"
  - Fields: Prospect ID, Workflow Type, Domain, Tier (A/B/C)
  - For Linkable prospects: Author Name already available from Tab 3 (via WF2)
  - For Guest Post prospects: Publication Name and Sample Article URL from Tab 4 (via WF3)
- **Tab 1 (Config):** Source Magnet Title, Target Industry Keywords
- **Tab 3 (Raw Linkable):** Author Name, Article URL, Article Title (for Linkable prospects)
- **Tab 4 (Raw Guest Post):** Publication Name, Domain (for Guest Post prospects)

### Outputs
- **Tab 6 (Enriched Prospects):** One row per enriched prospect with full dossier

---

## 4. Complete Node Map

```
┌───────────────────────────────────────┐
│  1. Manual Trigger                    │
└──────────────┬────────────────────────┘
               │
┌──────────────▼────────────────────────┐
│  2. Read Config + Vetted Prospects    │
└──────────────┬────────────────────────┘
               │
┌──────────────▼────────────────────────┐
│  3. Loop: Each Prospect               │
└──────────────┬────────────────────────┘
               │
┌──────────────▼────────────────────────┐
│  4. BRANCH: Linkable vs Guest Post    │
│     (determines contact resolution)   │
└──────┬───────────────┬────────────────┘
       │ Linkable      │ Guest Post
       │               │
┌──────▼──────┐ ┌──────▼───────────────┐
│ 5a. Pull    │ │ 5b. Find Editor      │
│ Author from │ │ via Exa or Linkup    │
│ Tab 3       │ │                      │
└──────┬──────┘ └──────┬───────────────┘
       │               │
       └───────┬───────┘
               │ (contact name + domain known)
┌──────────────▼────────────────────────┐
│  6. SUB-WF: Email Resolution          │
│     Prospeo → EnrichLayer fallback    │
└──────────────┬────────────────────────┘
               │
┌──────────────▼────────────────────────┐
│  7. SUB-WF: X Handle Resolution       │
│     EnrichLayer → Exa fallback        │
└──────────────┬────────────────────────┘
               │
┌──────────────▼────────────────────────┐
│  8. SUB-WF: Parallel Social Enrich    │
│     Source 1: Exa (always)            │
│     Source 2: Grok (if handle)        │
│     Source 3: LinkUpAPI (Tier A only) │
└──────────────┬────────────────────────┘
               │
┌──────────────▼────────────────────────┐
│  9. LLM: Social Context Scorer        │
└──────────────┬────────────────────────┘
               │
┌──────────────▼────────────────────────┐
│  10. IF: Guest Post?                  │
│      → Yes: Linkup.so pub context     │
│      → No: Skip                       │
└──────────────┬────────────────────────┘
               │
┌──────────────▼────────────────────────┐
│  11. Write to Tab 6                   │
└───────────────────────────────────────┘
```

---

## 5. Node-by-Node Specification

### Node 1: Manual Trigger
- **n8n node type:** Manual Trigger

### Node 2: Read Config + Vetted Prospects
- **n8n node type:** Two Google Sheets → Read Rows nodes

**Read Config (Tab 1):** Extract `source_magnet_title`, `target_industry_keywords`

**Read Vetted Prospects (Tab 5):** Filter Status = "Approved for Enrichment"
- Extract: Prospect ID, Workflow Type, Domain, Tier, and the linked data from Tab 3 or Tab 4

**Implementation note:** For Linkable prospects, you'll also need to read their corresponding row from Tab 3 (matching on Prospect ID) to get the Author Name, Article URL, and Article Title. Use a Merge node to join Tab 5 data with Tab 3 data on Prospect ID.

### Node 3: Loop — Each Prospect
- **n8n node type:** Split In Batches
- **Batch Size:** 1
- **Wait between batches:** 2 seconds (multiple API calls per prospect)

### Node 4: Branch on Workflow Type
- **n8n node type:** IF
- **Condition:** `workflow_type` equals "Linkable"
- **TRUE path:** Node 5a (pull author from Tab 3)
- **FALSE path:** Node 5b (find editor via Exa/Linkup)

---

### Node 5a: Pull Author from Tab 3 (Linkable Only)
- **n8n node type:** Code (JavaScript)
- **Purpose:** The author was already captured by Exa during WF2. Just format it for downstream use.

```javascript
const prospect = $input.item.json;

// Author data should already be merged from Tab 3
const contactName = prospect.author_name || "Unknown";
const nameParts = contactName.split(' ');
const firstName = nameParts[0] || "";
const lastName = nameParts.slice(1).join(' ') || "";

return {
  json: {
    ...prospect,
    contact_name: contactName,
    contact_first_name: firstName,
    contact_last_name: lastName,
    contact_role: "Author",
    contact_source: "Exa (WF2 discovery)",
    contact_resolution_method: "author_from_discovery"
  }
};
```

---

### Node 5b: Find Editor (Guest Post Only)
- **n8n node type:** HTTP Request (Exa Search API)
- **Purpose:** Find the editor or Head of Content at the target publication.

**Exa Search Call:**
```json
{
  "query": "Editor OR Head of Content OR Managing Editor at {{$json.publication_name}}",
  "type": "auto",
  "numResults": 5,
  "includeDomains": ["linkedin.com", "{{$json.domain}}"],
  "contents": {
    "text": {
      "maxCharacters": 1000
    }
  }
}
```

**Cost:** ~$0.007 + 5 × $0.001 = ~$0.012

**Parse the results (Code node after Exa call):**

```javascript
const results = $input.item.json.results || [];
const prospect = $('Branch on Workflow Type').item.json;

// Try to find a person who appears to be an editor
let bestMatch = null;

for (const r of results) {
  const text = (r.text || "").toLowerCase();
  const title = (r.title || "").toLowerCase();
  const author = r.author || "";

  // Score based on editorial role signals
  let score = 0;
  if (text.includes('editor') || title.includes('editor')) score += 3;
  if (text.includes('head of content') || title.includes('head of content')) score += 3;
  if (text.includes('managing editor')) score += 3;
  if (text.includes('editorial director')) score += 3;
  if (text.includes('content manager')) score += 2;
  if (text.includes(prospect.domain)) score += 1;

  if (score > 0 && (!bestMatch || score > bestMatch.score)) {
    bestMatch = {
      name: author || "Unknown",
      score: score,
      source_url: r.url,
      context: (r.text || "").substring(0, 200)
    };
  }
}

if (!bestMatch) {
  // Fallback: use the publication's generic contact
  bestMatch = {
    name: "Editor",
    score: 0,
    source_url: "",
    context: "No specific editor found — flag for manual research"
  };
}

const nameParts = bestMatch.name.split(' ');
const firstName = nameParts[0] || "";
const lastName = nameParts.slice(1).join(' ') || "";

return {
  json: {
    ...prospect,
    contact_name: bestMatch.name,
    contact_first_name: firstName,
    contact_last_name: lastName,
    contact_role: "Editor",
    contact_source: `Exa search (score: ${bestMatch.score})`,
    contact_resolution_method: "editor_search",
    editor_context: bestMatch.context,
    editor_source_url: bestMatch.source_url,
    flag_for_manual: bestMatch.score === 0
  }
};
```

**If Exa doesn't find an editor (score = 0):** The prospect is flagged for manual research. You'll see `flag_for_manual: true` in Tab 6 and can look up the editor yourself.

**Alternative approach with Linkup.so (for higher-value targets):**

```json
{
  "q": "Who is the editor or head of content at [Publication Name]? What is their full name and job title?",
  "depth": "deep",
  "outputType": "structured",
  "structuredOutputSchema": "{\"type\":\"object\",\"properties\":{\"editor_name\":{\"type\":\"string\"},\"editor_title\":{\"type\":\"string\"},\"linkedin_url\":{\"type\":\"string\"}}}"
}
```

Cost: €0.05 (deep). Use this for Tier A Guest Post prospects where getting the right editor is critical.

---

### Node 6: SUB-WF — Email Resolution

**Step 6a: Prospeo Enrich Person (Primary)**
- **n8n node type:** HTTP Request
- **Method:** POST
- **URL:** `https://api.prospeo.io/enrich-person`
- **Headers:**
  - X-KEY: `[YOUR_PROSPEO_API_KEY]`
  - Content-Type: application/json
- **Body:**

```json
{
  "only_verified_email": true,
  "data": {
    "first_name": "{{$json.contact_first_name}}",
    "last_name": "{{$json.contact_last_name}}",
    "company_website": "{{$json.domain}}"
  }
}
```

**Validated API behavior (Prospeo V2):**
- Returns `error: false` and a `person` object with `email` field if found
- Returns `error: true` if no match — **no credit charged**
- `only_verified_email: true` ensures you only pay for deliverable emails
- Cost: 1 credit (~$0.025) per verified match
- Won't charge if you enrich the same person twice (lifetime dedup)
- Rate limit: 150 requests/minute (per Cargo docs)

**Parse Prospeo Response (Code node):**

```javascript
const response = $input.item.json;
const prospect = $('Loop — Each Prospect').item.json;

if (!response.error && response.person && response.person.email) {
  return {
    json: {
      ...prospect,
      verified_email: response.person.email,
      email_source: "Prospeo",
      email_verified: true,
      linkedin_url: response.person.linkedin_url || "",
      prospeo_person_id: response.person.person_id || ""
    }
  };
} else {
  // Prospeo failed — proceed to EnrichLayer fallback
  return {
    json: {
      ...prospect,
      verified_email: "",
      email_source: "pending_fallback",
      email_verified: false,
      linkedin_url: response.person?.linkedin_url || ""
    }
  };
}
```

**Step 6b: IF — Email Found?**
- **n8n node type:** IF
- **Condition:** `verified_email` is not empty
- **TRUE path:** Skip to Node 7 (email resolved)
- **FALSE path:** Continue to Step 6c (EnrichLayer fallback)

**Step 6c: EnrichLayer Work Email Lookup (Fallback)**
- **n8n node type:** HTTP Request (or the n8n-nodes-enrichlayer community node)
- **Method:** GET
- **URL:** `https://enrichlayer.com/api/v2/contact/work-email`
- **Headers:** Authorization: Bearer [YOUR_ENRICHLAYER_API_KEY]
- **Query Parameters:**
  - `first_name`: contact first name
  - `last_name`: contact last name
  - `company_domain`: prospect domain
  - `fallback_to_cache`: `on-error`

**Validated API behavior (EnrichLayer):**
- Cost: 3 credits (~$0.05) per lookup
- **Refunded if no email found**
- Returns the work email if available

**Parse EnrichLayer Response:**

```javascript
const response = $input.item.json;
const prospect = $input.item.json; // carried forward

const email = response.email || "";

return {
  json: {
    ...prospect,
    verified_email: email,
    email_source: email ? "EnrichLayer" : "Manual",
    email_verified: email ? true : false,
    flag_for_manual: email ? prospect.flag_for_manual : true
  }
};
```

---

### Node 7: SUB-WF — X Handle Resolution

**Step 7a: EnrichLayer Person Profile (for twitter_profile_id)**

This step serves a dual purpose: get the X handle AND get the LinkedIn URL (if Prospeo didn't return one).

- **n8n node type:** HTTP Request
- **Method:** GET
- **URL:** `https://enrichlayer.com/api/v2/profile`
- **Headers:** Authorization: Bearer [YOUR_ENRICHLAYER_API_KEY]
- **Query Parameters:**
  - `profile_url`: LinkedIn URL if known (from Prospeo response)
  - OR use the Person Lookup endpoint first if no LinkedIn URL
  - `twitter_profile_id`: `include`
  - `use_cache`: `if-present`
  - `fallback_to_cache`: `on-error`

**If no LinkedIn URL is available, use Person Lookup first:**
- **URL:** `https://enrichlayer.com/api/v2/profile/resolve`
- **Query Parameters:**
  - `first_name`: contact first name
  - `last_name`: contact last name
  - `company_domain`: prospect domain
  - `enrich_profile`: `enrich`
  - `twitter_profile_id`: `include`

**Validated cost breakdown:**
- Person Profile with `twitter_profile_id: include`: 1 (base) + 1 (twitter) = 2 credits (~$0.034)
- Person Lookup with enrich: 2 (lookup) + 1 (enrich) + 1 (twitter) = 4 credits (~$0.067)
- `use_cache: if-present` avoids the 1-credit freshness surcharge
- `live_fetch` is NOT used (saves 9 credits) — cached data is fine for X handle discovery

**Parse response:**

```javascript
const response = $input.item.json;
const prospect = $input.item.json;

const twitterId = response.twitter_profile_id || "";
const linkedinUrl = response.linkedin_profile_url || prospect.linkedin_url || "";

let xHandle = "";
if (twitterId) {
  xHandle = twitterId; // EnrichLayer returns the handle directly
}

return {
  json: {
    ...prospect,
    x_handle: xHandle,
    x_handle_source: xHandle ? "EnrichLayer" : "pending_fallback",
    linkedin_url: linkedinUrl
  }
};
```

**Step 7b: IF — X Handle Found?**
- **TRUE path:** Skip to Node 8
- **FALSE path:** Continue to Exa fallback

**Step 7c: Exa X Handle Fallback**
- **n8n node type:** HTTP Request

```json
{
  "query": "{{$json.contact_name}} {{$json.publication_name}}",
  "type": "auto",
  "numResults": 3,
  "includeDomains": ["x.com", "twitter.com"],
  "contents": {
    "text": false
  }
}
```

**Parse X handle from URL:**

```javascript
const results = $input.item.json.results || [];
const prospect = $input.item.json;

let xHandle = "";
let xHandleUrl = "";

for (const r of results) {
  try {
    const url = new URL(r.url);
    const pathParts = url.pathname.split('/').filter(p => p.length > 0);
    if (pathParts.length >= 1) {
      const candidate = pathParts[0];
      // Skip non-profile paths
      if (!['search', 'explore', 'settings', 'home', 'i', 'hashtag'].includes(candidate)) {
        xHandle = candidate;
        xHandleUrl = r.url;
        break;
      }
    }
  } catch { continue; }
}

return {
  json: {
    ...prospect,
    x_handle: xHandle || prospect.x_handle,
    x_handle_source: xHandle ? "Exa search" : "Not found",
    x_handle_url: xHandleUrl
  }
};
```

---

### Node 8: SUB-WF — Parallel Social Enrichment

This is the three-source parallel enrichment with tier-based depth.

**Step 8a: Source 1 — Exa Recent Articles (ALWAYS runs)**
- **n8n node type:** HTTP Request

```json
{
  "query": "Recent articles written by {{$json.contact_name}}",
  "type": "auto",
  "numResults": 5,
  "startPublishedDate": "{{6 months ago in ISO 8601}}",
  "contents": {
    "highlights": true,
    "text": {
      "maxCharacters": 1000
    }
  }
}
```

**Parse:**

```javascript
const results = $input.item.json.results || [];

// Find the most recent, most relevant article
let bestArticle = null;
for (const r of results) {
  if (!bestArticle || (r.publishedDate && r.publishedDate > (bestArticle.publishedDate || ""))) {
    bestArticle = r;
  }
}

return {
  json: {
    source1_title: bestArticle?.title || "",
    source1_url: bestArticle?.url || "",
    source1_date: bestArticle?.publishedDate || "",
    source1_summary: (bestArticle?.highlights || []).join(" ") || (bestArticle?.text || "").substring(0, 300),
    source1_available: !!bestArticle
  }
};
```

**Cost:** ~$0.007 + 5 × $0.001 = ~$0.012

---

**Step 8b: Source 2 — Grok API X Posts (runs IF x_handle is available)**

**IF node:** `x_handle` is not empty → proceed, else skip with empty result.

- **n8n node type:** HTTP Request
- **Method:** POST
- **URL:** `https://api.x.ai/v1/chat/completions`
- **Headers:** Authorization: Bearer [YOUR_XAI_API_KEY]
- **Body:**

```json
{
  "messages": [
    {
      "role": "system",
      "content": "You are searching X (Twitter) for the most recent professional posts by a specific person. Focus on their key opinions, insights, and topics they are currently engaged with. Return 2-3 specific quotes with approximate dates. Ignore retweets and purely personal content. Focus on professional/industry content that would be relevant for personalized outreach."
    },
    {
      "role": "user",
      "content": "What has @{{$json.x_handle}} been posting about recently on X? Focus on their most substantive professional insights from the last 30 days."
    }
  ],
  "model": "grok-3-latest",
  "search_parameters": {
    "mode": "on",
    "sources": [
      {
        "type": "x",
        "x_handles": ["{{$json.x_handle}}"]
      }
    ],
    "return_citations": true,
    "from_date": "{{30 days ago in YYYY-MM-DD}}"
  }
}
```

**Parse response:**

```javascript
const response = $input.item.json;
const content = response.choices?.[0]?.message?.content || "";
const citations = response.citations || [];

// Find the first X citation URL
const xCitationUrl = citations.find(c =>
  c.url && (c.url.includes('x.com') || c.url.includes('twitter.com'))
)?.url || "";

return {
  json: {
    source2_content: content.substring(0, 500),
    source2_url: xCitationUrl,
    source2_date: new Date().toISOString().split('T')[0], // Approximate (today)
    source2_available: content.length > 50
  }
};
```

**Cost:** xAI token pricing. For a short synthesis: ~$0.005-0.02

---

**Step 8c: Source 3 — LinkUpAPI LinkedIn Activity (Tier A ONLY)**

**IF node:** `tier` equals "A" AND `source1_available` is false AND `source2_available` is false → proceed, else skip.

Actually, simplify: only run for Tier A regardless of other sources. The social scorer will pick the best one.

**Revised IF:** `tier` equals "A" → proceed.

- **n8n node type:** HTTP Request
- **Method:** POST
- **URL:** `https://api.linkupapi.com/v1/data/person/profile-info-direct`
- **Headers:** Authorization: Bearer [YOUR_LINKUPAPI_KEY]
- **Body:**

```json
{
  "linkedin_url": "{{$json.linkedin_url}}"
}
```

**Prerequisite:** This only works if we have a LinkedIn URL (from Prospeo or EnrichLayer). If no LinkedIn URL, skip.

**IF node before this call:** `linkedin_url` is not empty AND `tier` equals "A"

**Parse response:**

```javascript
const response = $input.item.json;

// LinkUpAPI returns profile data including recent activity
const activities = response.activities || response.recent_posts || [];
const latestActivity = activities[0] || {};

return {
  json: {
    source3_content: latestActivity.text || latestActivity.title || "",
    source3_url: latestActivity.url || latestActivity.link || "",
    source3_date: latestActivity.date || latestActivity.published_date || "",
    source3_available: !!(latestActivity.text || latestActivity.title)
  }
};
```

**Cost:** 1 credit (~$0.058 at Basic tier)

---

### Node 9: LLM — Social Context Scorer

**Merge all three source results** into a single item, then score.

- **n8n node type:** Merge (combine Source 1, 2, 3 results with the prospect data)
- Then: AI Agent node (any LLM)

**System Prompt:**

```
You are evaluating social context data to select the single best personalization point for an outreach email.

The campaign topic is: {{target_industry_keywords}}

You have up to three sources of recent activity for the contact. Some may be empty.

SCORING RULES:
Recency (higher = better):
- Last 7 days = 10
- Last 14 days = 8
- Last 30 days = 6
- Last 60 days = 4
- Last 90 days = 2
- Older or unknown = 1

Relevance (how closely the content relates to the campaign topic):
- Directly on-topic = 10
- Adjacent topic = 6
- Tangentially related = 3
- Unrelated or empty = 0

Combined Score = Recency + Relevance (max 20)

SELECTION PRIORITY:
1. Highest combined score wins
2. If tied, prefer X post > Web article > LinkedIn post (X is most personal/authentic)
3. If all scores are below 6, set flag_for_manual = true

RESPOND IN VALID JSON ONLY:
{
  "best_text": "A 1-2 sentence personalization reference for an email opening paragraph. Write it as if you're referencing something you genuinely noticed about their work. Do NOT start with 'I noticed' or 'I saw'.",
  "source_type": "X Post" or "Web Article" or "LinkedIn Post" or "None",
  "source_url": "the URL of the chosen source" or "",
  "recency_score": number,
  "relevance_score": number,
  "combined_score": number,
  "flag_for_manual": true or false
}
```

**User Prompt:**

```
Contact: {{contact_name}} ({{contact_role}} at {{publication_name}})

Source 1 — Recent Web Article:
Title: {{source1_title}}
URL: {{source1_url}}
Date: {{source1_date}}
Content: {{source1_summary}}

Source 2 — Recent X/Twitter Post:
Content: {{source2_content}}
URL: {{source2_url}}
Date: {{source2_date}}

Source 3 — LinkedIn Activity:
Content: {{source3_content}}
URL: {{source3_url}}
Date: {{source3_date}}

Select the single best personalization point from the available sources.
```

**Parse response:** Same JSON extraction pattern as WF1/WF2 — strip markdown fences, find JSON object, parse.

---

### Node 10: Publication Context (Guest Post Only)

**IF node:** `workflow_type` equals "Guest Post" → proceed. Otherwise skip.

- **n8n node type:** HTTP Request
- **Method:** POST
- **URL:** `https://api.linkup.so/v1/search`
- **Headers:** Authorization: Bearer [YOUR_LINKUP_API_KEY]
- **Body:**

```json
{
  "q": "What topics does {{$json.publication_name}} cover? What are their 5 most recent article headlines in the section most relevant to {{$json.target_industry_keywords}}? Do they have guest post or contribution guidelines?",
  "depth": "standard",
  "outputType": "structured",
  "structuredOutputSchema": "{\"type\":\"object\",\"properties\":{\"editorial_focus\":{\"type\":\"string\",\"description\":\"2-3 sentence description of the publications main topics and audience\"},\"recent_headlines\":{\"type\":\"array\",\"items\":{\"type\":\"string\"},\"description\":\"5 most recent relevant article headlines\"},\"guidelines_url\":{\"type\":\"string\",\"description\":\"URL to guest post or contribution guidelines page if found\"}}}"
}
```

**Validated Linkup.so behavior:**
- `depth: "standard"` costs €0.005 per call
- `outputType: "structured"` forces the response into the JSON schema — no extra charge
- No credit subtracted for failed/empty queries ("success-only" pricing)
- The response will be a JSON object matching the schema

**Parse:**

```javascript
const response = $input.item.json;
const structuredData = response.output || response.structured || {};

return {
  json: {
    pub_editorial_focus: structuredData.editorial_focus || "",
    pub_recent_headlines: (structuredData.recent_headlines || []).join(" | "),
    pub_guidelines_url: structuredData.guidelines_url || ""
  }
};
```

---

### Node 11: Write to Tab 6

- **n8n node type:** Google Sheets → Append Row
- **Sheet:** "Enriched Prospects" (Tab 6)
- **Column mapping:**

| Sheet Column | Source |
|-------------|--------|
| Prospect ID | carried from Tab 5 |
| Workflow Type | carried from Tab 5 |
| Tier | carried from Tab 5 |
| Contact Name | from Node 5a/5b |
| Contact Role | from Node 5a/5b |
| Verified Email | from Node 6 |
| Email Source | Prospeo / EnrichLayer / Manual |
| LinkedIn URL | from Prospeo or EnrichLayer |
| X Handle | from Node 7 |
| X Handle Source | EnrichLayer / Exa search / Not found |
| Best Personalization Text | from Node 9 (social scorer) |
| Personalization Source | X Post / Web Article / LinkedIn Post / None |
| Personalization URL | from Node 9 |
| Recency Score | from Node 9 |
| Relevance Score | from Node 9 |
| Recent Article Title | source1_title from Node 8a |
| Recent Article URL | source1_url from Node 8a |
| Publication Context (GP only) | pub_editorial_focus from Node 10 |
| Guidelines URL (GP only) | pub_guidelines_url from Node 10 |
| Section Target (Linkable only) | carried from Tab 3 (populated by WF2) |
| Status | "Enriched" or "Flagged for Manual Research" |

**Status logic:**

```javascript
const flagForManual = $json.flag_for_manual
  || !$json.verified_email
  || ($json.contact_name === "Unknown" || $json.contact_name === "Editor");

return {
  json: {
    ...$json,
    status: flagForManual ? "Flagged for Manual Research" : "Enriched"
  }
};
```

---

## 6. Cost Estimation Per Prospect (by Tier)

| Step | Tier A | Tier B | Tier C |
|------|--------|--------|--------|
| Contact Resolution (Exa for GP) | $0.012 | $0.012 | $0.012 |
| Prospeo Email | $0.025 | $0.025 | $0.025 |
| EnrichLayer fallback (30% of time) | $0.015 | $0.015 | $0.015 |
| EnrichLayer X handle | $0.034 | $0.034 | — |
| Exa X handle fallback (30%) | $0.002 | $0.002 | — |
| Exa Recent Articles | $0.012 | $0.012 | $0.012 |
| Grok API X Posts | $0.015 | $0.015 | — |
| LinkUpAPI LinkedIn | $0.058 | — | — |
| Linkup.so Pub Context (GP) | $0.005 | $0.005 | — |
| LLM Social Scoring | $0.005 | $0.005 | $0.005 |
| **Estimated Total** | **~$0.18** | **~$0.12** | **~$0.07** |

For a campaign with 200 prospects (20 Tier A, 80 Tier B, 100 Tier C):
- 20 × $0.18 = $3.60
- 80 × $0.12 = $9.60
- 100 × $0.07 = $7.00
- **Total enrichment cost: ~$20.20**

---

## 7. Sub-Workflow Architecture (Recommended)

Given the complexity, split WF5 into a master workflow + 4 sub-workflows in n8n:

| Workflow | Purpose | Nodes |
|----------|---------|-------|
| **WF5-Master** | Reads prospects, loops, calls sub-WFs, writes to Tab 6 | 5 nodes |
| **WF5a-Contact** | Contact resolution (branch on type, Exa/Linkup editor search) | 6 nodes |
| **WF5b-Email** | Prospeo → EnrichLayer waterfall | 5 nodes |
| **WF5c-Social** | X handle resolution + parallel social enrichment + scoring | 12 nodes |
| **WF5d-PubContext** | Linkup.so publication context (Guest Post only) | 3 nodes |

Each sub-workflow is called via n8n's "Execute Workflow" node. Data passes in/out as JSON. This makes each piece independently testable and debuggable.

---

## 8. Error Handling

| API | Error | Action |
|-----|-------|--------|
| Prospeo | No match (error: true) | Proceed to EnrichLayer fallback. No credit charged. |
| Prospeo | 429 rate limit | Wait 10 seconds, retry once. |
| EnrichLayer | No email found | Mark email_source = "Manual", flag_for_manual = true. Refunded. |
| EnrichLayer | No twitter_profile_id | Proceed to Exa X handle fallback. |
| Exa | No results for editor search | Flag contact_name = "Editor", flag_for_manual = true. |
| Exa | No results for X handle | Set x_handle = "", Source 2 is skipped. |
| Grok API | No posts found | Source 2 returns empty. Scorer works with Sources 1 and 3. |
| LinkUpAPI | No LinkedIn URL available | Source 3 is skipped entirely. |
| Linkup.so | Empty structured output | Publication context fields are left blank. |
| LLM Scorer | JSON parse failure | Default to Source 1 (web article) if available, else flag_for_manual. |

**Critical principle:** No single API failure should crash the workflow. Every step has a fallback or a graceful skip. The worst case is a prospect flagged for manual research — never a lost prospect.

---

## 9. Testing Checklist

### Test with 1 Linkable Asset prospect (Tier B):
- [ ] Author name pulled correctly from Tab 3
- [ ] Prospeo returns verified email (or gracefully falls back)
- [ ] EnrichLayer returns X handle (or Exa fallback works)
- [ ] Exa returns recent articles for Source 1
- [ ] Grok API returns X posts for Source 2 (if handle found)
- [ ] Social scorer picks the best personalization point
- [ ] Tab 6 row is complete with all fields populated

### Test with 1 Guest Post prospect (Tier A):
- [ ] Editor found via Exa search (or flagged for manual)
- [ ] All three social sources attempted
- [ ] Linkup.so returns publication context
- [ ] Guidelines URL found (if publication has a write-for-us page)

### Test failure scenarios:
- [ ] Prospect with no email found anywhere → Status = "Flagged for Manual Research"
- [ ] Prospect with no X handle → Source 2 skipped, scorer works with Sources 1 and 3
- [ ] Prospect with no social data at all → flag_for_manual = true

---

## 10. Data Flow Summary

```
Tab 5 (Vetted, Tiered)
├── Prospect ID, Domain, Workflow Type, Tier
│
├── IF Linkable → Pull Author from Tab 3
│   └── author_name, article_url, article_title
│
├── IF Guest Post → Find Editor
│   └── Exa search OR Linkup.so (Tier A)
│
├── Email Resolution
│   ├── Prospeo (primary) ─── found? → done
│   └── EnrichLayer (fallback) ─── found? → done → else flag manual
│
├── X Handle Resolution
│   ├── EnrichLayer twitter_profile_id ─── found? → done
│   └── Exa site:x.com search (fallback) ─── found? → done
│
├── Parallel Social Enrichment
│   ├── Source 1: Exa Recent Articles (always)
│   ├── Source 2: Grok API X Posts (if handle)
│   └── Source 3: LinkUpAPI LinkedIn (Tier A only)
│
├── Social Context Scorer (LLM)
│   └── Picks best personalization point, scores recency + relevance
│
├── Publication Context (Guest Post only)
│   └── Linkup.so structured output
│
└── WRITE → Tab 6 (Enriched Prospects)
    ├── Contact: name, role, email, LinkedIn, X handle
    ├── Social: best personalization text + source + scores
    ├── Publication: editorial focus, headlines, guidelines
    └── Status: Enriched / Flagged for Manual Research

YOUR REVIEW → approve for pitch or do manual research on flagged items
```
