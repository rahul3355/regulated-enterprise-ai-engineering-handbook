# Coding Exercises (15) with Solutions

## Exercise 1: RAG Document Scorer

**Problem**: Given a query and a list of documents with BM25 scores and vector similarity scores, combine them using weighted fusion and return the top-K documents.

```python
def hybrid_score_documents(query: str, documents: list[dict], 
                            alpha: float = 0.3, k: int = 4) -> list[dict]:
    """
    Combine BM25 and vector scores using weighted fusion.
    
    Args:
        query: The search query
        documents: List of dicts with 'id', 'content', 'bm25_score', 'vector_score'
        alpha: Weight for BM25 (1-alpha for vector)
        k: Number of top documents to return
    
    Returns:
        Top-K documents sorted by combined score
    """
    if not documents:
        return []
    
    # Normalize scores to [0, 1]
    bm25_scores = [d['bm25_score'] for d in documents]
    vector_scores = [d['vector_score'] for d in documents]
    
    def normalize(scores):
        min_s, max_s = min(scores), max(scores)
        if max_s == min_s:
            return [0.5] * len(scores)
        return [(s - min_s) / (max_s - min_s) for s in scores]
    
    bm25_norm = normalize(bm25_scores)
    vector_norm = normalize(vector_scores)
    
    # Combine
    for i, doc in enumerate(documents):
        doc['combined_score'] = alpha * bm25_norm[i] + (1 - alpha) * vector_norm[i]
    
    # Sort and return top-K
    documents.sort(key=lambda d: d['combined_score'], reverse=True)
    return documents[:k]
```

## Exercise 2: Prompt Template Engine

**Problem**: Implement a simple template engine that replaces `{{variable}}` placeholders with values from a dictionary.

```python
def render_template(template: str, variables: dict) -> str:
    """
    Render a template string with variable substitution.
    
    Args:
        template: String with {{variable}} placeholders
        variables: Dict of variable names to values
    
    Returns:
        Rendered string
    
    Raises:
        KeyError: If a required variable is missing
    """
    import re
    
    def replace_match(match):
        var_name = match.group(1).strip()
        if var_name not in variables:
            raise KeyError(f"Missing variable: {var_name}")
        return str(variables[var_name])
    
    return re.sub(r'\{\{(.+?)\}\}', replace_match, template)

# Test
assert render_template("Hello, {{name}}!", {"name": "World"}) == "Hello, World!"
assert render_template("{{a}} + {{b}} = {{c}}", {"a": 1, "b": 2, "c": 3}) == "1 + 2 = 3"
```

## Exercise 3: Rate Limiter (Token Bucket)

**Problem**: Implement a token bucket rate limiter.

```python
import time

class TokenBucket:
    def __init__(self, rate: float, capacity: int):
        """
        Args:
            rate: Tokens added per second
            capacity: Maximum tokens in the bucket
        """
        self.rate = rate
        self.capacity = capacity
        self.tokens = capacity
        self.last_refill = time.time()
    
    def consume(self, tokens: int = 1) -> bool:
        """Try to consume tokens. Returns True if successful."""
        self._refill()
        if self.tokens >= tokens:
            self.tokens -= tokens
            return True
        return False
    
    def _refill(self):
        now = time.time()
        elapsed = now - self.last_refill
        new_tokens = elapsed * self.rate
        self.tokens = min(self.capacity, self.tokens + new_tokens)
        self.last_refill = now
```

## Exercise 4: Sliding Window Counter

**Problem**: Implement a sliding window rate counter that tracks requests per time window.

