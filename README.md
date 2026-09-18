# CollabSphere — Smart Collaboration & AI Career Guidance System

CollabSphere is a full-stack web platform built for students and mentors to collaborate in real-time study rooms, organize tasks using Kanban boards, book mentor sessions with payment integration, and receive AI-powered career counseling via an in-house **RAG (Retrieval-Augmented Generation)** bot.

---

## 📖 Table of Contents
1. [Project Concept & What It Does](#-project-concept--what-it-does)
2. [Technology Stack Clarification](#-technology-stack-clarification)
3. [Page-to-Page Navigation & Architecture Flow](#-page-to-page-navigation--architecture-flow)
4. [AI Career Guidance: Why Lightweight Lookup & RAG Lookup?](#-ai-career-guidance-why-lightweight-lookup--rag-lookup)
5. [Environment Variables (.env) Explained](#-environment-variables-env-explained)
6. [Step-by-Step Guide to Run the Project (For Collaborators)](#-step-by-step-guide-to-run-the-project-for-collaborators)
7. [API Endpoints Summary](#-api-endpoints-summary)

---

## 🎯 1. Project Concept & What It Does

CollabSphere bridges the gap between peer group study and personalized career mentorship:
- **Role-Based Workspaces**: Separate interfaces and workflows for **Students** and **Mentors**.
- **Virtual Collaborative Rooms**: Students create study groups with real-time chat, shared study notes/files, and an interactive Kanban board (To Do, In Progress, Done).
- **Mentor Marketplace & Booking**: Mentors set availability and fees; students discover mentors, request 1-on-1 sessions, and complete bookings via Razorpay escrow.
- **RAG-Powered AI Career Counselor**: Instant guidance on streams, degrees, skills, and careers using domain-specific knowledge (`career_docs.txt`) combined with Groq Llama 3.1.

---

## ⚙️ 2. Technology Stack Clarification

| Layer | Technology | Role & Purpose |
|---|---|---|
| **Frontend** | **React 19 + Vite + TailwindCSS** | Single Page Application (SPA) running in the Node.js runtime. Fast hot-reload, Lucide icons, responsive UI. |
| **Routing** | **React Router DOM v7** | Client-side page navigation (`/login`, `/student`, `/mentor`, `/room/:id`). |
| **Backend** | **Python Flask + Blueprints** | Modular REST API. Blueprints act exactly like Express Routers (`app.register_blueprint`). |
| **Database** | **SQLAlchemy + SQLite / PostgreSQL** | Local development defaults to zero-config SQLite (`collab.db`). Can switch to PostgreSQL anytime. |
| **AI / NLP** | **TF-IDF Retriever + Groq API** | Lightweight in-memory document retrieval + Llama 3.1-8b LLM. |
| **Payments** | **Razorpay Gateway** | Order creation, payment verification, and escrow payouts. |

> **Note on Node & Express**: The Node.js environment powers our frontend build and dev server through **Vite**. The backend API is built in **Python Flask**. Flask Blueprints serve the same role as Express Routers (e.g., `user_bp`, `room_bp`, `chat_bp`, `auth_bp`).

---

## 🗺️ 3. Page-to-Page Navigation & Architecture Flow

```
                     ┌───────────────┐
                     │   / (Root)    │
                     └───────┬───────┘
                             │ (Redirects)
                             ▼
                     ┌───────────────┐
                     │    /login     │ ◄─── (AuthContext: stores JWT in localStorage)
                     └───────┬───────┘
             ┌───────────────┴───────────────┐
             │ (If role == 'student')         │ (If role == 'mentor')
             ▼                               ▼
    ┌──────────────────┐           ┌──────────────────┐
    │ /student         │           │ /mentor          │
    │ (StudentDashboard)│          │ (MentorDashboard)│
    └────────┬─────────┘           └────────┬─────────┘
             │                              ├── Pending Requests
             ├── 1. Dashboard Tab           ├── Booking Schedules
             ├── 2. Create Room Tab         └── Razorpay Payouts
             ├── 3. Room Workspace (/room/:id)
             │      ├── Group Chat
             │      └── Kanban Task Board
             ├── 4. Find Mentor Tab & Payment Modal
             └── 5. AI Career Bot Tab (RAG)
```

### Complete User Journey:
1. **Login & Registration** (`frontend/src/pages/Login.jsx`):
   - Hits `POST /api/auth/login` or `POST /api/auth/register`.
   - On success, saves JWT access token and user payload into `localStorage`.
   - Redirects students to `/student` and mentors to `/mentor`.
2. **Student Dashboard** (`frontend/src/pages/StudentDashboard.jsx`):
   - **DashboardTab**: Displays active and joined study rooms fetched from `GET /api/rooms`.
   - **CreateRoomTab**: Creates new study rooms via `POST /api/rooms`.
   - **Room Workspace** (`frontend/src/components/student/RoomDetailView.jsx`): Routed via `/room/:id`. Contains live polling chat (`/api/rooms/<id>/messages`) and Kanban task tracker (`/api/rooms/<id>/tasks`).
   - **FindMentorTab & PaymentModal**: Discovers mentors (`GET /api/users/mentors`), initiates requests (`POST /api/connections/request`), and triggers Razorpay checkout (`POST /api/payments/create-order`).
   - **CareerBotTab**: Interactive chat powered by the RAG pipeline (`POST /api/chat/ask`).
3. **Mentor Dashboard** (`frontend/src/pages/MentorDashboard.jsx`):
   - Mentors view incoming student connection requests, accept/reject requests (`/api/connections`), and monitor session payouts.

---

## 🧠 4. AI Career Guidance: Why Lightweight Lookup & RAG Lookup?

Located in [`backend/services/ai_bot.py`](backend/services/ai_bot.py).

### Why "Lightweight" TF-IDF Lookup?
Most RAG implementations use heavyweight vector databases (Pinecone, Chroma, Milvus) and cloud embedding APIs (OpenAI Ada/3-small). For this project:
1. **Zero External Vector DB Overhead**: No server setup, no extra monthly hosting bills.
2. **Instant In-Memory Speed**: Pure Python algorithms compute word frequencies in RAM in less than **2 milliseconds**.
3. **Keyword-Precise Matching**: Accurately matches technical qualifications, entrance exams, and course names without embedding drift.

### The 5-Step RAG Pipeline
1. **Document Ingestion**: `career_docs.txt` is loaded and split into section and paragraph chunks.
2. **Lightweight TF-IDF Scoring**:
   ```python
   # Formula: TF (Term Frequency) * IDF (Inverse Document Frequency)
   tf = doc_freq.get(term, 0) / total_doc_tokens
   idf = math.log((N + 1) / (df + 1)) + 1
   score += tf * idf
   ```
3. **Relevance Filtering**: Returns top 3 most relevant chunks exceeding the score threshold.
4. **Context Augmentation**: Injects retrieved knowledge chunks directly into the Groq system prompt:
   ```python
   context_block = "\n\n---\n".join(relevant_chunks[:3])
   system_content += f"\n=== CAREER KNOWLEDGE BASE CONTEXT ===\n{context_block}\n=== END CONTEXT ==="
   ```
5. **Generation**: Groq's ultra-fast `llama-3.1-8b-instant` model generates a concise, accurate answer grounded in your verified data.

---

## 🔐 5. Environment Variables (.env) Explained

### Why your friend didn't get `.env` when cloning:
For security, git ignores `.env` files using `.gitignore` so passwords and private API keys are never leaked to public repositories.

To make running easy, the project includes [`backend/.env.example`](backend/.env.example). Your friend simply needs to copy this file to `.env`:

```ini
# Groq AI API Key (Free from https://console.groq.com)
GROK_API_KEY=gsk_your_groq_key_here

# JWT Secret Key
JWT_SECRET_KEY=collabsphere-jwt-secret-2024

# Database Connection (Leave commented out to automatically use local SQLite!)
# DATABASE_URL=postgresql://postgres:password@localhost:5432/collab_db

# Razorpay Test Keys (Optional, for payment testing)
RAZORPAY_KEY_ID=rzp_test_your_key
RAZORPAY_KEY_SECRET=your_secret
```

> **Zero Database Setup Needed**: If `DATABASE_URL` is commented out, Flask automatically creates and runs `sqlite:///collab.db` locally!

---

## 🚀 6. Step-by-Step Guide to Run the Project (For Collaborators)

### Step 1: Clone the Repository
```bash
git clone https://github.com/Haryavardhan/collab.git
cd collab
```

---

### Step 2: Start the Backend Server (Terminal 1)
```bash
# 1. Navigate to backend directory
cd backend

# 2. Create Python virtual environment
python -m venv venv

# 3. Activate the virtual environment:
# Windows (Command Prompt / PowerShell):
venv\Scripts\activate
# Mac / Linux:
# source venv/bin/activate

# 4. Install dependencies
pip install -r requirements.txt

# 5. Create .env file from template:
# Windows PowerShell:
Copy-Item .env.example .env
# Windows CMD:
# copy .env.example .env
# Mac/Linux:
# cp .env.example .env

# 6. Run the Flask server
python app.py
```
Backend will be live at: `http://127.0.0.1:5000`

---

### Step 3: Start the Frontend Server (Terminal 2)
Open a **new, separate terminal** in the project root:
```bash
# 1. Navigate to frontend directory
cd frontend

# 2. Install Node.js packages
npm install

# 3. Run the development server
npm run dev
```
Frontend will be live at: `http://localhost:5173`

---

## 📡 7. API Endpoints Summary

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/api/auth/register` | Register new student or mentor |
| `POST` | `/api/auth/login` | Authenticate and return JWT token |
| `GET` | `/api/rooms` | Fetch user's study rooms |
| `POST` | `/api/rooms` | Create a new study room |
| `GET` | `/api/rooms/<id>/messages` | Fetch chat messages in room |
| `POST` | `/api/rooms/<id>/messages` | Send chat message in room |
| `GET` | `/api/rooms/<id>/tasks` | Fetch Kanban tasks for room |
| `POST` | `/api/rooms/<id>/tasks` | Add or update Kanban task |
| `GET` | `/api/users/mentors` | Fetch available mentors |
| `POST` | `/api/connections/request` | Send mentorship connection request |
| `POST` | `/api/payments/create-order`| Create Razorpay order |
| `POST` | `/api/chat/ask` | Send question to RAG Career Bot |
