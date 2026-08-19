# NovaCart Finance Reporting Agent

An autonomous RAG pipeline built in n8n that queries internal NovaCart sales data and external market reports from Pinecone, synthesizes them with GPT-4.1 into a structured monthly finance report, and emails it to stakeholders automatically — no manual data wrangling required.

![Architecture diagram](docs/flow_diagram.png)

## What it does

Every month, this pipeline runs unattended and:

1. Refreshes a shared knowledge base with the latest internal (sales/revenue) and external (market outlook) documents
2. Autonomously retrieves relevant context from both sources using an AI agent with tool-calling
3. Synthesizes a structured leadership report — Executive Summary, Internal Performance Highlights, External Market Context, Correlated Insights, and Recommendations
4. Emails the finished report directly to a stakeholder inbox as clean HTML

No human touches it in between ingestion and delivery.

## Architecture

Two separate n8n workflows, staggered to avoid race conditions:

| Workflow | Trigger | What it does |
|---|---|---|
| **Finance_RAG_Ingestion** | Monthly, day 28 | Pulls internal + external docs from Google Drive → chunks → embeds → tags with `source: internal`/`external` metadata → upserts into Pinecone |
| **Finance_RAG_Agent** | Monthly, day 1 | AI Agent (GPT-4.1) retrieves from Pinecone as a tool, synthesizes the monthly report, sends via Gmail |

## Tech stack

| Component | Tool |
|---|---|
| Workflow orchestration | [n8n](https://n8n.io) |
| Vector database | Pinecone |
| Embeddings | OpenAI `text-embedding-3-small` (1024 dimensions) |
| LLM / synthesis | OpenAI GPT-4.1 (AI Agent node) |
| Document sources | Google Drive |
| Delivery | Gmail (OAuth) |

## Repo structure
Two separate n8n workflows...
```
├── workflows/
│ ├── Finance_RAG_Ingestion.json
│ └── Finance_RAG_Agent.json
├── docs/
│ ├── flow_diagram.png
│ └── article.md (full writeup, also published on Medium)
└── screenshots/
└── ... (working pipeline, sample email output)
```

> Workflow JSON files have credential IDs and personal folder/index identifiers removed. You'll need your own Google Drive, OpenAI, Pinecone, and Gmail credentials configured in n8n to run them.

## Key challenges solved

- **Tool-mode wiring**: A vector store used by an AI Agent has to run in "Retrieve as Tool" mode and connect into the Agent's tool input — not the main workflow chain like a typical node.
- **Embedding dimension consistency**: Ingestion and retrieval embeddings must match exactly (model + dimensions), or Pinecone silently returns poor/empty results with no error.
- **Silent namespace mismatch**: The trickiest bug — ingestion and retrieval pointed at different Pinecone namespaces within the same index. The workflow executed successfully end-to-end but returned zero retrieved chunks, since a namespace mismatch throws no error. Aligning namespaces fixed it immediately.
- **Trigger race conditions**: Ingestion and reporting were originally scheduled at the same moment, risking the agent reading stale data mid-ingestion. Staggered to day 28 (ingest) → day 1 (report).
- **Grounded refusal over hallucination**: When internal source data doesn't cover the requested reporting month, the agent explicitly states the data gap rather than inventing plausible-looking figures — verified in production when the static demo sales dataset didn't extend into the live reporting month, and the agent correctly flagged the gap instead of fabricating numbers.

## Known limitations

- **Static internal dataset**: The internal sales/revenue source document is a fixed demo file that doesn't update on its own. The agent correctly reports "no data available" for months beyond the dataset's coverage rather than hallucinating — a real production deployment would need this replaced with a live sales data feed or a source refreshed on an actual schedule.
- **No deduplication on re-ingestion**: Each monthly ingestion run adds new vectors to Pinecone rather than overwriting prior ones, since there's no deterministic chunk ID. Over many months this could bloat the index and dilute retrieval quality — a good next step would be deterministic IDs (e.g. `filename + chunk index`) so re-runs upsert cleanly instead of duplicating.
- **Single shared namespace**: Internal and external data share one Pinecone namespace, differentiated only by metadata rather than physical separation. Works well at this scale, but a larger dataset might benefit from separate namespaces per source type.

## Version history

- **v2**: Added dynamic reporting-period calculation (correct prior-month labeling regardless of exact trigger time), defensive output parsing before email send (handles output field variance and strips stray Markdown), and source citation metadata (file name + Drive URL) so the report can cite specific documents instead of just "internal" or "external" in general.
- **v1**: Initial working pipeline — internal + external ingestion into Pinecone, AI Agent retrieval and synthesis, HTML report delivered via Gmail.

## Read the full writeup

The complete build story, including every bug and how it was diagnosed, is written up here: **https://medium.com/@somnath13/novacart-finance-reporting-agent-17bdb0b748e3**

## Why this project

Built as a hands-on exercise in agentic AI / RAG architecture — going beyond "ask an LLM a question" into a full retrieval-grounded pipeline with real scheduling, real failure modes, and real delivery to an inbox.