```python
from collections import defaultdict
import time

class SlidingWindowCounter:
    def __init__(self, window_seconds: int, max_requests: int):
        self.window = window_seconds
        self.max_requests = max_requests
        self.requests = defaultdict(list)  # key -> [timestamps]
    
    def is_allowed(self, key: str) -> bool:
        now = time.time()
        cutoff = now - self.window
        
        # Remove expired timestamps
        self.requests[key] = [t for t in self.requests[key] if t > cutoff]
        
        if len(self.requests[key]) < self.max_requests:
            self.requests[key].append(now)
            return True
        return False
    
    def get_remaining(self, key: str) -> int:
        now = time.time()
        cutoff = now - self.window
        self.requests[key] = [t for t in self.requests[key] if t > cutoff]
        return max(0, self.max_requests - len(self.requests[key]))
```

## Exercise 5: Document Deduplication

**Problem**: Given a list of documents, remove near-duplicates using Jaccard similarity on word shingles.

```python
def remove_duplicates(documents: list[str], threshold: float = 0.8, 
                      shingle_size: int = 5) -> list[str]:
    """
    Remove near-duplicate documents.
    
    Args:
        documents: List of document texts
        threshold: Jaccard similarity threshold for considering duplicates
        shingle_size: Size of word shingles
    
    Returns:
        List with duplicates removed (keeps first occurrence)
    """
    def get_shingles(text, k):
        words = text.lower().split()
        return set(' '.join(words[i:i+k]) for i in range(len(words) - k + 1))
    
    def jaccard(set1, set2):
        if not set1 and not set2:
            return 1.0
        intersection = len(set1 & set2)
        union = len(set1 | set2)
        return intersection / union if union > 0 else 0
    
    unique = []
    unique_shingles = []
    
    for doc in documents:
        shingles = get_shingles(doc, shingle_size)
        is_duplicate = False
        
        for existing_shingles in unique_shingles:
            if jaccard(shingles, existing_shingles) >= threshold:
                is_duplicate = True
                break
        
        if not is_duplicate:
            unique.append(doc)
            unique_shingles.append(shingles)
    
    return unique
```

## Exercise 6: Chunk Text by Token Count

**Problem**: Split text into chunks of approximately N words, respecting sentence boundaries.

```python
def chunk_text(text: str, max_words: int = 200, overlap: int = 30) -> list[str]:
    """
    Split text into chunks respecting sentence boundaries.
    
    Args:
        text: Input text
        max_words: Maximum words per chunk
        overlap: Number of overlapping words between chunks
    
    Returns:
        List of text chunks
    """
    import re
    
    # Split into sentences
    sentences = re.split(r'(?<=[.!?])\s+', text)
    
    chunks = []
    current_chunk = []
    current_word_count = 0
    
    for sentence in sentences:
        words = sentence.split()
        
        if current_word_count + len(words) > max_words and current_chunk:
            chunks.append(' '.join(current_chunk))
            # Start new chunk with overlap
            overlap_words = ' '.join(current_chunk[-overlap:]).split() if len(current_chunk) >= overlap else current_chunk
            current_chunk = overlap_words
            current_word_count = len(overlap_words)
        
        current_chunk.append(sentence)
        current_word_count += len(words)
    
    if current_chunk:
        chunks.append(' '.join(current_chunk))
    
    return chunks
```

## Exercise 7: LRU Cache

**Problem**: Implement a Least Recently Used (LRU) cache with O(1) get and put operations.

```python
from collections import OrderedDict

class LRUCache:
    def __init__(self, capacity: int):
        self.capacity = capacity
        self.cache = OrderedDict()
    
    def get(self, key) -> any:
        if key not in self.cache:
            return None
        self.cache.move_to_end(key)  # Mark as most recently used
        return self.cache[key]
    
    def put(self, key, value):
        if key in self.cache:
            self.cache.move_to_end(key)
        self.cache[key] = value
        if len(self.cache) > self.capacity:
            self.cache.popitem(last=False)  # Remove least recently used
```

## Exercise 8: Merge Intervals

**Problem**: Given overlapping time intervals, merge them into non-overlapping intervals.

