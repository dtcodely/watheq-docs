---
title: System Architecture
---

# System Architecture

The Watheq system relies on a modern software architecture that adopts the separation of the Client Layer from the API Layer to ensure security, scalability, and seamless development.

![Image suggestion: Architectural diagram showing system layers and data flow]

## Core Components

The project consists of the following structure:

```txt
WATHEQ/
├── backend/          # Node.js + Express + Prisma ORM
├── web/              # React + Vite + TypeScript + TailwindCSS
├── mobile/           # React Native (Expo)
├── desktop/          # Electron
├── database/         # PostgreSQL (via Docker)
└── docs-site/        # Comprehensive Documentation (MkDocs)
```

## Tech Stack

### Backend
| Technology | Role |
|---|---|
| **Node.js + Express** | REST API Server |
| **Prisma ORM** | Database Communication |
| **PostgreSQL** | Primary Database |
| **Redis + BullMQ** | Background Job Queue (OCR, FTS) |
| **JWT** | Authentication & Session Management |
| **Multer** | File Uploads |
| **Tesseract.js** | Optical Character Recognition (OCR) |
| **pdf-parse / mammoth** | Text extraction from PDF and DOCX |
| **Socket.io** | Real-time Notifications |
| **bcryptjs** | Password Encryption |

### Frontend (Web)
| Technology | Role |
|---|---|
| **React 18 + TypeScript** | UI Framework |
| **Vite** | Frontend Build Tool |
| **TailwindCSS** | Styling and Layouts |
| **react-i18next** | Localization (AR/EN) |
| **lucide-react** | Base Icons |

![Image suggestion: Logos of used technologies such as React, Node.js, PostgreSQL]

## Architecture Diagram & Data Flow

```txt
┌─────────────────────────────────────────────────┐
│                   Client Layer                  │
│  Web (React/Vite)  │  Mobile (Expo)  │ Desktop  │
└────────────────────┬────────────────────────────┘
                     │ HTTP REST + WebSocket
┌────────────────────▼────────────────────────────┐
│              API Layer (Express.js)             │
│  Auth  │  Documents  │  Users  │  Reports  ...  │
└───────────┬─────────────────────────┬───────────┘
            │ Prisma ORM              │ BullMQ
┌───────────▼──────────┐  ┌──────────▼───────────┐
│  PostgreSQL Database │  │   Redis (Job Queue)  │
│  + GIN Indexes       │  │   + Document Worker  │
└──────────────────────┘  └──────────────────────┘
```

## Backend Directory Structure (src/)

```txt
src/
├── controllers/        # HTTP Request Handlers (Endpoints)
├── services/           # Core Business Logic
│   ├── textExtraction.service.ts
│   ├── disposalCertificate.service.ts
│   └── ...
├── middleware/         # Request Interceptors (Auth, Upload)
├── queues/             # BullMQ Background Queues
├── workers/            # Background Queue Processors
├── routes/             # Route Definitions
├── utils/              # Utilities (Prisma, Redis, AI)
└── prisma/             # Database Schema & Migrations
```
