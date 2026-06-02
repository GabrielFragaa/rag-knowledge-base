# ⚙️ Setup & Configuração

> Versão pública/genérica. **Nenhuma credencial real** está incluída — configure as suas.

## Pré-requisitos

| Serviço | Para quê | Credencial esperada |
|---|---|---|
| **n8n** | Orquestração do workflow | self-hosted ou cloud |
| **OpenAI** | Geração de embeddings (1536 dim) | `OPENAI_API_KEY` |
| **Supabase / Postgres + pgvector** | Vector store (armazenamento e busca) | `SUPABASE_URL` + `SUPABASE_SERVICE_ROLE_KEY` |

## 1. Provisione a base vetorial (SQL pré-requisito)

No **SQL Editor do Supabase**, rode o script abaixo. Ele habilita a extensão `vector`, cria a tabela `documents` e a função `match_documents` (busca por distância de cosseno com filtro por metadados).

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

> ⚠️ **Acoplamento de dimensão:** `vector(1536)` precisa bater com o modelo de embeddings escolhido (ex.: `text-embedding-3-small` = 1536 dimensões). Se trocar o modelo, ajuste a dimensão da coluna.

## 2. Importe o workflow

1. `n8n → Workflows → Import from File → workflow.json`
2. Os nós já apontam para placeholders de credencial — basta selecionar a sua.

## 3. Crie as credenciais no n8n

Em `Credentials → New`, cadastre:

- **Supabase API** — Host (`SUPABASE_URL`) e Service Role Key (`Project Settings > API`)
- **OpenAI API** — sua chave de API

Depois, abra cada nó e selecione a credencial correspondente:

| Nó | Credencial |
|---|---|
| Embeddings OpenAI | OpenAI API |
| Inserir no Supabase pgvector | Supabase API |

## 4. Ative e teste

1. **Ative** o workflow.
2. Abra a **URL do formulário** (nó Form Trigger) e faça upload de um documento de teste, marcando uma ou mais categorias.
3. Confira no Supabase que a tabela `documents` recebeu linhas com `content`, `metadata` (categorias, fonte, data) e `embedding`.
4. A partir daí, um agente pode chamar `match_documents` para recuperar trechos por similaridade.

## Variáveis de ambiente (exemplo)

```env
OPENAI_API_KEY=sk-xxxxxxxx
SUPABASE_URL=https://xxxx.supabase.co
SUPABASE_SERVICE_ROLE_KEY=xxxxxxxx
```

> 🔒 Nunca comite credenciais reais. Use variáveis de ambiente / o gerenciador de credenciais do n8n.
