# Take-Home Project Ideas (5)

## Project 1: Build a Mini RAG System

**Scenario**: Build a document Q&A system that retrieves relevant banking policy documents and generates answers.

**Requirements**:
1. Ingest 50+ sample banking policy documents (PDF or text)
2. Implement chunking, embedding, and vector storage
3. Build a retrieval pipeline with hybrid search
4. Generate answers using an LLM API (OpenAI, Anthropic, or open-source)
5. Include citations in responses
6. Build a simple web interface (Streamlit, FastAPI + HTML)

**Deliverables**:
- Source code with README
- Docker Compose for local setup
- 1-page architecture document
- Demo video (3-5 minutes)

**Evaluation Criteria**:
| Criterion | Weight | What We Look For |
|---|---|---|
| Code quality | 25% | Clean, well-structured, tested |
| Architecture | 20% | Reasonable component choices, clear data flow |
| Retrieval quality | 20% | Does it find relevant documents? |
| Answer quality | 15% | Are answers grounded and accurate? |
| Documentation | 10% | Clear setup instructions, architecture explanation |
| Polish | 10% | Working demo, error handling |

**Time estimate**: 8-12 hours

---

## Project 2: Build a Rate-Limited API Gateway

**Scenario**: Build an API gateway that proxies requests to a backend service with rate limiting, authentication, and usage tracking.

**Requirements**:
1. Token bucket rate limiter (configurable per client)
2. API key authentication
3. Request/response logging
4. Usage dashboard (total requests, rate limit hits, errors)
5. Graceful degradation when backend is unavailable

**Deliverables**:
- Source code
- API documentation (OpenAPI spec)
- Load test results
- README with setup instructions

**Evaluation Criteria**:
| Criterion | Weight | What We Look For |
|---|---|---|
| Rate limiter correctness | 25% | Handles bursts, distributed-safe |
| API design | 20% | RESTful, well-documented, proper HTTP semantics |
| Error handling | 20% | Graceful degradation, meaningful error messages |
| Observability | 15% | Logging, metrics, dashboard |
| Testing | 10% | Unit and integration tests |
| Performance | 10% | Handles 1000+ RPS with < 10ms overhead |

**Time estimate**: 6-10 hours

---

## Project 3: Build a Prompt Management System

**Scenario**: Build a system for managing, versioning, and A/B testing prompts for LLM applications.

**Requirements**:
1. CRUD API for prompts with versioning
2. Prompt template rendering (variable substitution)
3. A/B testing: split traffic between two prompt variants
4. Performance tracking (which variant produces better responses)
5. Simple web UI for managing prompts

**Deliverables**:
- Source code
- Database schema
- API documentation
- Demo with sample prompts

**Evaluation Criteria**:
| Criterion | Weight | What We Look For |
|---|---|---|
| Data modeling | 25% | Clean schema, proper versioning |
| API design | 20% | Intuitive endpoints, proper validation |
| A/B testing logic | 20% | Correct traffic splitting, statistical comparison |
| UI/UX | 15% | Functional, intuitive interface |
| Testing | 10% | Test coverage, edge cases |
| Documentation | 10% | Clear architecture and usage docs |

**Time estimate**: 8-12 hours

---

## Project 4: Build a Document Ingestion Pipeline

**Scenario**: Build a pipeline that ingests banking documents, chunks them, generates embeddings, and stores them in a vector database.

**Requirements**:
1. Support multiple input formats (PDF, text, HTML)
2. Configurable chunking strategies
3. Embedding generation (batch API or self-hosted)
4. Vector database storage with metadata
5. Deduplication detection
6. Incremental update support (detect and re-index changed documents)
7. Monitoring dashboard (ingestion stats, error rates)

**Deliverables**:
- Source code
- Docker Compose (includes vector DB)
- Sample documents and test data
- Architecture document
- Performance benchmarks

**Evaluation Criteria**:
| Criterion | Weight | What We Look For |
|---|---|---|
| Pipeline design | 25% | Modular, configurable, error-resistant |
| Parsing quality | 20% | Handles different formats correctly |
| Chunking strategy | 15% | Respects document structure |
| Incremental updates | 15% | Efficient change detection |
| Monitoring | 15% | Useful metrics, error alerting |
| Testing | 10% | End-to-end test with sample data |

**Time estimate**: 10-15 hours

---

## Project 5: Build a Conversation Analytics Dashboard

**Scenario**: Build a system that tracks and analyzes AI conversation data, providing insights into usage patterns, quality, and cost.

**Requirements**:
1. API to log conversation events (query, response, feedback, latency)
2. Dashboard with key metrics: query volume, satisfaction rate, cost trends
3. Query categorization (auto-classify queries by topic)
4. Identify failing queries (low satisfaction, high refinement rate)
5. Export data as CSV

**Deliverables**:
- Source code
- Database schema
- Dashboard screenshots
- Sample data generator

**Evaluation Criteria**:
| Criterion | Weight | What We Look For |
|---|---|---|
| Data modeling | 20% | Efficient schema for analytics |
| Dashboard quality | 25% | Actionable insights, clear visualizations |
| Query classification | 15% | Reasonable auto-categorization |
| API design | 15% | Efficient logging endpoint |
| Code quality | 15% | Clean, tested, well-documented |
| Insights | 10% | Does the dashboard reveal useful patterns? |

**Time estimate**: 8-12 hours

---

## General Evaluation Framework

All projects are evaluated on:

1. **Does it work?** -- Can the reviewer run it and see the core functionality?
2. **Is it well-designed?** -- Clean architecture, appropriate technology choices
3. **Is the code quality good?** -- Readable, tested, follows best practices
4. **Is it documented?** -- Setup instructions, architecture explanation
5. **Does it show depth?** -- Goes beyond the minimum requirements
6. **Does it show judgment?** -- Tradeoffs discussed, alternatives considered

## Red Flags

- Code that doesn't run out of the box
- No tests whatsoever
- Hard-coded values with no explanation
- Copy-pasted code without understanding
- Over-engineered for the scope
- No error handling

## Green Flags

- Thoughtful tradeoff discussion
- Clean commit history (not one massive commit)
- Docker Compose that actually works
- Edge case handling
- Performance consideration mentioned
- "Here's what I would do with more time" section
