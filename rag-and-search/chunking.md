# Chunking Strategies

## Why Chunking Matters

Chunking is the first and most critical step in the RAG pipeline. Poor chunking decisions cascade through every downstream component: retrieval quality, re-ranking accuracy, and final answer groundedness all depend on how well documents are segmented.

A chunk is a contiguous segment of text that:
- Fits within the embedding model's context window (typically 512-8192 tokens)
- Represents a coherent unit of information
- Can be retrieved independently during query time

**Banking example**: A loan policy document has sections on eligibility, documentation, processing fees, and interest rates. If you chunk randomly, a query about "processing fees" might retrieve a chunk that mixes fee info with interest rate info, adding noise.

## Key Chunking Parameters

### Chunk Size

```
Small (100-200 tokens):        [Sentence-level granularity]
Medium (200-500 tokens):       [Paragraph-level granularity]  <-- Most common
Large (500-1000 tokens):       [Section-level granularity]
Very Large (1000-2000 tokens): [Multi-section granularity]
```

**Tradeoffs:**

| Chunk Size | Retrieval Precision | Context Completeness | Cost | Noise |
|---|---|---|---|---|
| Small (100-200) | High | Low | High (more embeddings) | Low |
| Medium (200-500) | Medium | Medium | Medium | Medium |
| Large (500-1000) | Lower | High | Lower (fewer embeddings) | Higher |

**Banking recommendation**: 300-500 tokens with 15% overlap for policy documents. This captures complete paragraphs while maintaining retrieval precision.

### Overlap

Overlap prevents information loss at chunk boundaries. Without overlap, a key sentence split across two chunks becomes incomplete in both.

```python
# Without overlap - information lost at boundary
Chunk 1: "The minimum processing fee for personal loans is 1.5% of"
Chunk 2: "the loan amount, subject to a maximum of $5,000."

# With 15% overlap - complete information preserved
Chunk 1: "The minimum processing fee for personal loans is 1.5% of the loan amount"
Chunk 2: "1.5% of the loan amount, subject to a maximum of $5,000."
```

**Recommended overlap**: 10-15% of chunk size. For 400-token chunks, use 40-60 token overlap.

## Chunking Strategies

### 1. Fixed-Size Chunking

Split text into chunks of a fixed token count.

```python
from langchain_text_splitters import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=400,
    chunk_overlap=60,
    separators=["\n\n", "\n", ". ", " ", ""],
    length_function=len  # or tiktoken for token count
)

chunks = splitter.split_text(document_text)
```

**Pros**: Simple, predictable, works well for uniform documents
**Cons**: May break mid-sentence; doesn't respect document structure

**Best for**: Log files, uniform policy documents, email threads

### 2. Recursive/Document-Aware Chunking

Respects document hierarchy by splitting at natural boundaries.

```python
from langchain_text_splitters import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter.from_tiktoken_encoder(
    chunk_size=400,
    chunk_overlap=60,
    separators=[
        "\n\n\n",     # Multiple newlines (section breaks)
        "\n\n",       # Paragraph breaks
        "\n",         # Line breaks
        ". ",         # Sentence ends
        "! ", "? ",   # Other sentence ends
        " ",          # Word boundaries
        ""            # Character level (last resort)
    ]
)
```

**Banking use case**: Policy documents with clear section headers. This preserves section boundaries.

**Pros**: Respects document structure; produces semantically coherent chunks
**Cons**: Chunk sizes vary; may create very small or very large chunks

### 3. Semantic Chunking

Split text based on semantic similarity between sentences. Boundaries occur where similarity drops.

```python
from langchain_experimental.text_splitter import SemanticChunker
from langchain_openai import OpenAIEmbeddings

embeddings = OpenAIEmbeddings(model="text-embedding-3-large")
chunker = SemanticChunker(
    embeddings,
    breakpoint_threshold_type="percentile",  # or "standard_deviation", "interquartile"
    breakpoint_threshold_amount=90           # split at 90th percentile distance
)

chunks = chunker.split_text(document_text)
```

**How it works**:
1. Embed each sentence
2. Calculate cosine distance between consecutive sentences
3. Find breakpoints where distance exceeds threshold
4. Group sentences between breakpoints into chunks

**Pros**: Natural semantic boundaries; adapts to content density
**Cons**: Slower (requires embedding each sentence first); threshold tuning needed
**Best for**: Complex documents with varying information density, such as annual reports

### 4. Document-Type Specific Chunking

#### Banking Policy Documents

