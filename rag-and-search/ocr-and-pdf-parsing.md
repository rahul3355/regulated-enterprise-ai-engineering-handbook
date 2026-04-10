# OCR and PDF Parsing

## Why OCR Matters in Banking

Banking documents come in many formats, many of which are scanned images or image-based PDFs:
- **Scanned loan applications**: Handwritten signatures, stamped forms
- **Regulatory filings**: Historical documents scanned from paper
- **Customer correspondence**: Letters, checks, deposit slips
- **Identity documents**: Passports, driver's licenses, utility bills
- **Board meeting minutes**: Scanned signed documents

Without OCR, these documents are invisible to your RAG system.

## PDF Types

| PDF Type | Description | Extraction Method |
|---|---|---|
| **Text-based PDF** | Contains actual text layer | Direct text extraction |
| **Image-based PDF** | Scanned pages as images | OCR required |
| **Hybrid PDF** | Text + embedded images | Text extraction + selective OCR |
| **Form PDF** | Fillable form fields | Form field extraction |

## OCR Engines

### Tesseract (Open Source)

```python
import pytesseract
from PIL import Image
import pdf2image

def ocr_pdf(pdf_path: str, lang: str = "eng") -> str:
    """OCR a scanned PDF using Tesseract."""
    
    # Convert PDF pages to images
    images = pdf2image.convert_from_path(pdf_path, dpi=300)
    
    all_text = []
    for i, image in enumerate(images):
        # OCR each page
        text = pytesseract.image_to_string(image, lang=lang)
        all_text.append(f"[Page {i+1}]\n{text}")
    
    return "\n\n".join(all_text)

def ocr_with_layout(pdf_path: str) -> str:
    """OCR preserving layout information."""
    
    images = pdf2image.convert_from_path(pdf_path, dpi=300)
    
    all_pages = []
    for i, image in enumerate(images):
        # Get OSD (orientation, script, detection) data
        ocr_data = pytesseract.image_to_data(image, output_type=pytesseract.Output.DICT)
        
        # Get structured text with bounding boxes
        hocr = pytesseract.image_to_pdf_or_hocr(image, extension="hocr")
        
        all_pages.append({
            "page": i + 1,
            "text": pytesseract.image_to_string(image),
            "confidence": np.mean([int(c) for c in ocr_data["conf"] if int(c) > 0]),
        })
    
    return all_pages
```

### AWS Textract

```python
import boto3

def textract_extract(pdf_path: str) -> dict:
    """Extract text, tables, and forms using AWS Textract."""
    
    textract = boto3.client("textract", region_name="us-east-1")
    
    with open(pdf_path, "rb") as f:
        pdf_bytes = f.read()
    
    # Analyze document (text + tables + forms)
    response = textract.analyze_document(
        Document={"Bytes": pdf_bytes},
        FeatureTypes=["TABLES", "FORMS", "SIGNATURES"]
    )
    
    # Extract blocks
    text_blocks = []
    tables = []
    form_fields = {}
    
    for block in response["Blocks"]:
        if block["BlockType"] == "LINE":
            text_blocks.append(block["Text"])
        elif block["BlockType"] == "TABLE":
            tables.append(extract_table(block, response["Blocks"]))
        elif block["BlockType"] == "KEY_VALUE_SET":
            if "KEY" in block.get("EntityTypes", []):
                key = block.get("Text", "")
            elif "VALUE" in block.get("EntityTypes", []):
                form_fields[key] = block.get("Text", "")
    
    return {
        "text": "\n".join(text_blocks),
        "tables": tables,
        "form_fields": form_fields,
        "signatures": [b for b in response["Blocks"] if b["BlockType"] == "SIGNATURE"],
    }
```

### Google Document AI

