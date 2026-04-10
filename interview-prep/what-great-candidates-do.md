# What Great Candidates Do

## The Difference Between Good and Great

Good candidates answer questions correctly. Great candidates demonstrate a deeper pattern of thinking that signals they will excel on the job. This document identifies the specific behaviors that distinguish great candidates.

## 1. They Ask Clarifying Questions Before Answering

**Good candidate**: Immediately starts answering the question with a prepared response.

**Great candidate**: "Before I dive in, let me make sure I understand the context. When you say 'design a policy search system,' is this for internal employees or external customers? What's the expected scale? Are there specific compliance requirements I should factor in?"

**Why it matters**: In real engineering work, requirements are never fully specified. Great candidates naturally seek clarification. Interviewers interpret this as real-world experience.

## 2. They Think Out Loud

**Good candidate**: Pauses for 30 seconds, then delivers a polished answer.

**Great candidate**: "Let me think through this step by step. First, I need to understand the core problem -- employees can't find policy information quickly. The root causes are probably: too many documents, poor search, and no central location. Let me think about what solutions exist..."

**Why it matters**: The interviewer can follow your thinking, identify strengths and gaps, and even guide you. It's collaborative problem-solving, not a test.

## 3. They Quantify Everything

**Good candidate**: "The system was slow, so we optimized it."

**Great candidate**: "The P95 latency was 4.2 seconds, which exceeded our 3-second SLO. We profiled and found the LLM generation was 70% of the time. We switched to gpt-4o-mini and added response caching, which brought P95 down to 1.8 seconds with a 60% cost reduction."

**Why it matters**: Numbers prove real experience. Anyone can say "I improved performance." Only someone who actually did it can cite specific metrics.

## 4. They Discuss Tradeoffs Without Being Asked

**Good candidate**: "I'd use pgvector for the vector database."

**Great candidate**: "I'd use pgvector because you already run PostgreSQL and it eliminates new infrastructure. The tradeoff is that pgvector caps out around 10-50M vectors, so if you expect to grow beyond that, you'd need to migrate. For your current scale of 5M chunks, it's the right choice. The alternative would be Pinecone for faster deployment, but that adds a new vendor dependency and ongoing cost of about $X/month."

**Why it matters**: Engineering is about making informed tradeoffs. Great candidates naturally think in tradeoffs.

## 5. They Consider Failure Modes

**Good candidate**: Describes the happy path architecture.

**Great candidate**: "Here's how it works in the normal case. Now, let me think about what can go wrong: if the LLM API times out, we have a circuit breaker that falls back to cached responses. If the vector database is down, we can degrade to full-text search. If both are down, we return a 'service unavailable' with an estimate of recovery time. Each failure mode has a different user experience impact."

**Why it matters**: Production systems spend more time handling failures than running happily. Great candidates design for failure from the start.

## 6. They Connect Technical Decisions to Business Impact

**Good candidate**: "We used a microservices architecture for better scalability."

**Great candidate**: "We chose microservices because the compliance team needed independent deployment cycles from the retail team -- they had regulatory deadlines that couldn't wait for the retail team's release schedule. The tradeoff was increased operational complexity, but the business benefit of independent deployments outweighed that cost. We measured this: deployment frequency went from monthly to weekly for compliance, while retail stayed at their comfortable monthly cadence."

**Why it matters**: Senior engineers understand that technology serves business needs. Great candidates always connect the dots.

## 7. They Admit What They Don't Know

**Good candidate**: Gives a confident but vague answer on an unfamiliar topic.

**Great candidate**: "I haven't worked directly with Milvus in production, so I can't speak to operational specifics. But from what I've studied, it's designed for billion-scale vector search with distributed architecture. The tradeoff vs. pgvector would be scale and features vs. operational simplicity. If I were evaluating it for this role, I'd run a proof-of-concept with our actual workload and compare retrieval precision, latency, and operational overhead."

**Why it matters**: Honesty is valued over bluffing. Interviewers would rather hear "I don't know, but here's how I'd figure it out" than a fabricated answer.

## 8. They Bring Banking Context Naturally

**Good candidate**: Answers the technical question in a generic way.

**Great candidate**: "In a banking context, I'd add a few things that might not be needed in a consumer application. First, every query and response needs audit logging -- not just for debugging but for regulatory compliance. Second, access control is not optional -- different employees have different clearance levels and the retrieval layer must enforce this. Third, the accuracy bar is much higher -- a hallucinated answer about loan fees can lead to regulatory violations. So I'd invest more in verification and human review than I would for a consumer chatbot."

**Why it matters**: It shows you understand the domain, not just the technology. Banking interviewers specifically look for this.

## 9. They Have Opinions (Backed by Experience)

**Good candidate**: "Both approaches are fine, it depends."

**Great candidate**: "In my experience, hybrid search (BM25 + vector) consistently outperforms pure vector search by 10-25% on retrieval precision. I've tested this across three different document collections and the result holds. So I'd always start with hybrid search unless there's a specific reason not to. The one exception is when your documents are very short (FAQs, Q&A pairs) where keyword matching dominates and vector search adds little."

**Why it matters**: Opinions based on experience signal real expertise. "It depends" on everything signals lack of conviction.

## 10. They Prepare Questions for the Interviewer

**Good candidate**: "No, I think you covered everything."

**Great candidate**: "Yes, I have a few questions. First, what's the biggest technical challenge the team is currently facing? Second, how do you measure the success of your GenAI initiatives -- is it adoption, quality, cost, or something else? Third, what's the biggest lesson you've learned from deploying AI in a banking context? And fourth, how does the engineering team balance innovation velocity with the compliance requirements that are inherent in banking?"

**Why it matters**: It shows genuine interest, strategic thinking, and helps the candidate evaluate whether the role is right for them.

## 11. They Structure Their Answers

**Good candidate**: Stream-of-consciousness answer.

**Great candidate**: "I'll approach this in three parts: first the architecture, then the tradeoffs, then the scaling strategy. For architecture, I see four main components..."

**Why it matters**: Structured thinking is easier to follow and evaluate. It also helps the candidate stay organized and not forget important points.

## 12. They Tell Stories, Not Just Facts

**Good candidate**: "I know how to optimize RAG pipelines."

**Great candidate**: "Let me tell you about a time I had to optimize a RAG pipeline. We had a policy search system with 3-second latency and the business needed it under 1.5 seconds. I profiled each component and found that the LLM generation was 60% of the time, retrieval was 25%, and re-ranking was 15%. The biggest lever was switching from gpt-4o to gpt-4o-mini, which cut generation time in half with acceptable quality impact. We also added caching for common queries, which gave us another 30% improvement on 35% of requests. End result was 1.2 seconds P95."

**Why it matters**: Stories are memorable, prove real experience, and naturally include context, decisions, tradeoffs, and results.

## How to Practice Being a Great Candidate

1. **Record yourself**: Listen back and identify weak patterns
2. **Practice with constraints**: Force yourself to include numbers, tradeoffs, and failure modes in every answer
3. **Get feedback**: Have someone who's interviewed candidates review your answers
4. **Study the role**: Understand what the day-to-day work actually involves
5. **Build opinions**: Form and articulate positions on common technology decisions
6. **Prepare stories**: Have 10-15 specific stories from your experience that demonstrate different competencies

## The Ultimate Test

After each practice interview, ask yourself:
- Did I sound like someone who has actually done this work?
- Did I demonstrate strategic thinking, not just tactical knowledge?
- Would I want to work with this person?

If the answer to all three is yes, you're ready.
