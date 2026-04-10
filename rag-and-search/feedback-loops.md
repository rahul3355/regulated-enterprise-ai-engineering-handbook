# Feedback Loops

## Why Feedback Matters

User feedback is the most valuable signal for improving RAG quality. Every interaction tells you whether the system is working correctly:
- **Thumbs up/down**: Explicit satisfaction signal
- **Click-through**: Which documents did the user actually read?
- **Query refinement**: Did the user rephrase their query? (indicates first attempt failed)
- **Time spent**: How long did they engage with the response?
- **Conversation continuation**: Did they ask follow-up questions?

## Types of Feedback Signals

### 1. Explicit Feedback

```python
from enum import Enum

class FeedbackType(Enum):
    THUMBS_UP = "thumbs_up"
    THUMBS_DOWN = "thumbs_down"
    CORRECTION = "correction"
    REPORT_ISSUE = "report_issue"

class UserFeedback:
    """Store user feedback on RAG responses."""
    
    def __init__(self, db_connection):
        self.db = db_connection
    
    def submit(self, query: str, response: str, sources: list,
               feedback_type: FeedbackType, user_id: str,
               comment: str = None) -> str:
        """Record user feedback."""
        
        feedback_id = str(uuid.uuid4())
        
        self.db.execute("""
            INSERT INTO user_feedback (
                feedback_id, query, response, sources, 
                feedback_type, user_id, comment, timestamp
            ) VALUES (%s, %s, %s, %s, %s, %s, %s, %s)
        """, (
            feedback_id, query, response, json.dumps(sources),
            feedback_type.value, user_id, comment, datetime.utcnow()
        ))
        
        return feedback_id
```

### 2. Implicit Feedback Signals

```python
class ImplicitFeedbackCollector:
    """Collect implicit signals from user behavior."""
    
    def log_query_response(self, session_id: str, query: str, 
                           response: str, sources: list) -> str:
        """Log a query-response interaction."""
        
        interaction_id = str(uuid.uuid4())
        
        self.db.execute("""
            INSERT INTO interactions (
                interaction_id, session_id, query, response, 
                sources, timestamp
            ) VALUES (%s, %s, %s, %s, %s, %s)
        """, (
            interaction_id, session_id, query, response,
            json.dumps(sources), datetime.utcnow()
        ))
        
        return interaction_id
    
    def log_source_click(self, interaction_id: str, source_index: int,
                         time_to_click: float):
        """User clicked on a cited source."""
        
        self.db.execute("""
            INSERT INTO source_clicks (interaction_id, source_index, time_to_click, timestamp)
            VALUES (%s, %s, %s, %s)
        """, (interaction_id, source_index, time_to_click, datetime.utcnow()))
    
    def log_query_refinement(self, session_id: str, original_query: str,
                             refined_query: str, time_between: float):
        """User refined their query (indicates first answer was unsatisfactory)."""
        
        self.db.execute("""
            INSERT INTO query_refinements (session_id, original_query, refined_query, time_between, timestamp)
            VALUES (%s, %s, %s, %s, %s)
        """, (session_id, original_query, refined_query, time_between, datetime.utcnow()))
    
    def log_response_view_time(self, interaction_id: str, view_duration_ms: float):
        """How long the user spent viewing the response."""
        
        self.db.execute("""
            INSERT INTO response_views (interaction_id, view_duration_ms, timestamp)
            VALUES (%s, %s, %s)
        """, (interaction_id, view_duration_ms, datetime.utcnow()))
```

### 3. Source Click-Through Analysis

```python
def analyze_source_clicks(interaction_id: str, db) -> dict:
    """Analyze which sources users clicked on."""
    
    clicks = db.query("""
        SELECT source_index, time_to_click
        FROM source_clicks
        WHERE interaction_id = %s
        ORDER BY time_to_click
    """, (interaction_id,))
    
    interaction = db.query("""
        SELECT sources FROM interactions WHERE interaction_id = %s
    """, (interaction_id,))
    
    sources = json.loads(interaction[0]["sources"])
    clicked_indices = set(c["source_index"] for c in clicks)
    
    return {
        "total_sources": len(sources),
        "clicked_sources": len(clicked_indices),
        "click_through_rate": len(clicked_indices) / len(sources),
        "first_click_index": clicks[0]["source_index"] if clicks else None,
        "clicked_source_titles": [sources[i].get("doc_title") for i in clicked_indices]
    }
```

## Using Feedback to Improve Retrieval

### Relevance Learning from Clicks

