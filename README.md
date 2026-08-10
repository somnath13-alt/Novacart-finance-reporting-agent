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
