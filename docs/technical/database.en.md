---
title: Database Schema
---

# Database Schema

The Watheq system relies on `PostgreSQL` as its core relational database and manages data interactions via `Prisma ORM`.

![Image suggestion: Database Entity Relationship Diagram (ERD)]

## Core Tables

### 1. `User`
Maintains user data and authorization levels.
- `role`: Defines the access privilege level (e.g., ADMIN, MANAGER, USER, VIEWER).
- `departmentId`: Links a user to a specific department for internal routing and display permissions.

### 2. `Document`
The central table in the system, storing every physical or digital transaction entered into Watheq.
- `type`: Identifies the document type (INCOMING, OUTGOING, INTERNAL).
- `status`: Transaction status (e.g., PENDING, REVIEW, APPROVED, ARCHIVED).
- `textContent`: *(Crucial for Advanced Search)* Extracted text from attachments via OCR. Operations on this field are accelerated utilizing a GIN index for massive-scale Full-Text Search (FTS).

### 3. `Attachment`
Linked to `Document` via a One-to-Many relationship.
- `fileUrl`: The path and location of the actual file saved on the storage server.
- `metadata`: Supplemental data detailing page counts, formatting, and antivirus scan outcomes.
- `hasTextExtracted`: A boolean flag indicating whether the file has been processed by the OCR Worker.

### 4. `Category` & `Department`
- **Category:** Serves thematic categorization (e.g., Financial, Administrative, Engineering) and supports sub-categories.
- **Department:** The overall departmental organizational structure applied in tracking delegations and intra-departmental authorizations.

### 5. `DisposalCertificate` & `DisposalRecord`
Dedicated objects ensuring authorized and lawful disposal and archiving of obsolete or processed documents.
- `pdfUrl`: Path to the digital, electronically signed PDF disposal certificate referencing the destroyed document's trace history and deletion motive.

## Encrypted Storage
To guarantee peak security parameters for confidential segments, highly restricted encrypted storage mechanisms can be enforced via the `isEncrypted=true` constraint on the document. Enabling this option will blind internal search engines (FTS) from indexing constraints and unilaterally prevents general user inspection without authenticating an autonomous Vault Password.

![Image suggestion: Representational icons of the core tables coupled with relationship arrows]