```python
def merge_intervals(intervals: list[tuple]) -> list[tuple]:
    """
    Merge overlapping intervals.
    
    Args:
        intervals: List of (start, end) tuples
    
    Returns:
        Merged non-overlapping intervals
    """
    if not intervals:
        return []
    
    # Sort by start time
    intervals.sort(key=lambda x: x[0])
    
    merged = [intervals[0]]
    for start, end in intervals[1:]:
        last_start, last_end = merged[-1]
        if start <= last_end:  # Overlapping
            merged[-1] = (last_start, max(last_end, end))
        else:
            merged.append((start, end))
    
    return merged
```

## Exercise 9: Top-K Frequent Words

**Problem**: Find the K most frequent words in a text.

```python
from collections import Counter
import re

def top_k_frequent(text: str, k: int) -> list[tuple[str, int]]:
    """
    Find the K most frequent words.
    
    Returns:
        List of (word, count) tuples sorted by frequency descending
    """
    words = re.findall(r'\b\w+\b', text.lower())
    counter = Counter(words)
    return counter.most_common(k)
```

## Exercise 10: Validate Nested Parentheses

**Problem**: Check if a string has properly nested parentheses, brackets, and braces.

```python
def is_valid_parentheses(s: str) -> bool:
    stack = []
    mapping = {')': '(', ']': '[', '}': '{'}
    
    for char in s:
        if char in mapping.values():
            stack.append(char)
        elif char in mapping:
            if not stack or stack.pop() != mapping[char]:
                return False
    
    return len(stack) == 0
```

## Exercise 11: Binary Search

```python
def binary_search(arr: list[int], target: int) -> int:
    """Return index of target in sorted array, or -1 if not found."""
    left, right = 0, len(arr) - 1
    
    while left <= right:
        mid = (left + right) // 2
        if arr[mid] == target:
            return mid
        elif arr[mid] < target:
            left = mid + 1
        else:
            right = mid - 1
    
    return -1
```

## Exercise 12: Cosine Similarity

```python
import math

def cosine_similarity(a: list[float], b: list[float]) -> float:
    """Compute cosine similarity between two vectors."""
    dot_product = sum(x * y for x, y in zip(a, b))
    norm_a = math.sqrt(sum(x * x for x in a))
    norm_b = math.sqrt(sum(x * x for x in b))
    
    if norm_a == 0 or norm_b == 0:
        return 0.0
    
    return dot_product / (norm_a * norm_b)
```

## Exercise 13: Flatten Nested Dictionary

```python
def flatten_dict(d: dict, parent_key: str = '', sep: str = '.') -> dict:
    """Flatten a nested dictionary."""
    items = {}
    for k, v in d.items():
        new_key = f"{parent_key}{sep}{k}" if parent_key else k
        if isinstance(v, dict):
            items.update(flatten_dict(v, new_key, sep))
        else:
            items[new_key] = v
    return items
```

## Exercise 14: Circular Buffer

```python
class CircularBuffer:
    def __init__(self, capacity: int):
        self.buffer = [None] * capacity
        self.capacity = capacity
        self.head = 0
        self.size = 0
    
    def write(self, value):
        if self.size == self.capacity:
            self.head = (self.head + 1) % self.capacity
        else:
            self.size += 1
        self.buffer[(self.head + self.size - 1) % self.capacity] = value
    
    def read(self) -> list:
        return [self.buffer[(self.head + i) % self.capacity] for i in range(self.size)]
```

## Exercise 15: Retry with Exponential Backoff

```python
import time
import random

def retry_with_backoff(func, max_retries=3, base_delay=1, max_delay=60):
    """Execute a function with exponential backoff retry."""
    last_exception = None
    
    for attempt in range(max_retries + 1):
        try:
            return func()
        except Exception as e:
            last_exception = e
            if attempt < max_retries:
                delay = min(base_delay * (2 ** attempt) + random.uniform(0, 1), max_delay)
                time.sleep(delay)
    
    raise last_exception
```
