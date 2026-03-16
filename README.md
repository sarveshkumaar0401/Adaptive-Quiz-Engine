# Adaptive-Quiz-Engine

# Peblo AI — Mini Content Ingestion + Adaptive Quiz Engine

Built by **Sarvesh Kumaar SB**

---

## Overview

This project is a backend system that ingests educational PDF content, generates quiz questions using an LLM, stores structured data in PostgreSQL, and serves quizzes through API endpoints with adaptive difficulty logic.

The entire pipeline is built using **n8n** (low-code workflow automation) with **Google Gemini** as the LLM provider and **PostgreSQL** as the database — all running locally via **Docker**.

---

## Architecture

```
PDF (Google Drive)
        ↓
POST /ingest (n8n Webhook)
        ↓
Extract Text (n8n PDF Extractor)
        ↓
Clean + Chunk Text (JavaScript Code Node)
        ↓
Extract Metadata — grade, subject, topic (Gemini 2.0 Flash)
        ↓
Store in PostgreSQL (source_documents + content_chunks)
        ↓
POST /generate-quiz (n8n Webhook)
        ↓
Fetch Chunks from PostgreSQL
        ↓
Generate Quiz Questions — MCQ, TrueFalse, FillBlank (Gemini 2.0 Flash)
        ↓
Store in PostgreSQL (quiz_questions)
        ↓
GET /quiz → Fetch questions by topic/difficulty
POST /submit-answer → Store answer + update adaptive difficulty
```

---

## Tech Stack

| Layer | Technology |
|---|---|
| Workflow Engine | n8n (self-hosted via Docker) |
| LLM Provider | Google Gemini 2.0 Flash |
| Database | PostgreSQL 15 (Docker) |
| API Layer | n8n Webhook Nodes |
| File Storage | Google Drive |
| API Testing | Postman |

---

## Database Schema

```sql
-- Tracks ingested PDF documents
CREATE TABLE source_documents (
  id SERIAL PRIMARY KEY,
  source_id VARCHAR(100) UNIQUE,
  filename VARCHAR(255),
  uploaded_at TIMESTAMP DEFAULT NOW()
);

-- Stores extracted text chunks from PDFs
CREATE TABLE content_chunks (
  id SERIAL PRIMARY KEY,
  source_id VARCHAR(100) REFERENCES source_documents(source_id),
  chunk_id VARCHAR(100) UNIQUE,
  grade INT,
  subject VARCHAR(100),
  topic VARCHAR(100),
  text TEXT,
  chunk_index INT,
  created_at TIMESTAMP DEFAULT NOW()
);

-- Stores generated quiz questions
CREATE TABLE quiz_questions (
  id SERIAL PRIMARY KEY,
  question_id VARCHAR(100) UNIQUE,
  source_chunk_id VARCHAR(100) REFERENCES content_chunks(chunk_id),
  question TEXT,
  type VARCHAR(20),
  options JSONB,
  answer TEXT,
  difficulty VARCHAR(20),
  created_at TIMESTAMP DEFAULT NOW()
);

-- Stores student answer submissions
CREATE TABLE student_answers (
  id SERIAL PRIMARY KEY,
  student_id VARCHAR(100),
  question_id VARCHAR(100) REFERENCES quiz_questions(question_id),
  selected_answer TEXT,
  is_correct BOOLEAN,
  submitted_at TIMESTAMP DEFAULT NOW()
);

-- Tracks adaptive difficulty per student
CREATE TABLE student_difficulty (
  id SERIAL PRIMARY KEY,
  student_id VARCHAR(100) UNIQUE,
  current_difficulty VARCHAR(20) DEFAULT 'easy',
  correct_streak INT DEFAULT 0,
  updated_at TIMESTAMP DEFAULT NOW()
);
```

---

## Setup Instructions

### Prerequisites
- Docker Desktop installed
- n8n running in Docker
- PostgreSQL running in Docker
- Google Cloud account (for Gemini API + Google Drive)
- Postman (for API testing)

### Step 1 — Clone the Repository

```bash
git clone https://github.com/sarveshkumaar/peblo-quiz-engine
cd peblo-quiz-engine
```

### Step 2 — Configure Environment Variables

```bash
cp .env.example .env
```

Fill in your values in `.env`.

### Step 3 — Start Docker Containers

```bash
# Start PostgreSQL
docker run --name postgres-kg \
  -e POSTGRES_PASSWORD=your_password \
  -p 5432:5432 -d postgres:15

# Start n8n
docker run --name n8n \
  -p 5678:5678 \
  -d n8nio/n8n
```

### Step 4 — Create Database Tables

```bash
docker exec -it postgres-kg psql -U postgres -d postgres
```

Then paste the SQL schema from the Database Schema section above.

### Step 5 — Configure n8n

1. Open n8n at `http://localhost:5678`
2. Import the workflow JSON files from the `/workflows` folder
3. Set up credentials:
   - **Google Drive OAuth2** — add your Client ID and Client Secret
   - **Google Gemini** — add your Gemini API key
   - **PostgreSQL** — Host: `postgres-kg`, Port: `5432`, Database: `postgres`

