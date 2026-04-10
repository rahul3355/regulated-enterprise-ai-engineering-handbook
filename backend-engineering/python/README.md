# Python for Banking GenAI Platforms

## Why Python

Python is the dominant language for GenAI platforms, data engineering, and machine learning. In a banking context, Python powers document analysis, risk scoring, regulatory reporting, and AI model serving. Its rich ecosystem (FastAPI, Pydantic, SQLAlchemy, Celery) makes it ideal for production backend services.

## Strengths

### AI/ML Ecosystem

```python
# Unmatched AI/ML library support
import openai          # OpenAI API
import anthropic       # Anthropic Claude
import langchain       # LLM orchestration
import transformers    # Hugging Face models
import torch           # Deep learning
import numpy           # Numerical computing
import pandas          # Data manipulation
import sklearn         # Machine learning
```

### Developer Productivity

```python
# Concise, readable code
from typing import Optional
from decimal import Decimal

def calculate_interest(
    principal: Decimal,
    rate: Decimal,
    days: int,
    compounding: str = 'daily',
) -> Decimal:
    """Calculate compound interest with daily/annual compounding."""
    if compounding == 'daily':
        return principal * (1 + rate / 365) ** days - principal
    return principal * (1 + rate) ** (days / 365) - principal
```

### Rapid Prototyping to Production

```
Same code works for:
- Jupyter notebook exploration
- Local script execution
- FastAPI production service
- Celery background job
- AWS Lambda function
```

## Weaknesses

### Performance

```python
# Python is interpreted, single-threaded (GIL)
# CPU-bound tasks are slow compared to Go/Java

# Solution: Use multiprocessing or offload to C extensions
from multiprocessing import Pool

def compute_risk_scores(documents):
    with Pool(processes=4) as pool:
        results = pool.map(calculate_risk_score, documents)
    return results
```

### Type Safety

```python
# Optional typing means bugs slip through
def process_payment(data):  # What type is data?
    amount = data['amount']  # KeyError if 'amount' missing

# Solution: Use Pydantic for runtime validation
from pydantic import BaseModel

class PaymentData(BaseModel):
    amount: Decimal
    currency: str
    recipient: str

def process_payment(data: PaymentData):  # Type enforced
    amount = data.amount  # Safe access
```

### Deployment Complexity

```
# Python dependency management is complex
# Virtual environments, pip, poetry, pipenv
# Native extensions (numpy, torch) are platform-specific
# Large Docker images (500MB+ with ML libraries)
```

## Typical Banking Use Cases

### GenAI Document Processing

```python
from fastapi import FastAPI, UploadFile
from pydantic import BaseModel

app = FastAPI()

class DocumentAnalysisResult(BaseModel):
    document_type: str
    extracted_fields: dict[str, str]
    confidence_scores: dict[str, float]
    risk_assessment: str

@app.post('/v1/documents/analyze', response_model=DocumentAnalysisResult)
async def analyze_document(file: UploadFile, analysis_type: str):
    """Analyze uploaded banking document (KYC, loan application, etc.)."""
    content = await file.read()

    # OCR + AI analysis
    text = await ocr_engine.process(content)
    result = await llm_client.analyze(
        text=text,
        prompt=f'Extract {analysis_type} fields from: {text}',
    )

    return DocumentAnalysisResult(
        document_type=analysis_type,
        extracted_fields=result.fields,
        confidence_scores=result.confidence,
        risk_assessment=result.risk_level,
    )
```

### Risk Scoring Engine

```python
import numpy as np
from sklearn.ensemble import GradientBoostingClassifier

class RiskScoringEngine:
    def __init__(self, model_path: str):
        self.model = GradientBoostingClassifier()
        self.model.load(model_path)

    def score_transaction(self, transaction: dict) -> dict:
        features = self.extract_features(transaction)
        probability = self.model.predict_proba([features])[0]

        return {
            'risk_score': float(probability[1]),
            'risk_level': self.classify_risk(probability[1]),
            'features_contributing': self.explain_features(features),
        }

    def classify_risk(self, score: float) -> str:
        if score < 0.2:
            return 'LOW'
        elif score < 0.6:
            return 'MEDIUM'
        return 'HIGH'
```

### Regulatory Reporting

```python
import pandas as pd
from datetime import datetime, timedelta

def generate_aml_report(date: datetime) -> pd.DataFrame:
    """Generate Anti-Money Laundering report."""
    # Query transactions for date range
    transactions = db.query(Transaction).filter(
        Transaction.date >= date - timedelta(days=30),
        Transaction.date <= date,
        Transaction.amount > 10000,  # Large transactions
    ).all()

    df = pd.DataFrame([t.to_dict() for t in transactions])

    # Flag suspicious patterns
    df['velocity'] = df.groupby('account_id')['amount'].transform('count')
    df['is_suspicious'] = (
        (df['amount'] > 50000) |
        (df['velocity'] > 10) |
        (df['country'].isin(HIGH_RISK_COUNTRIES))
    )

    return df[df['is_suspicious']]
```

### Customer Service Chatbot