```python
class RelevanceLearner:
    """Learn relevance from user click patterns."""
    
    def __init__(self, db):
        self.db = db
    
    def build_training_pairs(self, min_clicks: int = 10) -> list[dict]:
        """Build (query, relevant_docs) pairs from click data."""
        
        # Get queries with enough engagement
        queries = self.db.query("""
            SELECT query, COUNT(*) as interaction_count
            FROM interactions
            GROUP BY query
            HAVING COUNT(*) >= %s
        """, (min_clicks,))
        
        training_pairs = []
        
        for query_data in queries:
            query = query_data["query"]
            
            # Get all interactions for this query
            interactions = self.db.query("""
                SELECT interaction_id, sources FROM interactions WHERE query = %s
            """, (query,))
            
            # Count how often each source was clicked
            source_click_counts = {}
            for interaction in interactions:
                clicks = self.db.query("""
                    SELECT source_index FROM source_clicks 
                    WHERE interaction_id = %s
                """, (interaction["interaction_id"],))
                
                sources = json.loads(interaction["sources"])
                for click in clicks:
                    source_idx = click["source_index"]
                    if source_idx < len(sources):
                        source_id = sources[source_idx].get("doc_id")
                        source_click_counts[source_id] = source_click_counts.get(source_id, 0) + 1
            
            # Sources clicked more than 50% of the time are "relevant"
            threshold = len(interactions) * 0.5
            relevant_docs = [
                doc_id for doc_id, count in source_click_counts.items()
                if count >= threshold
            ]
            
            if relevant_docs:
                training_pairs.append({
                    "query": query,
                    "relevant_docs": relevant_docs,
                    "confidence": len(interactions)
                })
        
        return training_pairs
```

### Improving Embedding Models with Feedback

```python
def create_fine_tuning_dataset(feedback_db, min_positive: int = 500) -> list:
    """Create dataset for embedding model fine-tuning."""
    
    learner = RelevanceLearner(feedback_db)
    training_pairs = learner.build_training_pairs()
    
    # Filter to high-confidence pairs
    high_confidence = [p for p in training_pairs if p["confidence"] >= 10]
    
    # Create training examples
    examples = []
    for pair in high_confidence:
        query = pair["query"]
        for doc_id in pair["relevant_docs"]:
            doc_content = get_document_content(doc_id)
            examples.append(InputExample(
                texts=[query, doc_content],
                label=1.0  # Relevant
            ))
        
        # Add negative examples (documents NOT clicked)
        all_docs = get_all_document_ids()
        negative_docs = set(all_docs) - set(pair["relevant_docs"])
        for neg_doc_id in list(negative_docs)[:3]:  # 3 negatives per positive
            neg_content = get_document_content(neg_doc_id)
            examples.append(InputExample(
                texts=[query, neg_content],
                label=0.0  # Not relevant
            ))
    
    return examples
```

### Query Failure Analysis

```python
def analyze_query_failures(db, time_window_days: int = 7) -> dict:
    """Identify queries that consistently fail."""
    
    # Find queries that were refined (user rephrased)
    refinements = db.query("""
        SELECT original_query, refined_query, COUNT(*) as refinement_count
        FROM query_refinements
        WHERE timestamp > %s
        GROUP BY original_query, refined_query
        ORDER BY refinement_count DESC
        LIMIT 100
    """, (datetime.utcnow() - timedelta(days=time_window_days),))
    
    # Find queries with low satisfaction
    low_satisfaction = db.query("""
        SELECT i.query, 
               SUM(CASE WHEN f.feedback_type = 'thumbs_up' THEN 1 ELSE 0 END) as positives,
               SUM(CASE WHEN f.feedback_type = 'thumbs_down' THEN 1 ELSE 0 END) as negatives
        FROM interactions i
        LEFT JOIN user_feedback f ON i.interaction_id = f.interaction_id
        WHERE i.timestamp > %s AND f.feedback_id IS NOT NULL
        GROUP BY i.query
        HAVING negatives > positives
        ORDER BY negatives DESC
        LIMIT 50
    """, (datetime.utcnow() - timedelta(days=time_window_days),))
    
    return {
        "frequently_refined_queries": refinements[:20],
        "low_satisfaction_queries": low_satisfaction,
        "total_refinements": len(refinements),
        "total_low_satisfaction": len(low_satisfaction),
    }
```

## Feedback Dashboard

