---
title: API Reference
---

# API Reference

All backend API routes are prefixed with the base path `/api`.
Interacting with protected endpoints requires providing a valid authentication token (JWT) inside the request headers.

```http
Authorization: Bearer <token>
```

![Image suggestion: Diagram illustrating JWT authentication flow and data exchange between client and server]

---

## 1. Authentication (`/api/auth`)

| Endpoint | Method | Description |
|---|---|---|
| `/register` | POST | Registers a new user. (Restricted to higher privilege users like ADMIN or MANAGER) |
| `/login` | POST | Submits user credentials and returns a valid JWT session token |
| `/me` | GET | Fetches full profile details and contextual data of the currently authenticated user |

---

## 2. Documents Management (`/api/documents`)

The primary route controller for retrieving, altering, and handling documents across the system:

| Endpoint | Method | Description |
|---|---|---|
| `/` | GET | Fetches a paginated list of documents. Flexibly supports query filters (e.g., search, fullText, type, status, departmentId). |
| `/` | POST | Creates a new document. The request payload must utilize `FormData` if attachments are incorporated. |
| `/:id` | GET | Retrieves the absolute and comprehensive details referencing a specific document. |
| `/:id` | PATCH | Submits partial updates to modify document attributes (e.g., status alterations, trace adjustments, assignment changes). |
| `/:id` | DELETE | Permanently deletes a document (subject to system compliance policies). |

---

## 3. Disposals & Destruction Lifecycle (`/api/disposals`)

Dedicated routes concerning the authorized and phased physical/digital destruction procedures:

| Endpoint | Method | Description |
|---|---|---|
| `/` | POST | Initiates a disposal protocol on targeted expired documents. (Generates a `DisposalAction` record). |
| `/:id/certify` | POST | Finalizes destruction by rendering a non-reputable PDF Disposal Certificate, appended with timestamps and organizational stamps. |

---

## 4. Advanced Full-Text Search (FTS) Phase

Massive-scale, content-aware deep searching within OCR-extracted payload texts can be triggered by injecting the following query parameters into the fetch route `GET /api/documents`:

- `search`: The target string, identifier, or keyword clause required.
- `fullText=true`: An optional explicit flag telling the database engine to sweep through the `textContent` fields.

**Response Example (Enhanced Snippet Generator):**
The server resolves matches intelligently by presenting the circumjacent text containing the query.
```json
{
  "data": [
    {
      "id": "123",
      "subject": "Hardware Supply Contract",
      "textContent": "... per agreement, hardware supply contract was verified and stamped by the department ...",
      "snippet": "... per agreement, hardware supply contract was verified and stamped ..." 
    }
  ],
  "total": 1
}
```

![Image suggestion: Postman or similar REST client screenshot demonstrating a search response with the highlighted snippet]