### Step 6 — Activate Workflows

In n8n, activate all 3 workflows:
- `Ingestion Workflow`
- `Quiz Generation Workflow`
- `Quiz + Submit Answer Workflow`

---

## API Endpoints

### POST /ingest
Ingests a PDF from Google Drive and stores extracted content in the database.

**URL:** `http://localhost:5678/webhook/ingest`

**Request:**
```json
{
  "file_id": "your_google_drive_file_id"
}
```

**Response:**
```json
{
  "status": "success",
  "source_id": "PEBLO_PDF_GRADE1_MATH_NUMBERS",
  "total_chunks": "1"
}
```

---

### POST /generate-quiz
Generates quiz questions from ingested content using Gemini LLM.

**URL:** `http://localhost:5678/webhook/generate-quiz`

**Request:**
```json
{
  "source_id": "PEBLO_PDF_GRADE1_MATH_NUMBERS"
}
```

**Response:**
```json
{
  "success": true
}
```

---

### GET /quiz
Retrieves quiz questions filtered by topic and difficulty.

**URL:** `http://localhost:5678/webhook/quiz`

**Query Parameters:**
- `topic` — e.g. `shapes`, `grammar`, `plants`
- `difficulty` — `easy`, `medium`, `hard`

**Example:**
```
GET http://localhost:5678/webhook/quiz?topic=shapes&difficulty=easy
```

**Response:**
```json
{
  "id": 1,
  "question_id": "PEBLO_PDF_GRADE1_MATH_NUMBERS_CH_01_Q01",
  "source_chunk_id": "PEBLO_PDF_GRADE1_MATH_NUMBERS_CH_01",
  "question": "How many sides does a square have?",
  "type": "MCQ",
  "options": ["2", "3", "4", "5"],
  "answer": "4",
  "difficulty": "easy",
  "created_at": "2026-03-16T12:42:21.197Z"
}
```

---

### POST /submit-answer
Submits a student answer and updates adaptive difficulty.

**URL:** `http://localhost:5678/webhook/submit-answer`

**Request:**
```json
{
  "student_id": "S001",
  "question_id": "PEBLO_PDF_GRADE1_MATH_NUMBERS_CH_01_Q01",
  "selected_answer": "4"
}
```

**Response:**
```json
{
  "success": true
}
```

---

## Adaptive Difficulty Logic

The system tracks each student's difficulty level and adjusts it based on their answers:

```
Correct answer:
  easy   → medium
  medium → hard
  hard   → hard (stays at max)

Wrong answer:
  hard   → medium
  medium → easy
  easy   → easy (stays at min)
```

Each student has their own difficulty record in the `student_difficulty` table which is updated after every answer submission.

---

## Sample Source IDs

After ingesting the 3 provided PDFs:

| Source ID | Subject | Grade | Topic |
|---|---|---|---|
| `PEBLO_PDF_GRADE1_MATH_NUMBERS` | Mathematics | 1 | Numbers, Counting and Shapes |
| `PEBLO_PDF_GRADE3_SCIENCE_PLANTS_ANIMALS` | Science | 3 | Plants and Animals |
| `PEBLO_PDF_GRADE4_ENGLISH_GRAMMAR` | English | 4 | Grammar and Vocabulary |

---

## Environment Variables

```env
# Google Gemini API Key
GEMINI_API_KEY=

# PostgreSQL Connection
POSTGRES_HOST=localhost
POSTGRES_PORT=5432
POSTGRES_DB=postgres
POSTGRES_USER=postgres
POSTGRES_PASSWORD=

# Google OAuth2 (for Google Drive)
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=

# n8n
N8N_PORT=5678
```

---

## Project Structure

```
peblo-quiz-engine/
├── README.md
├── .env.example
├── workflows/
│   ├── ingestion_workflow.json
│   ├── quiz_generation_workflow.json
│   └── quiz_submit_workflow.json
└── schema/
    └── init.sql
```

---

## Workflows

### 1. Ingestion Workflow (POST /ingest)
```
Webhook → Google Drive → Extract from File → Code (chunk) → Gemini (metadata) → Code (merge) → PostgreSQL (source_documents) → PostgreSQL (content_chunks) → Respond to Webhook
```

### 2. Quiz Generation Workflow (POST /generate-quiz)
```
Webhook → PostgreSQL (fetch chunks) → Gemini (generate questions) → Code (parse + structure) → PostgreSQL (store questions) → Respond to Webhook
```

### 3. Quiz + Submit Workflow (GET /quiz + POST /submit-answer)
```
GET /quiz:
Webhook → PostgreSQL (fetch questions by topic/difficulty) → Respond to Webhook

POST /submit-answer:
Webhook → PostgreSQL (fetch correct answer) → Code (check + adaptive logic) → PostgreSQL (store answer) → PostgreSQL (update difficulty) → Respond to Webhook
```