```python
def chunk_policy_document(document: str) -> list[str]:
    """Chunk banking policy documents by section headers."""
    import re
    
    # Split on section headers like "Section 3.2:", "Article IV:", "### Eligibility"
    section_pattern = r'(?=(?:Section|Article|Chapter|Part|###)\s+[\d\w.]+:?)'
    sections = re.split(section_pattern, document)
    
    chunks = []
    for section in sections:
        if len(section.split()) < 50:  # Skip tiny sections
            continue
        # Further split large sections
        if len(section.split()) > 400:
            subsections = re.split(r'(?=(?:Subsection|\*\*|a\)|b\)|i\.))', section)
            for sub in subsections:
                if len(sub.split()) > 20:
                    chunks.append(sub.strip())
        else:
            chunks.append(section.strip())
    
    return chunks
```

#### Financial Reports

Split by table of contents structure. Financial reports have predictable patterns: executive summary, financial statements, notes, risk factors.

#### Legal Contracts

Split by numbered clauses. Legal documents have rigid structure with clause numbers.

```python
def chunk_legal_contract(document: str) -> list[str]:
    """Split contract by clause numbers."""
    clause_pattern = r'(?=\d+\.\s+[A-Z])'  # "1. Definitions", "2. Obligations"
    clauses = re.split(clause_pattern, document)
    return [c.strip() for c in clauses if len(c.strip()) > 50]
```

### 5. Table-Aware Chunking

Tables in banking documents contain critical structured data (fee schedules, interest rate tables). Standard chunking destroys table structure.

```python
def chunk_with_tables(document: str) -> list:
    """Extract tables as separate chunks, preserving structure."""
    from langchain_text_splitters import MarkdownHeaderTextSplitter
    
    # Convert document to markdown with tables
    # Tables become their own chunks with markdown formatting
    headers_to_split_on = [
        ("#", "Header 1"),
        ("##", "Header 2"),
    ]
    
    splitter = MarkdownHeaderTextSplitter(headers_to_split_on=headers_to_split_on)
    chunks = splitter.split_text(document)
    
    # Tables are preserved as markdown tables within chunks
    return chunks
```

**Table extraction approach**:
1. Detect tables using layout analysis (PDF) or HTML parsing
2. Convert to markdown or structured format
3. Embed table separately from surrounding text
4. Store as metadata: `{"type": "table", "headers": ["Fee Type", "Amount"]}`

## Advanced Techniques

### Parent-Child Chunking

Store chunks at multiple granularities. Retrieve at fine granularity but provide parent context.

```
Document: "Personal Loan Policy"
├── Parent Chunk (full document summary)
│   ├── Child Chunk 1: "Eligibility requirements: age 21+, income $30K+"
│   ├── Child Chunk 2: "Processing fees: 1.5% min, $5000 max"
│   └── Child Chunk 3: "Interest rates: 8-14% based on credit score"
```

**Retrieval**: Find the relevant child chunk via embedding search.
**Context assembly**: Return the child chunk + its parent summary for rich context.

```python
# Ingestion
for doc in documents:
    parent_chunk = summarize_document(doc)
    child_chunks = split_into_chunks(doc, chunk_size=300)
    
    # Store parent with reference to children
    store_parent(parent_chunk, doc_id=doc.id)
    
    # Store children with reference to parent
    for child in child_chunks:
        store_embedding(child, parent_id=parent_chunk.id, doc_id=doc.id)

# Retrieval
def retrieve_with_context(query, k=4):
    # Find best child chunks
    child_results = vector_search(query, k=k)
    
    # Get parent context for each child
    enriched_results = []
    for child in child_results:
        parent = get_parent(child.parent_id)
        enriched_results.append({
            "content": child.content,
            "parent_context": parent.summary,
            "metadata": child.metadata
        })
    
    return enriched_results
```

### Sliding Window Chunking

Overlap chunks more aggressively to ensure no information falls between cracks.

```python
def sliding_window_chunk(text, window_size=400, stride=200):
    """Slide a window across text with configurable stride."""
    words = text.split()
    chunks = []
    
    for i in range(0, len(words) - window_size + 1, stride):
        chunk = words[i:i + window_size]
        chunks.append(' '.join(chunk))
    
    return chunks
```

### Hierarchical/Agentic Chunking

Use an LLM to intelligently chunk documents by understanding semantic boundaries.

```python
def llm_aware_chunk(text, model="gpt-4o-mini"):
    """Use LLM to identify natural chunk boundaries."""
    prompt = f"""Split the following document into logical chunks.
    Each chunk should be a self-contained unit of information.
    Return as JSON with chunk_text and chunk_title fields.

    Document:
    {text[:4000]}"""

    # Call LLM
    response = call_llm(prompt)
    chunks = parse_json(response)
    
    return [{"title": c["chunk_title"], "content": c["chunk_text"]} for c in chunks]
```

