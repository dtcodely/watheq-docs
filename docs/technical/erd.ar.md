---
title: مخطط ERD التفصيلي
---

# مخطط ERD التفصيلي — قاعدة بيانات واثق

> **المصدر**: `backend/prisma/schema.prisma` | **إجمالي النماذج**: 27 Model + 4 Enum | **قاعدة البيانات**: PostgreSQL

---

## نظرة سريعة — إحصائيات قاعدة البيانات

| البند                 | العدد |
| --------------------- | ----- |
| Models (جداول)        | 27    |
| Enums                 | 4     |
| Self Relations        | 5     |
| One-to-One Relations  | 2     |
| One-to-Many Relations | 30+   |
| Many-to-Many (Pivot)  | 3     |
| Unique Constraints    | 15+   |
| فهارس (Indexes)       | 30+   |

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
    String defaultLanguage "default en"
    Boolean sidebarCollapsed
    String signature "nullable"
    String signaturePin "nullable"
    String departmentId FK "nullable"
  }

  Department {
    String id PK
    String departmentNumber
    String deptCode "nullable"
    String name
    String nameAr "nullable"
    String managerId FK "nullable"
    String parentId FK "nullable"
    Boolean isActive
    String color "nullable"
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
    String folderId FK "nullable"
    String cabinetId FK "nullable"
    Boolean isPrivate
    Boolean isPublic
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

## 3. التخزين — الخزائن والمجلدات والمواقع الفعلية

```mermaid
erDiagram
  Cabinet {
    String id PK
    String name
    String nameAr "nullable"
    String departmentId FK "nullable"
    String parentId FK "nullable"
    DateTime deletedAt "nullable"
    Int orderIndex
  }

  Folder {
    String id PK
    String name
    String nameAr "nullable"
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
  Cabinet ||--o{ Document : "documents"
  Folder o|--o{ Folder : "children"
  Folder ||--o{ Document : "documents"
  PhysicalLocation ||--o{ Document : "documents"
```

---

## 4. الصلاحيات والأدوار

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
    String label "nullable"
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

## 5. تتبع الوثائق — المشاهدات والتحويلات والتواقيع والإصدارات

```mermaid
erDiagram
  DocumentView {
    String id PK
    String documentId FK
    String viewedById FK
    DateTime viewedAt
  }

  ForwardedDocument {
    String id PK
    String documentId FK
    String forwardedById FK
    String forwardToId FK
    String departmentId FK
  }

  DocumentSignature {
    String id PK
    String documentId FK
    String userId FK
    String signatureData
    DateTime signedAt
    String ipAddress "nullable"
    String userAgent "nullable"
  }

  DocumentVersion {
    String id PK
    String documentId FK
    Int versionNumber
    String contentSnapshot "nullable"
    String subjectSnapshot
    String changeSummary "nullable"
    String createdById FK
  }

  Document ||--o{ DocumentView : "views"
  Document ||--o{ ForwardedDocument : "forwards"
  Document ||--o{ DocumentSignature : "signatures"
  Document ||--o{ DocumentVersion : "versions"
  DocumentVersion ||--o{ Attachment : "attachments"
```

---

## 6. سير العمل والمرفقات والإشعارات

```mermaid
erDiagram
  Workflow {
    String id PK
    String name
    String documentId FK
    Status status "default PENDING"
  }

  WorkflowStep {
    String id PK
    String workflowId FK
    String name
    Int order
    Status status "default PENDING"
    String assignedToId FK
    String comment "nullable"
    DateTime completedAt "nullable"
    Boolean isEscalated
    String originalAssignedToId "nullable"
  }

  Attachment {
    String id PK
    String filename
    String originalName
    String mimeType
    Int size
    String path
    String documentId FK
    String documentVersionId FK "nullable"
  }

  Notification {
    String id PK
    String userId FK
    String type "default INFO"
    String title
    String titleAr "nullable"
    String message
    String messageAr "nullable"
    Boolean isRead
    DateTime readAt "nullable"
  }

  Document ||--o{ Workflow : "workflows"
  Workflow ||--o{ WorkflowStep : "steps"
  WorkflowStep }|--|| User : "assignedTo"
  Document ||--o{ Attachment : "attachments"
  User ||--o{ Notification : "notifications"
```

---

## 7. التصنيف والاحتفاظ والجهات الخارجية والتدقيق والإتلاف

```mermaid
erDiagram
  Category {
    String id PK
    String name
    String nameAr "nullable"
    String slug
    String type "default DOCUMENT"
    String parentId FK "nullable"
    String retentionPolicyId FK "nullable"
  }

  RetentionPolicy {
    String id PK
    String name
    String nameAr "nullable"
    Int period
    String periodUnit "default YEARS"
    RetentionAction action "default DISPOSE"
  }

  ExternalParty {
    String id PK
    String code
    String name
    String nameAr "nullable"
    String contactEmail "nullable"
    String contactPhone "nullable"
    Boolean isActive
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
    String trackingNumber
    String disposedById FK
    DateTime disposedAt
    String policyName
    String certificateId "nullable"
    String method "default SECURE_DELETE"
  }

  Category o|--o{ Category : "children"
  Category ||--o{ Document : "documents"
  RetentionPolicy ||--o{ Category : "categories"
  RetentionPolicy ||--o{ Document : "documents"
  Document ||--o{ DisposalLog : "disposalLogs"
  AuditLog o|--|| User : "user"
```

---

## تحليل العلاقات

### علاقات ذاتية (Self Relations)

| النموذج      | العلاقة               | الوصف               |
| ------------ | --------------------- | ------------------- |
| `Department` | `DepartmentHierarchy` | هيكل هرمي للأقسام   |
| `Cabinet`    | `CabinetHierarchy`    | هيكل هرمي للخزائن   |
| `Folder`     | `FolderHierarchy`     | هيكل هرمي للمجلدات  |
| `Category`   | `CategoryHierarchy`   | هيكل هرمي للتصنيفات |
| `Document`   | `DocumentRelation`    | ربط الوثائق ببعضها  |

### علاقات Many-to-Many (عبر جداول وسيطة)

| الجدول الأيسر | الجدول الأيمن | الجدول الوسيط          |
| ------------- | ------------- | ---------------------- |
| `Role`        | `Permission`  | `RolePermission`       |
| `User`        | `Role`        | `UserRole`             |
| `User`        | `Department`  | `UserDepartmentAccess` |

---

## نقاط القوة التقنية

!!! success "مميزات تصميم قاعدة البيانات" - **UUID** — جميع المعرفات الأساسية من نوع UUID (تدعم التوزيع) - **Soft Delete** — User, Cabinet, Folder, Document تدعم `deletedAt` - **Full-Text Search** — حقل `textContent` مع GIN index - **Encryption Ready** — Folder: `encryptionSalt` + `folderKeyHash`، Document: `isEncrypted` - **MFA** — مصادقة ثنائية مع backup codes - **Audit Trail** — AuditLog يسجل action + entity + entityId + ipAddress - **Electronic Signature** — توقيع إلكتروني مع PIN + IP + User-Agent - **Legal Hold** — حماية الوثائق من الإتلاف أثناء التحقيقات

!!! warning "نقاط تحسين مقترحة" - **`User.role` (String)** — ازدواجية مع `UserRole`، يُقترح الاعتماد على الجدول الوسيط فقط - **`Category.allowedRoles` (String)** — يُقترح إنشاء جدول وسيط `CategoryRole` - **`DisposalLog.method` (String)** — يُقترح تحويله إلى Enum `DisposalMethod` - **`Attachment.size` (Int)** — يُقترح `BigInt` للملفات التي تتجاوز 2GB
