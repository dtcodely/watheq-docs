---
title: ERD — Database Schema
---

# ERD — Watheq Database Schema

> **Source**: `backend/prisma/schema.prisma` | **Total Models**: 27 + 4 Enums | **Database**: PostgreSQL

---

## Quick Stats

| Item                  | Count |
| --------------------- | ----- |
| Models (Tables)       | 27    |
| Enums                 | 4     |
| Self Relations        | 5     |
| One-to-One Relations  | 2     |
| One-to-Many Relations | 30+   |
| Many-to-Many (Pivot)  | 3     |
| Unique Constraints    | 15+   |
| Indexes               | 30+   |

```mermaid
erDiagram
  User {
    String id PK
    String email
    String username
    String password
    String firstName "nullable"
    String lastName "nullable"
    String role "default USER"
    Boolean isActive
    Boolean isTwoFactorEnabled
    String avatar "nullable"
    DateTime lastLoginAt "nullable"
    DateTime deletedAt "nullable"
    String defaultTheme "default system"
    String signature "nullable"
    String signaturePin "nullable"
    String departmentId FK "nullable"
  }

  Department {
    String id PK
    String departmentNumber
    String name
    String nameAr "nullable"
    String managerId FK "nullable"
    String parentId FK "nullable"
    Boolean isActive
  }

  Document {
    String id PK
    String trackingNumber
    String subject
    DocumentType type "default INTERNAL"
    Status status "default DRAFT"
    String sender
    String recipient
    String departmentId FK
    String createdById FK
    String categoryId FK "nullable"
    Boolean isEncrypted
    Boolean isLegalHold
    DateTime archivedAt "nullable"
    DateTime expiryDate "nullable"
    String textContent "nullable"
    String retentionPolicyId FK "nullable"
    DateTime deletedAt "nullable"
  }

  User ||--o{ Document : "createdBy"
  Department ||--o{ Document : "department"
  Department ||--o{ User : "users"
  Department o|--|| User : "manager"
  Department o|--o{ Department : "children"
```

---

## 3. Storage — Cabinet, Folder, PhysicalLocation

```mermaid
erDiagram
  Cabinet {
    String id PK
    String name
    String departmentId FK "nullable"
    String parentId FK "nullable"
    DateTime deletedAt "nullable"
    Int orderIndex
  }

  Folder {
    String id PK
    String name
    String cabinetId FK
    String parentId FK "nullable"
    DateTime deletedAt "nullable"
    String encryptionSalt "nullable"
    String folderKeyHash "nullable"
    Boolean isLocked
  }

  PhysicalLocation {
    String id PK
    String label
    String room
    String cabinet "nullable"
    String shelf "nullable"
    Boolean isActive
  }

  Department ||--o{ Cabinet : "cabinets"
  Cabinet o|--o{ Cabinet : "children"
  Cabinet ||--o{ Folder : "folders"
  Folder o|--o{ Folder : "children"
  PhysicalLocation ||--o{ Document : "documents"
```

---

## 4. Access Control — Roles & Permissions

```mermaid
erDiagram
  Permission {
    String id PK
    String name
    String module
    String action
  }

  Role {
    String id PK
    String name
    Int priority
    Boolean isDefault
    Boolean isActive
  }

  RolePermission {
    String id PK
    String roleId FK
    String permissionId FK
  }

  UserRole {
    String id PK
    String userId FK
    String roleId FK
  }

  UserDepartmentAccess {
    String id PK
    String userId FK
    String departmentId FK
    AccessLevel accessLevel
  }

  Role ||--o{ RolePermission : "rolePermissions"
  Permission ||--o{ RolePermission : "permissions"
  User ||--o{ UserRole : "userRoles"
  Role ||--o{ UserRole : "roleList"
  User ||--o{ UserDepartmentAccess : "departmentAccess"
  Department ||--o{ UserDepartmentAccess : "userAccess"
```

---

## 5. Document Tracking

```mermaid
erDiagram
  DocumentView {
    String id PK
    String documentId FK
    String viewedById FK
    DateTime viewedAt
  }

  DocumentSignature {
    String id PK
    String documentId FK
    String userId FK
    String signatureData
    DateTime signedAt
    String ipAddress "nullable"
  }

  DocumentVersion {
    String id PK
    String documentId FK
    Int versionNumber
    String changeSummary "nullable"
    String createdById FK
  }

  Attachment {
    String id PK
    String filename
    String mimeType
    Int size
    String documentId FK
    String documentVersionId FK "nullable"
  }

  Document ||--o{ DocumentView : "views"
  Document ||--o{ DocumentSignature : "signatures"
  Document ||--o{ DocumentVersion : "versions"
  DocumentVersion ||--o{ Attachment : "attachments"
```

---

## 6. Workflow, Notifications & Audit

```mermaid
erDiagram
  Workflow {
    String id PK
    String name
    String documentId FK
    Status status
  }

  WorkflowStep {
    String id PK
    String workflowId FK
    Int order
    Status status
    String assignedToId FK
    Boolean isEscalated
  }

  Notification {
    String id PK
    String userId FK
    String type "default INFO"
    String title
    String titleAr "nullable"
    Boolean isRead
  }

  AuditLog {
    String id PK
    String userId FK "nullable"
    String action
    String entity
    String entityId "nullable"
    Json details "nullable"
    String ipAddress "nullable"
  }

  DisposalLog {
    String id PK
    String documentId FK
    String disposedById FK
    DateTime disposedAt
    String certificateId "nullable"
    String method "default SECURE_DELETE"
  }

  Document ||--o{ Workflow : "workflows"
  Workflow ||--o{ WorkflowStep : "steps"
  User ||--o{ Notification : "notifications"
  Document ||--o{ DisposalLog : "disposalLogs"
```

---

## Relationship Summary

### Self Relations

| Model        | Relation Name         | Purpose                   |
| ------------ | --------------------- | ------------------------- |
| `Department` | `DepartmentHierarchy` | Nested department tree    |
| `Cabinet`    | `CabinetHierarchy`    | Nested cabinet tree       |
| `Folder`     | `FolderHierarchy`     | Nested folder tree        |
| `Category`   | `CategoryHierarchy`   | Nested category tree      |
| `Document`   | `DocumentRelation`    | Linking related documents |

### Many-to-Many (via Pivot Tables)

| Left   | Right        | Pivot                  |
| ------ | ------------ | ---------------------- |
| `Role` | `Permission` | `RolePermission`       |
| `User` | `Role`       | `UserRole`             |
| `User` | `Department` | `UserDepartmentAccess` |

---

## Design Highlights

!!! success "Database Design Strengths" - **UUIDs** — All primary keys are UUIDs for distribution support - **Soft Delete** — User, Cabinet, Folder, Document support `deletedAt` - **Full-Text Search** — `textContent` field with GIN index for FTS - **Encryption Ready** — Folders use `encryptionSalt` + `folderKeyHash` - **MFA** — Two-factor authentication with backup codes - **Audit Trail** — AuditLog records action + entity + entityId + ipAddress - **Electronic Signature** — Includes PIN, IP address, and User-Agent logging - **Legal Hold** — Protects documents from disposal during active investigations