```python
from google.cloud import documentai_v1

def document_ai_parse(pdf_path: str, project_id: str, location: str) -> dict:
    """Parse document using Google Document AI."""
    
    client = documentai_v1.DocumentProcessorServiceClient()
    processor_name = f"projects/{project_id}/locations/{location}/processors/processor-id"
    
    with open(pdf_path, "rb") as f:
        content = f.read()
    
    document = {"content": content, "mime_type": "application/pdf"}
    request = {"name": processor_name, "raw_document": document}
    
    result = client.process_document(request=request)
    document_text = result.document.text
    
    # Extract entities
    entities = []
    for entity in result.document.entities:
        entities.append({
            "type": entity.type_,
            "mention": entity.mention_text,
            "confidence": entity.confidence,
        })
    
    return {
        "text": document_text,
        "entities": entities,
        "pages": len(result.document.pages),
    }
```

## Table Extraction

Tables in banking documents contain critical structured data: fee schedules, interest rate tables, amortization schedules.

### PDF Table Extraction with Camelot

```python
import camelot

def extract_tables_from_pdf(pdf_path: str) -> list:
    """Extract tables from PDF using Camelot."""
    
    # Camelot works best with text-based PDFs
    tables = camelot.read_pdf(pdf_path, flavor="lattice", pages="all")
    
    extracted = []
    for i, table in enumerate(tables):
        df = table.df
        
        # Convert to markdown for embedding
        markdown_table = df.to_markdown(index=False)
        
        extracted.append({
            "page": table.parsing_report["page"],
            "accuracy": table.parsing_report["accuracy"],
            "data": df.to_dict("records"),
            "markdown": markdown_table,
            "headers": list(df.columns),
        })
    
    return extracted
```

### Table Chunking Strategy

```python
def chunk_table_for_embedding(table_markdown: str, context: str) -> list[dict]:
    """Chunk a table into row-level pieces with context."""
    
    import pandas as pd
    from io import StringIO
    
    df = pd.read_table(StringIO(table_markdown), sep="|")
    
    chunks = []
    
    # Chunk 1: Table header + summary
    header_chunk = {
        "content": f"Table: {context}\nHeaders: {', '.join(df.columns)}\n"
                   f"This table has {len(df)} rows and {len(df.columns)} columns.",
        "metadata": {"chunk_type": "table_summary", "row_range": "all"}
    }
    chunks.append(header_chunk)
    
    # Chunk 2+: Individual rows (or groups of rows for large tables)
    for i, row in df.iterrows():
        row_text = f"Table: {context}\n"
        for col in df.columns:
            row_text += f"{col}: {row[col]}\n"
        
        chunks.append({
            "content": row_text,
            "metadata": {
                "chunk_type": "table_row",
                "row_index": i,
                "row_range": f"{i}-{i}"
            }
        })
    
    return chunks
```

## Complete PDF Processing Pipeline

```python
class PDFProcessor:
    def __init__(self, ocr_engine: str = "tesseract", extract_tables: bool = True):
        self.ocr_engine = ocr_engine
        self.extract_tables = extract_tables
    
    def process(self, pdf_path: str) -> dict:
        """Process a PDF document end-to-end."""
        
        # Step 1: Detect PDF type
        pdf_type = self.detect_pdf_type(pdf_path)
        
        result = {
            "pdf_type": pdf_type,
            "text": "",
            "tables": [],
            "images": [],
            "metadata": {},
        }
        
        # Step 2: Extract based on type
        if pdf_type == "text":
            result["text"] = self.extract_text(pdf_path)
        elif pdf_type == "scanned":
            result["text"] = self.ocr_extract(pdf_path)
        elif pdf_type == "hybrid":
            result["text"] = self.extract_text(pdf_path)
            ocr_text = self.ocr_extract_images_only(pdf_path)
            result["text"] += "\n\n" + ocr_text
        
        # Step 3: Extract tables
        if self.extract_tables:
            result["tables"] = self.extract_tables_from_pdf(pdf_path)
        
        # Step 4: Extract metadata
        result["metadata"] = self.extract_metadata(pdf_path)
        
        # Step 5: Extract images
        result["images"] = self.extract_images(pdf_path)
        
        return result
    
    def detect_pdf_type(self, pdf_path: str) -> str:
        """Detect if PDF is text-based, scanned, or hybrid."""
        from pypdf import PdfReader
        
        reader = PdfReader(pdf_path)
        has_text = False
        has_images = False
        
        for page in reader.pages:
            text = page.extract_text()
            if text and len(text.strip()) > 10:
                has_text = True
            
            resources = page.get("/Resources")
            if resources and "/XObject" in resources:
                has_images = True
        
        if has_text and not has_images:
            return "text"
        elif has_images and not has_text:
            return "scanned"
        else:
            return "hybrid"
    
    def extract_text(self, pdf_path: str) -> str:
        """Extract text from text-based PDF."""
        from pypdf import PdfReader
        
        reader = PdfReader(pdf_path)
        text_parts = []
        
        for i, page in enumerate(reader.pages):
            text = page.extract_text()
            if text:
                text_parts.append(f"[Page {i+1}]\n{text}")
        
        return "\n\n".join(text_parts)
    
    def ocr_extract(self, pdf_path: str) -> str:
        """OCR a scanned PDF."""
        images = pdf2image.convert_from_path(pdf_path, dpi=300)
        
        all_text = []
        for i, image in enumerate(images):
            text = pytesseract.image_to_string(image)
            all_text.append(f"[Page {i+1}]\n{text}")
        
        return "\n\n".join(all_text)
```

