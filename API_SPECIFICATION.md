# Legal Lite — API Specification

**Base path:** `/api/v1`  
**Format:** JSON over HTTPS; multipart upload for files  
**Status:** Initial contract for hackathon MVP

## 1. Conventions

- UUIDs identify users, documents, analyses, clauses, sessions and reports.
- Timestamps use ISO 8601 UTC.
- Page numbers are 1-based.
- All document-owned resources require authentication and owner authorization.
- Large/slow operations return a resource ID and status for polling.
- Do not return storage keys or internal file paths.

### Standard success envelope

```json
{
  "data": {},
  "meta": {
    "request_id": "uuid",
    "api_version": "v1"
  }
}
```

### Standard error envelope

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "The uploaded file is not supported.",
    "details": []
  },
  "meta": {
    "request_id": "uuid"
  }
}
```

HTTP status conventions: `200` success, `201` created, `202` accepted/processing, `204` deleted, `400` malformed request, `401` unauthenticated, `403` forbidden, `404` not found, `413` too large, `415` unsupported media, `422` semantic validation, `429` rate limited, `500` unexpected server error.

## 2. Authentication

Authentication may initially be implemented using secure session cookies or bearer tokens. Do not store plaintext passwords.

| Method | Endpoint | Purpose |
|---|---|---|
| POST | `/auth/register` | Create account |
| POST | `/auth/login` | Authenticate |
| POST | `/auth/logout` | End session |
| GET | `/auth/me` | Current user |

Registration/login request fields should be validated and rate-limited. Use secure, HttpOnly, SameSite cookies if cookie sessions are selected.

## 3. Documents

### Upload
`POST /documents/upload`

Content type: `multipart/form-data`

Fields:
- `file`: PDF or DOCX.
- `language_hint`: optional `en` or `hi`.

Response: `202 Accepted`

```json
{
  "data": {
    "document_id": "uuid",
    "status": "uploaded",
    "filename": "rental-agreement.pdf"
  }
}
```

Validate file signature, size, page count and supported format. Use generated private storage names.

### List documents
`GET /documents?limit=20&cursor=...`

Return only documents owned by the authenticated user.

### Get document
`GET /documents/{document_id}`

Return metadata and processing status.

### Get extracted pages
`GET /documents/{document_id}/pages`

Optional query: `page=1`.

Page item:
```json
{
  "page_number": 1,
  "text": "Extracted page text...",
  "extraction_method": "native_pdf",
  "quality": "good"
}
```

### Delete
`DELETE /documents/{document_id}`

Delete the original and derived data according to the retention policy. Return `204`.

## 4. Analysis

### Start analysis
`POST /documents/{document_id}/analyze`

Request:
```json
{
  "languages": ["en", "hi"],
  "include_risk_observations": true,
  "inference": {
    "provider": "local",
    "cloud_consent": false
  }
}
```

Cloud processing is prohibited unless the user explicitly opts in. A local failure must not trigger silent cloud fallback.

Response: `202 Accepted`
```json
{
  "data": {
    "analysis_id": "uuid",
    "status": "queued"
  }
}
```

### Get analysis
`GET /analyses/{analysis_id}`

Possible statuses: `queued`, `processing`, `completed`, `failed`.

Completed response includes summary, clause IDs, explanations, observations and source references.

### Get clauses
`GET /analyses/{analysis_id}/clauses`

Clause object:
```json
{
  "clause_id": "uuid",
  "category": "deposit",
  "page_start": 2,
  "page_end": 2,
  "original_text": "The exact extracted clause...",
  "explanations": {
    "en": {
      "plain_language": "In simple terms...",
      "key_points": ["The tenant pays a deposit."]
    },
    "hi": {
      "plain_language": "सरल भाषा में...",
      "key_points": ["किरायेदार को सिक्योरिटी डिपॉज़िट देना होगा।"]
    }
  }
}
```

Never overwrite or paraphrase `original_text`.

## 5. Risk observations

`GET /analyses/{analysis_id}/observations`

Observation object:
```json
{
  "observation_id": "uuid",
  "clause_id": "uuid",
  "category": "contractual_ambiguity",
  "level": "medium",
  "title": "Deposit refund timing is unclear",
  "reason": "The clause does not specify a refund deadline.",
  "supporting_excerpt": "The exact excerpt from the clause.",
  "suggested_question": "Can the agreement specify a refund deadline?",
  "legal_source_ids": [],
  "confidence": "moderate"
}
```

Allowed categories: `legal_concern`, `contractual_ambiguity`, `commercial_concern`. Allowed levels: `high`, `medium`, `low`. Every observation must cite a valid clause and excerpt. These are review prompts, not legal conclusions.

## 6. Legal sources

`GET /legal-sources/{source_id}`

Return source title, publisher, URL, jurisdiction, provision identifier and verified version metadata. Do not invent effective dates.

## 7. Document-grounded chat

`POST /documents/{document_id}/chat`

Request:
```json
{
  "session_id": null,
  "message": "When can the landlord return my deposit?",
  "language": "en"
}
```

Response:
```json
{
  "data": {
    "session_id": "uuid",
    "message_id": "uuid",
    "answer": "The agreement says...",
    "insufficient_evidence": false,
    "citations": [
      {
        "type": "document_clause",
        "clause_id": "uuid",
        "page_number": 2,
        "excerpt": "..."
      }
    ]
  }
}
```

Citations may reference document clauses and retrieved legal sources. Validate all IDs against the current document and retrieval context. If evidence is insufficient, say so.

## 8. Translation

`POST /translate`

Request:
```json
{
  "text": "The original clause text",
  "source_language": "en",
  "target_language": "hi",
  "context": "rental_agreement"
}
```

Response contains translated text and an uncertainty/review indicator. Preserve names, dates, amounts, defined terms and conditions.

## 9. Compare documents (post-MVP)

`POST /documents/compare`

Request:
```json
{
  "document_ids": ["uuid", "uuid"],
  "language": "en"
}
```

Only compare documents owned by the user. Treat this as a later feature, not an MVP dependency.

## 10. Reports

### Generate report
`POST /analyses/{analysis_id}/reports`

Response: `202 Accepted` with `report_id` and status.

### Download report
`GET /reports/{report_id}/download`

Return an authorized file stream or short-lived download response. Do not expose a public permanent URL.

Report sections: document metadata, plain-language summary, clauses and explanations, attention observations, supporting excerpts, page references, legal-source references, generated date and disclaimer.

## 11. Core Pydantic models (illustrative)

```python
from datetime import datetime
from typing import Literal
from pydantic import BaseModel, Field
from uuid import UUID

