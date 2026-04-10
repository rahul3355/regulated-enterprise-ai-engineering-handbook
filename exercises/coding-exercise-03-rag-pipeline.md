# Coding Exercise 03: RAG Pipeline

> Build a Retrieval-Augmented Generation pipeline using pgvector — the core of every enterprise GenAI system.

## Problem Statement

You are building a RAG pipeline for the bank's internal knowledge assistant. The pipeline must:

1. Ingest policy documents (text) and store them with embeddings in PostgreSQL using pgvector
2. Retrieve relevant documents for a user query using semantic similarity search
3. Generate a response by combining retrieved documents with the query
4. Include source citations in the response

**Banking Context:** The bank has thousands of policy documents (HR policies, compliance guidelines, IT procedures). Employees need to ask questions and get accurate, sourced answers. Every answer must cite its source documents so employees can verify accuracy.

## Constraints

- Use PostgreSQL with the `pgvector` extension
- Use `sentence-transformers/all-MiniLM-L6-v2` for embeddings (384 dimensions)
- Maximum 3 documents retrieved per query
- Similarity threshold: cosine distance < 0.5
- Documents have metadata: `title`, `category`, `version`, `effective_date`
- Response must include citations with document title and version
- Handle the case where no relevant documents are found

## Expected Output

```python
# Ingest:
ingest_document(
    doc_id="POL-001",
    title="Expense Reporting Policy",
    category="finance",
    content="All employees must submit expense reports within 30 days...",
    version="3.2",
    effective_date="2026-01-01"
)

# Query:
result = query_rag("What is the deadline for submitting expense reports?")

# Response:
{
    "query": "What is the deadline for submitting expense reports?",
    "response": "According to the Expense Reporting Policy (v3.2), employees must submit expense reports within 30 days of the expense date. Expenses submitted after 30 days require manager approval.",
    "citations": [
        {
            "doc_id": "POL-001",
            "title": "Expense Reporting Policy",
            "version": "3.2",
            "category": "finance",
            "similarity_score": 0.82
        }
    ],
    "documents_retrieved": 1,
    "latency_ms": 45
}
```

## Hints

### Hint 1: Database Setup with pgvector

```sql
-- Enable the pgvector extension
CREATE EXTENSION IF NOT EXISTS vector;

-- Create the documents table
CREATE TABLE policy_documents (
    doc_id VARCHAR(50) PRIMARY KEY,
    title VARCHAR(255) NOT NULL,
    category VARCHAR(50) NOT NULL,
    content TEXT NOT NULL,
    version VARCHAR(20) NOT NULL,
    effective_date DATE NOT NULL,
    embedding vector(384),
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Create an index for similarity search
CREATE INDEX ON policy_documents USING ivfflat (embedding vector_cosine_ops);
```

### Hint 2: Generate Embeddings

```python
from sentence_transformers import SentenceTransformer

model = SentenceTransformer('all-MiniLM-L6-v2')

def generate_embedding(text: str) -> list[float]:
    """Generate a 384-dimensional embedding for the given text."""
    return model.encode(text).tolist()
```

### Hint 3: Similarity Search Query

```sql
-- Find documents most similar to the query embedding
SELECT doc_id, title, category, content, version,
       1 - (embedding <=> $1) AS similarity_score
FROM policy_documents
WHERE 1 - (embedding <=> $1) > 0.5
ORDER BY similarity_score DESC
LIMIT 3;
```

### Hint 4: Response Generation (Mock)

```python
def generate_response(query: str, documents: list[dict]) -> str:
    """
    Generate a response using retrieved documents.
    In production, this would call an LLM API with a prompt like:

    'Answer the question using ONLY the provided context.
    If the context doesn't contain the answer, say so.

    Context:
    {doc content here}

    Question: {query}
    '
    """
    # For this exercise, extract a relevant sentence manually
    # In production, call the LLM API
    context = "\n\n".join(doc["content"] for doc in documents)
    # Simple keyword-based answer for the exercise
    keywords = query.lower().split()
    for keyword in keywords:
        for doc in documents:
            if keyword in doc["content"].lower():
                sentences = doc["content"].split(". ")
                for s in sentences:
                    if keyword in s.lower():
                        return s + "."
    return "I couldn't find specific information about that in our policy documents."
```

## Example Solution