## Image Processing for Banking Documents

### Check and ID Processing

```python
import cv2
import numpy as np

def preprocess_banking_document(image_path: str) -> np.ndarray:
    """Preprocess banking document image for better OCR."""
    
    image = cv2.imread(image_path)
    
    # Convert to grayscale
    gray = cv2.cvtColor(image, cv2.COLOR_BGR2GRAY)
    
    # Denoise
    denoised = cv2.fastNlMeansDenoising(gray, h=30)
    
    # Threshold (binarize)
    _, binary = cv2.threshold(denoised, 0, 255, cv2.THRESH_BINARY + cv2.THRESH_OTSU)
    
    # Deskew
    angle = get_skew_angle(binary)
    if abs(angle) > 0.5:
        binary = rotate_image(binary, angle)
    
    return binary

def get_skew_angle(image: np.ndarray) -> float:
    """Detect skew angle of document."""
    coords = np.column_stack(np.where(image > 0))
    angle = cv2.minAreaRect(coords)[-1]
    
    if angle < -45:
        angle = -(90 + angle)
    else:
        angle = -angle
    
    return angle
```

## Quality Checks

```python
def ocr_quality_check(text: str, expected_pages: int) -> dict:
    """Check OCR output quality."""
    
    checks = {}
    
    # Check 1: Text length
    checks["text_length"] = len(text)
    checks["has_content"] = len(text.strip()) > 50
    
    # Check 2: Page count match
    actual_pages = len(re.findall(r'\[Page \d+\]', text))
    checks["expected_pages"] = expected_pages
    checks["actual_pages"] = actual_pages
    checks["page_match"] = actual_pages == expected_pages
    
    # Check 3: Garbled text detection
    # High ratio of special characters may indicate poor OCR
    special_chars = sum(1 for c in text if not c.isalnum() and not c.isspace() and c not in '.,;:!?-()[]{}"\'/\n')
    checks["special_char_ratio"] = special_chars / max(len(text), 1)
    checks["quality_ok"] = checks["special_char_ratio"] < 0.1
    
    # Check 4: Common banking terms presence
    banking_terms = ["loan", "account", "balance", "fee", "interest", "rate", "date", "amount"]
    found_terms = [t for t in banking_terms if t in text.lower()]
    checks["banking_terms_found"] = found_terms
    checks["has_banking_context"] = len(found_terms) >= 2
    
    return checks
```

## Best Practices

1. **Use 300 DPI minimum**: Lower DPI significantly reduces OCR accuracy
2. **Preprocess images**: Denoise, deskew, and binarize before OCR
3. **Combine text extraction + OCR**: Handle hybrid PDFs correctly
4. **Validate OCR output**: Check for garbled text and low confidence scores
5. **Extract tables separately**: Tables need special handling for RAG
6. **Track source format**: Know which documents required OCR vs. text extraction
7. **Use cloud OCR for production**: Textract or Document AI are more accurate than Tesseract
8. **Language support**: Configure OCR for all languages your bank operates in
9. **Handle forms specially**: Form fields need key-value extraction, not just text
