# LangChain for Production

This directory covers using LangChain and LangGraph for building production GenAI applications in a banking environment.

## Key Topics

- LangChain components for production pipelines
- LangGraph for stateful agent workflows
- When to use LangChain vs. building custom
- Production hardening patterns
- Common anti-patterns and pitfalls
- Integration with bank's existing infrastructure

## When to Use LangChain

**USE LangChain/LangGraph when:**
- Building complex multi-step workflows (chains, agents)
- Need stateful agent execution (LangGraph)
- Rapid prototyping and iteration
- Integrating multiple tools and data sources
- Team already familiar with the ecosystem

**AVOID LangChain when:**
- Simple single-prompt API calls (use direct SDK)
- Maximum performance/latency is critical
- Team needs to deeply understand every layer
- Debugging complex chain behavior is too costly
- The abstraction hides important banking-specific requirements

## Production Patterns

```python
# LangGraph for banking compliance workflow
from langgraph.graph import StateGraph, END

class ComplianceAnalysisState(TypedDict):
    query: str
    retrieved_documents: list
    risk_assessment: dict
    citations_verified: bool
    human_review_required: bool
    final_output: str

# Build compliance analysis graph
compliance_graph = StateGraph(ComplianceAnalysisState)

compliance_graph.add_node("retrieve", retrieve_documents_node)
compliance_graph.add_node("analyze", compliance_analysis_node)
compliance_graph.add_node("verify_citations", citation_verification_node)
compliance_graph.add_node("human_review", human_review_node)
compliance_graph.add_node("finalize", finalize_output_node)

compliance_graph.set_entry_point("retrieve")
compliance_graph.add_edge("retrieve", "analyze")
compliance_graph.add_edge("analyze", "verify_citations")
compliance_graph.add_conditional_edges("verify_citations",
    lambda s: "human_review" if s["risk_level"] in ["HIGH", "CRITICAL"] else "finalize")
compliance_graph.add_edge("human_review", "finalize")
compliance_graph.add_edge("finalize", END)
```

## Cross-References

- [../agents.md](../agents.md) — Agent architectures
- [../tool-calling.md](../tool-calling.md) — Tool calling patterns
- [../enterprise-genai-architecture.md](../enterprise-genai-architecture.md) — Build vs. buy decisions
- [../prompt-engineering.md](../prompt-engineering.md) — Prompt design within chains