class InferenceOptions(BaseModel):
    provider: Literal["local", "cloud"] = "local"
    cloud_consent: bool = False

class AnalyzeRequest(BaseModel):
    languages: list[Literal["en", "hi"]] = Field(default=["en", "hi"])
    include_risk_observations: bool = True
    inference: InferenceOptions = Field(default_factory=InferenceOptions)

class Citation(BaseModel):
    type: Literal["document_clause", "legal_source"]
    source_id: UUID
    page_number: int | None = None
    excerpt: str

class RiskObservation(BaseModel):
    observation_id: UUID
    clause_id: UUID
    category: Literal[
        "legal_concern", "contractual_ambiguity", "commercial_concern"
    ]
    level: Literal["high", "medium", "low"]
    title: str
    reason: str
    supporting_excerpt: str
    suggested_question: str
```

All model-generated data must be validated and reconciled with source records before returning it.

## 12. Authorization requirements

- Every document endpoint verifies document ownership.
- Every analysis endpoint verifies ownership through its document.
- Every chat session is scoped to its owner and document.
- Every report download checks ownership and expiry.
- Never trust a user-supplied owner ID.
- Return a consistent not-found/forbidden policy that does not leak another user's resource existence.

## 13. Acceptance tests

1. Valid PDF upload returns a document ID and queued/processing state.
2. Unsupported and oversized uploads are rejected.
3. Extracted page numbers are 1-based and stable.
4. Clause original text exactly matches the stored extraction span.
5. Invalid model category/level values are rejected.
6. Observation excerpt must exist in its referenced clause.
7. Cloud provider is never called without explicit consent.
8. Local provider failure never silently invokes cloud.
9. Chat citations must refer to the selected document or retrieved legal source.
10. Cross-user access to documents, analyses, chats and reports is rejected.
11. Deleted documents and derived resources cannot be retrieved.
12. Report downloads require authorization.

## 14. Implementation order

1. Authentication and ownership primitives.
2. Upload, document metadata and page extraction.
3. Clause segmentation and analysis status.
4. Structured explanation and Hindi output.
5. Evidence-linked observations.
6. Legal-source ingestion and RAG.
7. Document-grounded chat.
8. PDF report generation.
9. Security, evaluation and deployment hardening.