```python
"""
RAG Pipeline for Banking Knowledge Assistant
Uses pgvector for semantic document retrieval.
"""

import json
import time
from datetime import datetime
from typing import Optional

import psycopg2
from psycopg2.extras import RealDictCursor
from sentence_transformers import SentenceTransformer

# ─── Configuration ─────────────────────────────────────────────

DB_CONFIG = {
    "dbname": "banking_rag",
    "user": "rag_user",
    "password": "secure_password",
    "host": "localhost",
    "port": 5432,
}

EMBEDDING_MODEL = SentenceTransformer('all-MiniLM-L6-v2')
EMBEDDING_DIM = 384
MAX_DOCUMENTS = 3
SIMILARITY_THRESHOLD = 0.5

# ─── Database ──────────────────────────────────────────────────

def init_db():
    """Initialize the database with pgvector extension and schema."""
    conn = psycopg2.connect(**DB_CONFIG)
    with conn.cursor() as cur:
        cur.execute("CREATE EXTENSION IF NOT EXISTS vector;")
        cur.execute("""
            CREATE TABLE IF NOT EXISTS policy_documents (
                doc_id VARCHAR(50) PRIMARY KEY,
                title VARCHAR(255) NOT NULL,
                category VARCHAR(50) NOT NULL,
                content TEXT NOT NULL,
                version VARCHAR(20) NOT NULL,
                effective_date DATE NOT NULL,
                embedding vector(384),
                created_at TIMESTAMP DEFAULT NOW(),
                updated_at TIMESTAMP DEFAULT NOW()
            );
        """)
        cur.execute("""
            CREATE INDEX IF NOT EXISTS doc_embedding_idx
            ON policy_documents USING ivfflat (embedding vector_cosine_ops)
            WITH (lists = 100);
        """)
    conn.commit()
    conn.close()

def generate_embedding(text: str) -> list[float]:
    """Generate embedding for text."""
    return EMBEDDING_MODEL.encode(text).tolist()

def ingest_document(
    doc_id: str,
    title: str,
    category: str,
    content: str,
    version: str,
    effective_date: str
) -> bool:
    """Ingest a document with its embedding."""
    embedding = generate_embedding(content)
    conn = psycopg2.connect(**DB_CONFIG)
    with conn.cursor() as cur:
        cur.execute("""
            INSERT INTO policy_documents
                (doc_id, title, category, content, version, effective_date, embedding)
            VALUES (%s, %s, %s, %s, %s, %s, %s)
            ON CONFLICT (doc_id) DO UPDATE SET
                title = EXCLUDED.title,
                category = EXCLUDED.category,
                content = EXCLUDED.content,
                version = EXCLUDED.version,
                effective_date = EXCLUDED.effective_date,
                embedding = EXCLUDED.embedding,
                updated_at = NOW();
        """, (doc_id, title, category, content, version, effective_date, json.dumps(embedding)))
    conn.commit()
    conn.close()
    return True

def retrieve_documents(query: str, max_docs: int = MAX_DOCUMENTS) -> list[dict]:
    """Retrieve relevant documents using semantic similarity."""
    query_embedding = generate_embedding(query)
    conn = psycopg2.connect(**DB_CONFIG)
    with conn.cursor(cursor_factory=RealDictCursor) as cur:
        cur.execute("""
            SELECT doc_id, title, category, content, version,
                   effective_date,
                   1 - (embedding <=> %s::vector) AS similarity_score
            FROM policy_documents
            WHERE 1 - (embedding <=> %s::vector) > %s
            ORDER BY similarity_score DESC
            LIMIT %s;
        """, (json.dumps(query_embedding), json.dumps(query_embedding),
              SIMILARITY_THRESHOLD, max_docs))
        results = cur.fetchall()
    conn.close()

    # Convert Decimal similarity to float and remove content from citation
    docs = []
    for row in results:
        docs.append({
            "doc_id": row["doc_id"],
            "title": row["title"],
            "category": row["category"],
            "content": row["content"],
            "version": row["version"],
            "effective_date": str(row["effective_date"]),
            "similarity_score": float(row["similarity_score"]),
        })
    return docs

def generate_response(query: str, documents: list[dict]) -> str:
    """
    Generate a response from retrieved documents.

    In production, this would call an LLM with:
    - System prompt defining the assistant's role
    - Retrieved documents as context
    - User query
    - Instructions to cite sources

    For this exercise, we synthesize a response from the documents.
    """
    if not documents:
        return "I couldn't find relevant information about that topic in our policy documents. Please contact the HR or Compliance team directly for guidance."

    # Build context from documents
    context_parts = []
    for i, doc in enumerate(documents, 1):
        context_parts.append(
            f"[Document {i}: {doc['title']} v{doc['version']}]\n{doc['content']}"
        )
    context = "\n\n".join(context_parts)

    # In production, this would be an LLM call:
    # prompt = f"""Answer the question using ONLY the provided context.
    # Cite the source document and version.
    #
    # Context:
    # {context}
    #
    # Question: {query}
    #
    # Answer:"""
    # response = call_llm_api(prompt)

    # For the exercise, provide a constructed response
    top_doc = documents[0]
    return (
        f"Based on {top_doc['title']} (v{top_doc['version']}, "
        f"effective {top_doc['effective_date']}), here's what I found:\n\n"
        f"{top_doc['content'][:500]}..."
        f"\n\nThis information is sourced from our internal policy repository."
    )

def query_rag(query: str) -> dict:
    """Full RAG pipeline: retrieve, generate, cite."""
    start_time = time.time()

    # Retrieve
    documents = retrieve_documents(query)

    # Generate response
    response_text = generate_response(query, documents)

    latency_ms = int((time.time() - start_time) * 1000)

    # Build citations (without full content)
    citations = []
    for doc in documents:
        citations.append({
            "doc_id": doc["doc_id"],
            "title": doc["title"],
            "version": doc["version"],
            "category": doc["category"],
            "similarity_score": round(doc["similarity_score"], 3),
        })

    return {
        "query": query,
        "response": response_text,
        "citations": citations,
        "documents_retrieved": len(documents),
        "latency_ms": latency_ms,
    }

# ─── Example Usage ─────────────────────────────────────────────

if __name__ == "__main__":
    # Initialize
    init_db()

    # Ingest sample documents
    ingest_document(
        doc_id="POL-001",
        title="Expense Reporting Policy",
        category="finance",
        content="All employees must submit expense reports within 30 days of the expense date. Expenses over $50 require itemized receipts. Expenses over $500 require manager pre-approval. International expenses must be submitted in USD using the exchange rate on the date of the expense.",
        version="3.2",
        effective_date="2026-01-01"
    )

    ingest_document(
        doc_id="POL-002",
        title="Remote Work Policy",
        category="hr",
        content="Employees may work remotely up to 3 days per week with manager approval. Remote workers must maintain a secure workspace with no shared screens visible. All company data must be accessed through the corporate VPN. Managers must approve remote work schedules quarterly.",
        version="2.1",
        effective_date="2025-06-01"
    )

    ingest_document(
        doc_id="POL-003",
        title="Data Classification Policy",
        category="security",
        content="All data is classified as Public, Internal, Confidential, or Restricted. Confidential data includes customer information, financial records, and employee data. Restricted data includes passwords, encryption keys, and trading algorithms. Confidential data must be encrypted at rest and in transit.",
        version="4.0",
        effective_date="2026-03-01"
    )

    # Query
    result = query_rag("What is the deadline for submitting expense reports?")
    print(json.dumps(result, indent=2))
```

