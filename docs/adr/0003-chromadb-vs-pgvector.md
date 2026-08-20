# ADR 0003: Embedded ChromaDB over pgvector

**Status:** Accepted
**Date:** 2026-08-19

## Context

PostgreSQL is already in the stack for session state and chat history. Putting pgvector on top of it
would reuse what's already deployed, without adding a new technology. That argument is real, which is
why this decision can't be "obviously, use whatever's most specialized." What's at stake isn't speed:
at this MVP's scale, with a self-authored corpus and a single instance, both ChromaDB and pgvector
return results in milliseconds. What's actually being decided is how simple the system's mental model
stays, and how many responsibilities get coupled into the same service.

## Alternatives

**Embedded ChromaDB in the app process**
The vector store runs inside the FastAPI container. Advantages: no extra service, no extensions to
enable, an API oriented directly at RAG (add, query, metadata). Disadvantages: if the app is
horizontally replicated, each replica would have its own copy; and it consumes memory from the same
process that serves requests.

**pgvector on top of the existing PostgreSQL**
Enable the extension and store embeddings in a separate table. Advantages: a single database for
everything, no second storage engine. Disadvantages: mixes relational and vector storage in the same
service; requires migrations and vector-index management inside Postgres; and couples the vector
store's lifecycle to that of the critical database.

**A managed vector store like Pinecone / Weaviate**
A specialized external API. Discarded quickly: fixed or per-query cost, added network latency, and one
more external dependency to worry about. For this MVP it adds nothing that the two local options don't
already cover.

## Decision and why

**Decision:** embedded ChromaDB, inside the app's container.

The main reason isn't performance or "specialization." It's separation of concerns and operational
simplicity. Using pgvector would mean PostgreSQL — already the store for session state and chat
history — also becomes the vector store. One service with two responsibilities of a different
nature: transactional and similarity search. If the vector store grows, it degrades the relational
store; if the relational store gets updated, it competes with vector queries. That's not a problem at
this scale, but it's coupling that's avoidable from the design stage.

Embedded ChromaDB keeps the vector store as a conceptually separate piece, even though it physically
lives in the same process as the app. That lets me reason about the ingestion and retrieval flow
without dragging in Postgres migrations or schemas. The trade-off is that it consumes memory from the
app's process — but on a single free-tier instance, that cost is comparable to what pgvector would
cost running inside Postgres. There's no real resource advantage between the two options; what there
is, is an advantage in clarity.

## Consequences

**Future migration.** If at some point I want to move to pgvector or a managed service, the pain
depends on how coupled the code is to Chroma's API. The mitigation is the same as in the embeddings
ADR: define a `VectorStore` interface with add, query, delete operations, with ChromaDB as just one
implementation. Store the embeddings model and chunking metadata in the corpus from day one, so a
migration doesn't mean re-ingesting blind.

**Bounded scalability.** This decision explicitly holds for the corpus size this project has today:
self-authored documents, no automated ingestion, on a single instance. If the corpus grew a lot,
embedded ChromaDB would stop being the right choice: the vector store would need to move to a separate
process or a managed service. That's why it's essential that the `VectorStore` interface exist from
the MVP, even while the implementation stays embedded.

**Deferred autoscaling.** Horizontal scaling is deferred today, and with embedded ChromaDB that's a
real limitation: two replicas would have two diverging copies of the index. As long as the system is a
monolith on a single EC2 instance, that's acceptable and documented as a limit. If autoscaling were
ever enabled, the vector store would have to move out of the process and become a separate service —
exactly the kind of change the `VectorStore` interface should absorb without rewriting the pipeline.