```python
def generate_feedback_dashboard(db, days: int = 30) -> dict:
    """Generate comprehensive feedback analytics."""
    
    since = datetime.utcnow() - timedelta(days=days)
    
    # Overall satisfaction
    feedback_counts = db.query("""
        SELECT feedback_type, COUNT(*) as count
        FROM user_feedback
        WHERE timestamp > %s
        GROUP BY feedback_type
    """, (since,))
    
    total_feedback = sum(f["count"] for f in feedback_counts)
    thumbs_up = next((f["count"] for f in feedback_counts if f["feedback_type"] == "thumbs_up"), 0)
    thumbs_down = next((f["count"] for f in feedback_counts if f["feedback_type"] == "thumbs_down"), 0)
    
    # Satisfaction rate
    satisfaction_rate = thumbs_up / max(thumbs_up + thumbs_down, 1)
    
    # Query categories with satisfaction
    category_satisfaction = db.query("""
        SELECT q.category,
               SUM(CASE WHEN f.feedback_type = 'thumbs_up' THEN 1 ELSE 0 END) as positives,
               SUM(CASE WHEN f.feedback_type = 'thumbs_down' THEN 1 ELSE 0 END) as negatives
        FROM interactions i
        JOIN queries q ON i.query = q.query_text
        JOIN user_feedback f ON i.interaction_id = f.interaction_id
        WHERE i.timestamp > %s
        GROUP BY q.category
    """, (since,))
    
    # Source utilization
    source_utilization = db.query("""
        SELECT s.doc_title, s.doc_id,
               COUNT(DISTINCT c.interaction_id) as click_count,
               COUNT(DISTINCT i.interaction_id) as appearance_count
        FROM interactions i,
             jsonb_array_elements(i.sources::jsonb) with ordinality as s(source, idx)
        LEFT JOIN source_clicks c ON i.interaction_id = c.interaction_id 
                                     AND c.source_index = s.idx - 1
        WHERE i.timestamp > %s
        GROUP BY s.doc_title, s.doc_id
        ORDER BY click_count DESC
        LIMIT 20
    """, (since,))
    
    return {
        "period_days": days,
        "total_interactions": get_interaction_count(db, since),
        "total_feedback": total_feedback,
        "satisfaction_rate": satisfaction_rate,
        "thumbs_up": thumbs_up,
        "thumbs_down": thumbs_down,
        "feedback_rate": total_feedback / max(get_interaction_count(db, since), 1),
        "category_satisfaction": category_satisfaction,
        "top_used_sources": source_utilization[:10],
        "query_failures": analyze_query_failures(db),
    }
```

## Closing the Loop: Auto-Improvement

### Automatic Query Rewriting Improvement

```python
def learn_query_rewrites_from_feedback(db) -> dict:
    """Learn better query rewrites from user refinements."""
    
    # Find common refinement patterns
    refinements = db.query("""
        SELECT original_query, refined_query, COUNT(*) as count
        FROM query_refinements
        GROUP BY original_query, refined_query
        HAVING COUNT(*) >= 3
    """)
    
    rewrite_rules = {}
    for ref in refinements:
        # Detect what changed
        original = ref["original_query"]
        refined = ref["refined_query"]
        
        # Extract the pattern
        # E.g., "loan fees" -> "personal loan processing fees"
        # Pattern: "X fees" -> "X loan processing fees"
        
        rewrite_rules[original] = {
            "rewrite": refined,
            "confidence": ref["count"],
            "learned_from": "user_refinements"
        }
    
    return rewrite_rules
```

### Feedback-Driven Retrieval Tuning

```python
def tune_retrieval_parameters(feedback_data: dict) -> dict:
    """Adjust retrieval parameters based on feedback patterns."""
    
    recommendations = {}
    
    # If users frequently click source #2 or #3, increase K
    avg_first_click_position = feedback_data.get("avg_first_click_position", 1)
    if avg_first_click_position > 2:
        recommendations["increase_k"] = {
            "current": 4,
            "recommended": 6,
            "reason": f"Users frequently click sources beyond position 1 (avg: {avg_first_click_position:.1f})"
        }
    
    # If click-through rate is low, improve retrieval quality
    ctr = feedback_data.get("click_through_rate", 0)
    if ctr < 0.3:
        recommendations["improve_retrieval_quality"] = {
            "action": "add re-ranking or query rewriting",
            "reason": f"Low source CTR ({ctr:.1%}) suggests poor retrieval relevance"
        }
    
    # If satisfaction varies by department, department-specific tuning
    dept_satisfaction = feedback_data.get("department_satisfaction", {})
    low_depts = [d for d, s in dept_satisfaction.items() if s < 0.6]
    if low_depts:
        recommendations["department_specific_tuning"] = {
            "action": f"Review and improve content for: {', '.join(low_depts)}",
            "reason": "These departments have below-average satisfaction"
        }
    
    return recommendations
```

## Best Practices

1. **Collect both explicit and implicit signals**: Explicit is direct but sparse; implicit is abundant but noisy
2. **Use feedback for retrieval, not generation**: Feedback improves what you retrieve, not how you generate
3. **Act on patterns, not individual signals**: One thumbs down means nothing; 50 thumbs down on the same query means something
4. **Close the loop**: Don't just collect feedback -- use it to improve the system
5. **Track feedback trends**: Is satisfaction improving or degrading over time?
6. **Segment by user type**: Different user groups may have different satisfaction patterns
7. **Alert on drops**: If satisfaction rate drops below threshold, investigate immediately
8. **Combine with golden dataset**: Feedback reveals real-world failures; golden dataset tests known scenarios
9. **A/B test improvements**: Before rolling out retrieval changes, A/B test with feedback as the metric
10. **Respect privacy**: Anonymize feedback data; comply with data retention policies