```python
from langchain.chains import ConversationalRetrievalChain
from langchain.vectorstores import FAISS
from langchain.embeddings import OpenAIEmbeddings

class BankingChatbot:
    def __init__(self):
        # Load banking knowledge base
        embeddings = OpenAIEmbeddings()
        self.vectorstore = FAISS.load_local('banking_knowledge', embeddings)

        # Create conversational chain
        self.chain = ConversationalRetrievalChain.from_llm(
            llm=ChatOpenAI(model='gpt-4'),
            retriever=self.vectorstore.as_retriever(),
            memory=ConversationBufferMemory(),
        )

    def respond(self, query: str, chat_history: list) -> dict:
        result = self.chain({
            'question': query,
            'chat_history': chat_history,
        })

        return {
            'response': result['answer'],
            'sources': result.get('source_documents', []),
        }
```

## Banking-Specific Considerations

### Decimal Precision

```python
# NEVER use float for money
amount = 0.1 + 0.2  # 0.30000000000000004

# ALWAYS use Decimal
from decimal import Decimal, ROUND_HALF_UP

amount = Decimal('0.1') + Decimal('0.2')  # 0.3

# Banking rounding: round to 2 decimal places
result = amount.quantize(Decimal('0.01'), rounding=ROUND_HALF_UP)
```

### Audit Logging

```python
import logging
import json

# Structured logging for audit trail
audit_logger = logging.getLogger('audit')
audit_handler = logging.FileHandler('/var/log/banking/audit.log')
audit_logger.addHandler(audit_handler)

def log_audit_event(event: str, user_id: str, details: dict):
    audit_logger.info(json.dumps({
        'event': event,
        'user_id': user_id,
        'timestamp': datetime.utcnow().isoformat(),
        'details': details,
    }))

# Usage
log_audit_event(
    event='payment.initiated',
    user_id='USR-001',
    details={
        'payment_id': 'PAY-001',
        'amount': '1000.00',
        'currency': 'USD',
        'recipient': 'ACC-002',
    },
)
```

### Security

```python
# Input sanitization
from bleach import clean

def sanitize_input(text: str) -> str:
    """Remove potentially malicious input."""
    return clean(text, tags=[], strip=True)

# Secret management
from dotenv import load_dotenv
import os

load_dotenv()
API_KEY = os.environ['OPENAI_API_KEY']  # Never hardcode

# SQL injection prevention (always use parameterized queries)
cursor.execute(
    'SELECT * FROM accounts WHERE id = %s',
    (account_id,),  # Parameterized, not string interpolation
)
```

## Performance Optimization

```python
# Use orjson for fast JSON serialization
import orjson
serialized = orjson.dumps(data)  # 2-10x faster than json.dumps

# Use uvloop for faster async
import uvloop
asyncio.set_event_loop_policy(uvloop.EventLoopPolicy())

# Use connection pooling
from sqlalchemy import create_engine
engine = create_engine(DB_URL, pool_size=20, max_overflow=10)

# Use caching
from functools import lru_cache

@lru_cache(maxsize=1024)
def get_exchange_rate(currency_pair: str) -> Decimal:
    return fetch_rate_from_api(currency_pair)
```

## Packaging and Deployment

```toml
# pyproject.toml
[tool.poetry]
name = "banking-genai-service"
version = "1.0.0"
description = "GenAI service for banking document analysis"
authors = ["Engineering Team <engineering@bank.com>"]

[tool.poetry.dependencies]
python = "^3.11"
fastapi = "^0.110.0"
uvicorn = "^0.27.0"
pydantic = "^2.6.0"
sqlalchemy = "^2.0.0"
openai = "^1.12.0"
celery = "^5.3.0"
redis = "^5.0.0"

[tool.poetry.group.dev.dependencies]
pytest = "^8.0.0"
pytest-asyncio = "^0.23.0"
ruff = "^0.2.0"
mypy = "^1.8.0"

[build-system]
requires = ["poetry-core"]
build-backend = "poetry.core.masonry.api"
```

## Common Mistakes

1. **Using float for money**: Precision errors in financial calculations
2. **No type hints**: Code becomes unmaintainable
3. **Ignoring the GIL**: Expecting multi-threading to speed up CPU-bound work
4. **Blocking async code**: Using synchronous calls inside async functions
5. **No structured logging**: Debugging production issues without context
6. **Hardcoded secrets**: API keys and passwords in source code
7. **Large Docker images**: Not using multi-stage builds, bloated deployments

## Interview Questions

1. How do you handle decimal precision in Python for banking calculations?
2. Design a FastAPI service that processes uploaded KYC documents.
3. How do you run CPU-intensive ML inference without blocking the event loop?
4. What is the GIL and how does it affect backend service design?
5. How do you structure a Python project for deployment as a microservice?

## Cross-References

- `./fastapi.md` - Production FastAPI service structure
- `./pydantic.md` - Data validation with Pydantic
- `./async-python.md` - Asyncio patterns and event loops
- `./celery.md` - Background task processing
- `./sqlalchemy.md` - Database ORM patterns
- `../api-design/README.md` - REST API design principles
- `../backend-testing/README.md` - Testing strategies
