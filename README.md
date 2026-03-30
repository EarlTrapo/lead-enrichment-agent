# lead-enrichment-agent

An automated B2B lead enrichment pipeline built on n8n. Triggers when new company records land in a Google Sheet, researches each company using web search, extracts structured profile data, and then runs a contact finder to pull verified decision-maker emails — all without manual input.

---

## What it does

A new row added to the lead sheet kicks off the pipeline. For each company, an AI agent searches the web and returns a structured profile: what the company does, estimated headcount, location, and any decision-maker signals it can find. That profile gets written to an enriched sheet. The pipeline then hands the domain off to a contact finder service to pull verified email addresses for the right titles — founders, ops leads, directors, and similar roles.

The result is a populated enriched sheet with company context and verified contacts, ready for outreach.

---

## Pipeline stages

```
Google Sheets trigger (new row)
        │
        ▼
Loop Per Company (batches of 5)
        │
        ▼
AI Agent — web research per company
├── Tavily web search
└── Postgres chat memory
        │
        ▼
Collect + extract fields (JSON parse + cleanup)
        │
        ▼
Append enriched row to Google Sheet
        │
        ▼
Email finder (Apify actor) ← completes the pipeline
```

---

## Stages in detail

### Trigger
Watches a Google Sheet for new rows. Each new row is expected to contain at minimum a `company_name` and `domain` field.

### Loop Per Company
Processes companies in batches. After each company is researched, the loop waits before continuing — this prevents rate-limiting issues with the search tool.

### AI research agent
Takes the company name and domain, searches the web, and returns a structured JSON profile:

| Field | Description |
|---|---|
| `company_name` | Confirmed company name |
| `domain` | Website domain |
| `description` | One-sentence summary of what they do |
| `employee_estimate` | Headcount range |
| `location` | City and country |
| `decision_maker_hint` | Any name or title found publicly |
| `notes` | Relevant context — owner-operated, pain points, etc. |

Fields that cannot be confirmed are returned as `null`. The agent never fabricates.

### Field extraction
A code node parses the agent's JSON output, strips any markdown formatting, and handles malformed responses gracefully before writing to the sheet.

### Enriched sheet write
Appends the structured profile to a separate `enriched` tab in the same Google Sheet, keeping the raw input and enriched output in the same file.

### Email finder (pipeline completion)
This stage runs an Apify actor — a leads scraper that takes the company domain and returns verified contact profiles filtered by title (founders, COOs, directors, ops leads, etc.) with confirmed email addresses.

**This is the step that completes the pipeline.** Without a working email finder connected, the workflow produces company profiles but stops short of actionable contacts. You can swap the Apify actor for any equivalent service (Apollo, Hunter, Prospeo, etc.) — the domain is the input and verified emails are the expected output.

---

## Data layer

| Store | Purpose |
|---|---|
| Google Sheets (`list1` tab) | Input — raw company names and domains |
| Google Sheets (`enriched` tab) | Output — structured profiles |
| Postgres | Chat memory for the AI agent |

---

## Setup

### Prerequisites
- n8n (self-hosted or cloud)
- Google account with Sheets OAuth configured
- LLM API key (any provider supported by n8n)
- Tavily API key
- Postgres database
- Apify account (or equivalent contact finder)

### Steps

1. Import `leadEnrichment.json` into your n8n instance
2. Reconnect credentials: Google Sheets (trigger + read/write), LLM provider, Tavily, Postgres, Apify
3. Create or point to a Google Sheet with two tabs:
   - Input tab: columns — `company_name`, `domain`
   - Output tab: columns — `company_name`, `domain`, `description`, `employee_estimate`, `location`, `decision_maker_hint`, `notes`
4. Update the Google Sheets trigger and append nodes to point to your sheet ID
5. Configure the Apify actor node with your preferred leads scraper and target country/titles
6. Activate the workflow

### Input format

Each row in the input sheet should contain:
```
company_name | domain
Acme Corp    | acme.com
```

---

## File structure

```
lead-enrichment-agent/
├── leadEnrichment.json    # Full n8n workflow export
├── README.md
└── assets/
    └── workflow.png       # Workflow canvas screenshot
```

---

## Built by

[@0xTrapo](https://x.com/0xTrapo) — AI automation consultant building workflow systems for B2B operations.