## Extensions

1. **Document chunking:** Split long documents into chunks of 500 tokens with 50-token overlap. Store each chunk as a separate row linked to the parent document.

2. **Hybrid search:** Combine semantic search (pgvector) with keyword search (PostgreSQL full-text search using `tsvector`). Rank results using RRF (Reciprocal Rank Fusion).

3. **Document-level access control:** Add a `required_clearance` field to documents. Filter retrieved documents based on the user's clearance level.

4. **Query rewriting:** Use an LLM to rewrite the user's query into a more effective search query before embedding generation.

5. **Evaluation pipeline:** Build a golden dataset of query-document pairs. Measure recall@k and MRR (Mean Reciprocal Rank) for your retrieval system.

## Interview Relevance

RAG is the #1 topic in GenAI engineering interviews:

| Skill | Why It Matters |
|-------|---------------|
| Embedding generation | Core of semantic search |
| Vector similarity search | How retrieval works |
| Database design | pgvector schema, indexes |
| Response synthesis | Combining retrieval with generation |
| Citation tracking | Trustworthy AI in regulated environments |

**Follow-up questions:**
- "How would you handle a document that's 10,000 words long?"
- "What happens when the embedding model changes?"
- "How do you evaluate retrieval quality?"
- "How would you implement access control on retrieved documents?"
- "What's the difference between IVFFlat and HNSW indexes in pgvector?"
