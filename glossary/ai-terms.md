# AI/ML/GenAI Terms Glossary

## Embedding

**Definition**: A dense numerical vector representation of text (or other data) that captures semantic meaning. Similar texts produce similar vectors.

**Banking Example**: Banking policy documents are converted to 3072-dimensional embeddings. When a customer asks "What is the minimum down payment?", the query is embedded and compared to policy embeddings to find the most relevant documents.

**Related Concepts**: Vector Database, Similarity Search, RAG, Cosine Similarity

**Common Misunderstanding**: "Embeddings preserve the original text." They do not. An embedding is a lossy compression that captures meaning but not exact wording. You cannot reconstruct the original text from an embedding.

**Interview Relevance**: HIGH. Fundamental to understanding how RAG and semantic search work.

---

## Fine-Tuning

**Definition**: The process of further training a pre-trained model on a specific dataset to adapt it to a particular task or domain.

**Banking Example**: A base GPT-4 model is fine-tuned on the bank's mortgage policy documents, product specifications, and compliance guidelines to produce responses that are consistent with the bank's specific offerings and requirements.

**Related Concepts**: Pre-training, Transfer Learning, LoRA (Low-Rank Adaptation), Prompt Engineering

**Common Misunderstanding**: "Fine-tuning always makes the model better." Fine-tuning on a narrow dataset can cause the model to forget general knowledge (catastrophic forgetting) or overfit to the training data.

**Interview Relevance**: HIGH. Frequently discussed when deciding between fine-tuning vs. RAG for domain adaptation.

---

## Hallucination

**Definition**: When a generative AI model produces information that is factually incorrect, fabricated, or not supported by its training data or provided context.

**Banking Example**: A customer asks "What is the current 30-year fixed mortgage rate?" and the model responds "2.5%." The actual rate is 6.5%. This hallucination could lead a customer to make incorrect financial decisions and expose the bank to liability.

**Related Concepts**: Grounding, Faithfulness, Source Attribution, Fact Checking

**Common Misunderstanding**: "Hallucination means the model is lying." Models do not have intent. Hallucination is a byproduct of how LLMs generate probable text, not verified facts.

**Interview Relevance**: HIGH. Critical concern in production GenAI systems, especially in regulated domains.

---

## Prompt Engineering

**Definition**: The practice of designing and optimizing input prompts to elicit desired outputs from language models.

**Banking Example**: A well-engineered mortgage advice prompt includes: the customer's query, retrieved policy context, system instructions about compliance disclaimers, output format specifications, and examples of correct responses.

**Related Concepts**: System Prompt, Few-Shot Learning, Chain-of-Thought, RAG, Template

**Common Misunderstanding**: "Prompt engineering is just writing good questions." It involves structured template design, context management, output format specification, safety guardrails, and iterative testing against evaluation datasets.

**Interview Relevance**: HIGH. Practical skill that is often tested in GenAI engineering interviews.

---

## RAG (Retrieval-Augmented Generation)

**Definition**: An architecture that combines retrieval of relevant documents from a knowledge base with LLM generation, allowing the model to produce responses grounded in specific, up-to-date information.

**Banking Example**: When a customer asks about current mortgage rates, the RAG system: (1) embeds the query, (2) retrieves the latest rate sheet from the vector database, (3) assembles a prompt with the query and retrieved context, (4) sends to the LLM, (5) returns the grounded response with source citations.

**Related Concepts**: Embedding, Vector Database, Retrieval, Context Window, Grounding

**Common Misunderstanding**: "RAG eliminates hallucination." RAG reduces hallucination by providing context, but the model can still misinterpret, misrepresent, or fabricate information not in the retrieved documents.

**Interview Relevance**: VERY HIGH. The dominant architecture for production GenAI in enterprise settings.

---

## Token

**Definition**: A unit of text that an LLM processes. Tokens are typically sub-word units. A token is approximately 4 characters or 0.75 words in English.

**Banking Example**: A mortgage advisory prompt containing the customer's query (50 tokens), retrieved policy documents (800 tokens), and system instructions (100 tokens) totals 950 input tokens. The model's response of 300 words uses approximately 400 output tokens.

