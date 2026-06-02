# Setup & Configuration

> Public, generic version. No real credentials are included, configure your own.

## Prerequisites

| Service | What for | Expected credential |
|---|---|---|
| **n8n** | Runs the ingestion workflow | self-hosted or cloud |
| **OpenAI** | Generates the embeddings (1536 dim) | `OPENAI_API_KEY` |
| **Supabase / Postgres + pgvector** | Vector store (storage and search) | `SUPABASE_URL` + `SUPABASE_SERVICE_ROLE_KEY` |

## 1. Provision the vector base (prerequisite SQL)

In the Supabase SQL Editor, run the script below. It enables the `vector` extension, creates the `documents` table, and the `match_documents` function (cosine distance search with metadata filtering).

```sql
create extension if not exists vector;

create table if not exists documents (
  id bigserial primary key,
  content text,
  metadata jsonb,
  embedding vector(1536)
);

create or replace function match_documents (
  query_embedding vector(1536),
  match_count int default null,
  filter jsonb default '{}'
) returns table (
  id bigint,
  content text,
  metadata jsonb,
  similarity float
) language plpgsql as $$
#variable_conflict use_column
begin
  return query
  select id, content, metadata,
    1 - (documents.embedding <=> query_embedding) as similarity
  from documents
  where metadata @> filter
  order by documents.embedding <=> query_embedding
  limit match_count;
end;
$$;
```

> Dimension coupling: `vector(1536)` must match the chosen embedding model (for example, `text-embedding-3-small` is 1536 dimensions). If you change the model, adjust the column dimension.

## 2. Import the workflow

1. `n8n` > `Workflows` > `Import from File` > `workflow.json`
2. The nodes already point to credential placeholders, just select your own.

## 3. Create the credentials in n8n

In `Credentials` > `New`, register:

- **Supabase API**: Host (`SUPABASE_URL`) and Service Role Key (`Project Settings > API`)
- **OpenAI API**: your API key

Then open each node and select the matching credential:

| Node | Credential |
|---|---|
| OpenAI Embeddings | OpenAI API |
| Insert into Supabase pgvector | Supabase API |

## 4. Activate and test

1. Activate the workflow.
2. Open the form URL (Form Trigger node) and upload a test document, selecting one or more categories.
3. Check in Supabase that the `documents` table received rows with `content`, `metadata` (categories, source, date), and `embedding`.
4. From there, an agent can call `match_documents` to retrieve passages by similarity.

## Environment variables (example)

```env
OPENAI_API_KEY=sk-xxxxxxxx
SUPABASE_URL=https://xxxx.supabase.co
SUPABASE_SERVICE_ROLE_KEY=xxxxxxxx
```

> Never commit real credentials. Use environment variables or the n8n credential manager.
