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
├── workflows/
│ ├── Finance_RAG_Ingestion.json
│ └── Finance_RAG_Agent.json
├── docs/
│ ├── flow_diagram.png
│ └── article.md (full writeup, also published on Medium)
└── screenshots/
└── ... (working pipeline, sample email output)

> Workflow JSON files have credential IDs and personal folder/index identifiers removed. You'll need your own Google Drive, OpenAI, Pinecone, and Gmail credentials configured in n8n to run them.

## Key challenges solved

- **Tool-mode wiring**: A vector store used by an AI Agent has to run in "Retrieve as Tool" mode and connect into the Agent's tool input — not the main workflow chain like a typical node.
- **Embedding dimension consistency**: Ingestion and retrieval embeddings must match exactly (model + dimensions), or Pinecone silently returns poor/empty results with no error.
- **Silent namespace mismatch**: The trickiest bug — ingestion and retrieval pointed at different Pinecone namespaces within the same index. The workflow executed successfully end-to-end but returned zero retrieved chunks, since a namespace mismatch throws no error. Aligning namespaces fixed it immediately.
- **Trigger race conditions**: Ingestion and reporting were originally scheduled at the same moment, risking the agent reading stale data mid-ingestion. Staggered to day 28 (ingest) → day 1 (report).

## Read the full writeup

The complete build story, including every bug and how it was diagnosed, is written up here: **[Medium article](#)** *(add your published link)*

## Why this project

Built as a hands-on exercise in agentic AI / RAG architecture — going beyond "ask an LLM a question" into a full retrieval-grounded pipeline with real scheduling, real failure modes, and real delivery to an inbox.