**Related Concepts**: Context Window, Token Limit, Cost Per Token, Tokenizer

**Common Misunderstanding**: "One word equals one token." Many words are split into multiple tokens (e.g., "unfortunately" might be 3 tokens), while short words can be single tokens. Tokenization depends on the model's vocabulary.

**Interview Relevance**: HIGH. Understanding tokens is essential for cost estimation, prompt optimization, and model selection.

---

## Context Window

**Definition**: The maximum number of tokens an LLM can process in a single request, including both input (prompt) and output (response).

**Banking Example**: GPT-4 Turbo has a 128,000-token context window. A mortgage advisory session with extensive customer history, 10 retrieved policy documents, and system instructions might use 15,000 tokens, leaving 113,000 tokens for the response.

**Related Concepts**: Token, Prompt Engineering, KV Cache, Sliding Window, RAG

**Common Misunderstanding**: "Larger context windows are always better." Larger contexts increase cost (more tokens processed), reduce accuracy on middle-context content (attention dilution), and increase latency.

**Interview Relevance**: HIGH. Affects system design decisions for RAG and conversation management.

---

## Temperature

**Definition**: A parameter controlling the randomness of LLM output. Higher temperature (0-2) produces more diverse and creative output. Lower temperature produces more deterministic and focused output.

**Banking Example**: For mortgage advice, temperature is set to 0.1 to ensure consistent, factual responses. For marketing content generation, temperature might be set to 0.7 to produce more creative variations.

**Related Concepts**: Top-p, Top-k, Nucleus Sampling, Deterministic Output

**Common Misunderstanding**: "Temperature 0 means completely deterministic." Temperature 0 still uses greedy decoding, which can produce different outputs if there are ties in token probabilities. For full determinism, also set a seed.

**Interview Relevance**: MODERATE. Important for understanding model behavior control.

---

## Vector Database

**Definition**: A database optimized for storing and searching high-dimensional vectors, supporting approximate nearest neighbor (ANN) search for similarity matching.

**Banking Example**: The bank's RAG system uses pgvector (PostgreSQL extension) to store 500,000 document embeddings. When a customer queries, their query is embedded and the vector database returns the 10 most similar policy documents in under 200ms.

**Related Concepts**: Embedding, ANN (Approximate Nearest Neighbor), HNSW, IVF, Cosine Similarity

**Common Misunderstanding**: "Vector databases only store vectors." They also store metadata alongside vectors and support hybrid search combining vector similarity with metadata filtering.

**Interview Relevance**: HIGH. Core component of RAG architecture.

---

## Grounding

**Definition**: The process of connecting LLM outputs to verifiable sources of information, ensuring responses are based on facts rather than model training data alone.

**Banking Example**: Every GenAI response includes citations to the specific policy documents used as context. The system verifies that claims in the response are supported by the cited sources, providing a grounding score.

**Related Concepts**: RAG, Faithfulness, Source Attribution, Hallucination Detection

**Common Misunderstanding**: "Providing context to the model means the response is grounded." The response is only grounded if it accurately uses the provided context. The model can still ignore, misinterpret, or contradict the context.

**Interview Relevance**: HIGH. Critical for production GenAI in regulated domains.

---

## Fine-tuning vs. RAG

**Definition**: Two approaches to adapting LLMs for domain-specific use. Fine-tuning modifies model weights. RAG provides external context without modifying the model.

**Banking Example**: The bank uses RAG for mortgage policy information (frequently changing, needs citations) and fine-tuning for customer service tone and style (stable, does not need citations).

| Aspect | Fine-Tuning | RAG |
|--------|------------|-----|
| Data freshness | Static (model training data) | Dynamic (updated knowledge base) |
| Citations | No | Yes |
| Cost per query | Lower | Higher (retrieval overhead) |
| One-time cost | Higher (training) | Lower (indexing documents) |
| Best for | Style, format, domain language | Facts, policies, current information |

**Interview Relevance**: VERY HIGH. Common system design interview question.
