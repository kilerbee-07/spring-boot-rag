# Spring Boot Docs RAG Assistant

A Retrieval-Augmented Generation (RAG) system that answers technical questions about Spring Boot by retrieving relevant excerpts from official Spring Boot documentation and generating grounded, cited answers using Google's Gemini model.

## Overview

This project scrapes official Spring Boot reference documentation, chunks it by section, embeds each chunk into a vector space, stores the vectors in a Postgres database (via Supabase + pgvector), and retrieves the most relevant chunks for any user question using cosine similarity search. Retrieved chunks are passed to Gemini as grounding context, so answers are based on actual documentation text rather than the model's general training knowledge — with every answer citing which doc section it came from.

## Architecture

```
User question
     │
     ▼
Embed question (sentence-transformers, all-MiniLM-L6-v2)
     │
     ▼
Similarity search (Supabase pgvector, match_documents RPC)
     │
     ▼
Top-K relevant chunks retrieved
     │
     ▼
Chunks inserted into prompt as grounding context
     │
     ▼
Gemini (gemini-flash-latest) generates cited answer
     │
     ▼
Answer + source sections + similarity scores printed
```

## Tech Stack

| Component | Tool |
|---|---|
| Scraping | `requests` + `BeautifulSoup` |
| Embedding model | `sentence-transformers` (`all-MiniLM-L6-v2`, 384-dim) |
| Vector database | Supabase (Postgres + pgvector extension) |
| LLM | Google Gemini (`gemini-flash-latest`) |
| Runtime | Google Colab |

## Knowledge Base

The system currently indexes four official Spring Boot reference pages, covering distinct subsystems:

- **Data / SQL** — `reference/data/sql.html` (embedded databases, connection pooling, schema init)
- **Servlets** — `reference/web/servlet.html` (filters, listeners, dispatcher types)
- **NoSQL** — `reference/data/nosql.html` (MongoDB, Redis, Elasticsearch connections)
- **Security** — `reference/web/spring-security.html` (auto-configuration, default users, login strategy)

As of the latest ingest, this totals **254 chunks** across all four pages, each tagged with its source section title and URL for citation.

## Key Design Decisions

**Section-aware chunking.** Rather than splitting text by a fixed character count alone, the chunker tracks the most recent HTML heading and flushes a chunk whenever the section changes. This keeps each chunk topically coherent and lets every chunk carry an accurate `section_title` for citation, instead of a generic page title.

**Idempotent ingestion.** `ingest_pages()` checks existing content in Supabase before inserting, skipping any chunk whose text already exists in the table. This was added after an early version caused duplicate rows on re-run; the current version can be safely re-run any number of times (e.g., after a Colab disconnect) without corrupting the dataset.

**Grounded, citation-required prompting.** The Gemini prompt explicitly instructs the model to answer *only* from the provided excerpts and to say so if the excerpts are insufficient, rather than falling back on general knowledge. Every answer includes source section names, URLs, and similarity scores for transparency.

## Notebook Structure

The Colab notebook is organized into seven cells, designed to minimize redundant work and API usage:

1. **Install dependencies** — one-time package installs.
2. **Setup** — initializes the embedding model, Supabase client, and Gemini client.
3. **Scraping + chunking functions** — `scrape_page()`, `chunk_blocks()`.
4. **Ingest pipeline** — `ingest_pages()`, with the duplicate-content guard.
5. **Retrieval** — `retrieve_chunks()`, calls the `match_documents` Postgres RPC.
6. **Generation** — `build_context()` and `ask_rag()`, which ties retrieval and generation together.
7. **Usage examples** — verified example queries across all four ingested topics.

Cells 1–6 only need to be re-run after a runtime restart or disconnect; they define functions and clients but don't scrape, embed, or call the LLM on their own. Only Cell 7's calls cost time or API quota.

## Example Queries

```python
answer_sql, sources_sql = ask_rag("How does Spring Boot configure connection pooling?")
answer_servlet, sources_servlet = ask_rag("How does Spring Boot handle servlet filters?")
answer_nosql, sources_nosql = ask_rag("How do I connect Spring Boot to a MongoDB database?")
answer_security, sources_security = ask_rag("What does Spring Boot auto-configure by default when Spring Security is on the classpath?")
```

Each call prints the question, the generated answer, and the retrieved source sections with their similarity scores.

## Adding More Documentation

To expand coverage, call `ingest_pages()` with additional Spring Boot reference URLs:

```python
ingest_pages([
    "https://docs.spring.io/spring-boot/reference/testing/spring-boot-applications.html",
    "https://docs.spring.io/spring-boot/reference/actuator/endpoints.html",
])
```

Good candidate topics not yet covered: testing, Actuator/monitoring, caching, messaging (Kafka/AMQP), and externalized configuration.

## Known Limitations

- Retrieval is limited to exact-topic similarity search (top-K cosine similarity); it does not yet re-rank results or handle multi-hop questions spanning multiple unrelated sections.
- The embedding model (`all-MiniLM-L6-v2`) is lightweight and fast but less nuanced than larger embedding models for subtle semantic distinctions.
- Gemini's free-tier quota (20 requests/day at time of writing) constrains how many test queries can be run per day.
- The system is scoped to Spring Boot reference docs only; it has no awareness of Spring Framework, Spring Cloud, or third-party library documentation unless explicitly ingested.

## Future Improvements

- Wrap `ask_rag()` in an interactive query loop for a more conversational testing experience.
- Add retry/error handling around Gemini calls for graceful quota-limit handling.
- Expand the knowledge base to cover testing, Actuator, caching, and messaging.
- Experiment with re-ranking retrieved chunks before generation to improve precision on ambiguous questions.