**Pros**: Highly semantic; understands document intent
**Cons**: Expensive; slow; inconsistent outputs

## Chunking Evaluation

How do you know your chunking strategy works?

### Metrics

1. **Chunk Coherence**: Do chunks contain complete thoughts? (manual review sample)
2. **Retrieval Precision**: What % of retrieved chunks are relevant? (golden dataset)
3. **Context Sufficiency**: Do retrieved chunks contain enough info to answer?
4. **Chunk Count**: Total chunks (affects storage and search cost)

### A/B Testing Chunking Strategies

```python
strategies = {
    "fixed_300": {"chunk_size": 300, "overlap": 45, "method": "recursive"},
    "fixed_500": {"chunk_size": 500, "overlap": 75, "method": "recursive"},
    "semantic": {"method": "semantic", "threshold": 90},
    "section_aware": {"method": "regex_sections"},
}

for name, config in strategies.items():
    chunks = chunk_documents(test_documents, config)
    index_chunks(chunks, collection=f"test_{name}")
    scores = evaluate_retrieval(golden_queries, f"test_{name}")
    print(f"{name}: precision@5={scores['precision']:.3f}")
```

## Banking-Specific Chunking Guidelines

| Document Type | Chunk Size | Overlap | Strategy |
|---|---|---|---|
| Policy documents | 300-400 tokens | 15% | Section-aware recursive |
| Financial reports | 400-500 tokens | 10% | Table-aware hierarchical |
| Legal contracts | 200-300 tokens | 10% | Clause-boundary split |
| Customer FAQs | 150-200 tokens | 5% | Q&A pair (natural boundaries) |
| Training materials | 400-600 tokens | 15% | Semantic chunking |
| Email threads | 200-300 tokens | 10% | Per-message split |
| Meeting notes | 300-400 tokens | 10% | Section-aware |

## Common Mistakes

1. **Chunking too small**: Sentences split across chunks lose context. Fix: increase chunk size or overlap.
2. **Chunking too large**: Irrelevant information adds noise to retrieval. Fix: decrease chunk size.
3. **Ignoring tables**: Tables become garbled text. Fix: detect and extract tables separately.
4. **No metadata**: Losing document structure information. Fix: store section title, doc type, date.
5. **Fixed chunking for varied documents**: Annual reports and FAQs need different strategies. Fix: route by document type.
6. **Not testing**: Assuming default chunking works. Fix: evaluate with golden dataset.

## Code: Complete Chunking Pipeline

```python
from typing import List, Dict
from dataclasses import dataclass
import hashlib
import uuid

@dataclass
class Chunk:
    content: str
    metadata: Dict
    embedding: List[float] = None
    chunk_id: str = None
    
    def __post_init__(self):
        if not self.chunk_id:
            self.chunk_id = hashlib.sha256(
                self.content.encode()
            ).hexdigest()[:16]

class DocumentChunker:
    def __init__(self, embedding_model, default_chunk_size=400, 
                 default_overlap=60):
        self.embedding_model = embedding_model
        self.default_chunk_size = default_chunk_size
        self.default_overlap = default_overlap
    
    def chunk(self, document: str, metadata: Dict, 
              strategy: str = "recursive") -> List[Chunk]:
        """Full pipeline: split -> embed -> create chunks."""
        
        # Step 1: Split
        if strategy == "recursive":
            raw_chunks = self._recursive_split(document)
        elif strategy == "semantic":
            raw_chunks = self._semantic_split(document)
        elif strategy == "section_aware":
            raw_chunks = self._section_split(document)
        else:
            raw_chunks = self._recursive_split(document)
        
        # Step 2: Enrich metadata
        chunks = []
        for chunk in raw_chunks:
            chunk_meta = {
                **metadata,
                "chunk_length": len(chunk.split()),
                "doc_hash": hashlib.sha256(
                    document.encode()
                ).hexdigest()[:8]
            }
            chunks.append(
                Chunk(content=chunk, metadata=chunk_meta)
            )
        
        # Step 3: Embed (batched)
        contents = [c.content for c in chunks]
        embeddings = self.embedding_model.encode_batch(contents)
        for chunk, emb in zip(chunks, embeddings):
            chunk.embedding = emb
        
        return chunks
    
    def _recursive_split(self, text: str) -> List[str]:
        """Recursive character splitting."""
        splitter = RecursiveCharacterTextSplitter(
            chunk_size=self.default_chunk_size,
            chunk_overlap=self.default_overlap,
            separators=["\n\n", "\n", ". ", " ", ""]
        )
        return splitter.split_text(text)
```
