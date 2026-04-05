# Document Processing

Source: `app/services/doc_extraction.py`

## Overview

`DocumentProcessor` extracts text from uploaded documents and splits it into overlapping chunks. These chunks become topics for the generation pipeline when `request.doc_paths` is provided. The processor handles format detection, text extraction, whitespace normalization, and boundary-aware chunking in a single `process_document()` call.

---

## Supported Formats

| Format | Extension(s) | Extraction Method | Library |
|---|---|---|---|
| PDF | `.pdf` | `fitz.open()` -- page text | PyMuPDF (fitz) |
| Word | `.doc`, `.docx` | `docx.Document()` -- paragraphs | python-docx |
| Plain text | `.txt` | `open()` -- `read()` | Built-in |

Unsupported file extensions raise `ValueError` before any extraction is attempted.

---

## Constructor Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `chunk_size` | `int` | `1000` | Maximum characters per chunk |
| `overlap` | `int` | `100` | Overlapping characters between adjacent chunks |

---

## Processing Pipeline

1. **Format Detection**: The file extension (lowercased via `Path.suffix`) determines which extraction method to call. Unsupported formats raise `ValueError`.

2. **Text Extraction**:
   - PDF: Opens via `fitz.open()`, joins text from all pages with spaces.
   - DOCX: Loads via `docx.Document()`, joins all paragraph texts with spaces.
   - TXT: Reads the entire file as UTF-8.

3. **Text Cleaning** (`_clean_text`):
   - Collapses multiple whitespace characters (`\s+`) to a single space.
   - Strips leading and trailing whitespace.

4. **Chunking** (`_chunk_text`):
   - Iterates through text in steps of `chunk_size`.
   - For each chunk boundary, looks backward for a sentence break (`.`) first, then a word break (` `).
   - If a break point is found, adjusts the chunk end to that position (inclusive of the break character).
   - Overlaps chunks by `overlap` characters (next chunk starts at `end - overlap`).
   - Filters out empty chunks after stripping.

---

## Chunking Algorithm Detail

```
Text: "First sentence. Second sentence. Third sentence. Fourth..."
chunk_size = 30, overlap = 5

Chunk 1: "First sentence." (break at period)
         Start: 0, End: 16
Chunk 2: "nce. Second sentence." (overlap from position 11)
         Start: 11, End: 33
...
```

Key behaviors:

- Prefers sentence boundaries (`.`) over word boundaries (` `). The search uses `str.rfind()` to find the last occurrence within the chunk window.
- Falls back to word boundaries if no period is found within the chunk range.
- If neither break is found, cuts at the exact `chunk_size` position.
- Overlap prevents information loss at chunk boundaries.
- The break point logic only applies when `end < len(text)` -- the final chunk always extends to the end of the text.

### Implementation

```python
def _chunk_text(self, text: str) -> List[str]:
    chunks = []
    start = 0

    while start < len(text):
        end = start + self.chunk_size

        if end < len(text):
            break_point = text.rfind('.', start, end)
            if break_point == -1:
                break_point = text.rfind(' ', start, end)
            if break_point != -1:
                end = break_point + 1

        chunks.append(text[start:end].strip())
        start = end - self.overlap

    return [chunk for chunk in chunks if chunk]
```

Source: `app/services/doc_extraction.py` -- `_chunk_text()`.

---

## Integration with Generation

When `SynthesisRequest.doc_paths` is set, both `SynthesisService` and `SynthesisLegacyService` use `DocumentProcessor` to convert documents into topic lists:

```python
processor = DocumentProcessor(chunk_size=1000, overlap=100)
for path in request.doc_paths:
    chunks = processor.process_document(path)
    topics.extend(chunks)
```

### Chunk Count vs Requested Items

| Condition | Topics Used | Questions Per Topic |
|---|---|---|
| `num_questions <= len(chunks)` | First `num_questions` chunks | 1 |
| `num_questions > len(chunks)` | All chunks | `ceil(num_questions / len(chunks))` |

When documents are the source, the output key changes from `"Seeds"` to `"Generated_From"` in the saved JSON to distinguish document-sourced generation from topic-sourced generation.

Source: `app/services/synthesis_service.py`, `app/services/synthesis_legacy_service.py`.

---

## Document Info

`get_document_info(file_path)` processes the document and returns metadata without persisting results. It calls `process_document()` internally, so the same extraction and chunking logic applies.

```json
{
  "chunk_count": 15,
  "total_chars": 14500,
  "avg_chunk_size": 966.67
}
```

| Field | Computation |
|---|---|
| `chunk_count` | `len(chunks)` |
| `total_chars` | `sum(len(chunk) for chunk in chunks)` |
| `avg_chunk_size` | `total_chars / chunk_count` (0 when no chunks) |

Source: `app/services/doc_extraction.py` -- `get_document_info()`.

---

## Error Handling

| Scenario | Behavior |
|---|---|
| Unsupported file format | Raises `ValueError` with the offending extension |
| Any processing error | Wrapped in a generic `Exception` with file path context: `"Error processing {file_path}: {message}"` |

The outer `try/except` in `process_document()` catches all exceptions -- including the `ValueError` for unsupported formats -- and re-raises them wrapped in `Exception`. This means callers always receive an `Exception` with the file path embedded in the message, regardless of the original error type.

Source: `app/services/doc_extraction.py` -- `process_document()`.
