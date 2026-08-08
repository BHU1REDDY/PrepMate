# PrepMate

An AI-powered interview prep tool. Tell it a role, experience level, and topics to focus on, and it generates a set of interview questions and answers with Google's Gemini API — organized into sessions you can revisit, pin, and expand on later.

## Features

- **Email/password auth** with JWT (7‑day tokens), plus profile photo upload
- **AI-generated Q&A sessions** — give a role, years of experience, and topics; get back a structured set of questions with beginner-friendly answers (code blocks included where relevant)
- **"Learn More" on any question** — asks Gemini to explain the underlying concept in more depth
- **Session dashboard** — all your sessions, each with its questions, editable notes, and pinning for the ones you want to revisit
- **Structured AI output** — Gemini responses are constrained with `responseMimeType`/`responseSchema` rather than relying on prompt instructions alone, so generation reliably returns valid JSON

## Tech stack

| Layer | Tech |
|---|---|
| Frontend | React 19, Vite, Tailwind CSS, React Router, Axios, Framer Motion, react-markdown |
| Backend | Node.js, Express 5, Mongoose |
| Database | MongoDB |
| AI | Google Gemini (`@google/genai`, model: `gemini-2.0-flash-lite`) |
| Auth | JWT + bcrypt |
| Deployment | Vercel (backend runs as a single serverless function) |

## Architecture

```
React client (Vite SPA)
  │  Authorization: Bearer <jwt>
  ▼
Express server (server.js)
  ├─ /api/auth       register · login · profile · upload-image
  ├─ protect (JWT middleware)  — guards everything below
  ├─ /api/sessions   create · my-sessions · :id · delete
  ├─ /api/questions  add · :id/pin · :id/note
  └─ /api/ai         generate-questions · generate-explanation ──▶ Google Gemini API
                                                                      │
  MongoDB (Users · Sessions · Questions) ◀────────────────────────────┘
```

Register/login write straight to MongoDB and return a JWT. Every other route sits behind a `protect` middleware that verifies that JWT before continuing. Generating questions is a separate step from saving them: the client previews Gemini's output first, and only a follow-up `POST /api/sessions/create` persists it, so a rejected generation never touches the database.

## Data model

**User** — `name`, `email` (unique), `password` (bcrypt hash), `profileImageUrl`

**Session** — `user` (ref → User), `role`, `experience`, `topicsToFocus`, `description`, `questions` (ref → Question[])

**Question** — `session` (ref → Session), `question`, `answer`, `note`, `isPinned`

## Getting started

### Prerequisites

- Node.js 18+
- A MongoDB connection string (e.g. MongoDB Atlas)
- A Google Gemini API key

### Backend

```bash
cd backend
npm install
```

Create `backend/.env`:

```
MONGO_URI=your_mongodb_connection_string
JWT_SECRET=your_jwt_secret
GEMINI_API_KEY=your_gemini_api_key
PORT=8000
```

```bash
npm run dev   # starts on http://localhost:8000
```

### Frontend

```bash
cd frontend/PrepMate
npm install
npm run dev   # starts on http://localhost:5173
```

Point the frontend's API base URL at your backend if it's not already configured for `localhost:8000`.

## API reference

| Method | Route | Access | Description |
|---|---|---|---|
| POST | `/api/auth/register` | Public | Create a user, returns a JWT |
| POST | `/api/auth/login` | Public | Authenticate, returns a JWT |
| GET | `/api/auth/profile` | Private | Get the logged-in user |
| POST | `/api/auth/upload-image` | Public | Upload a profile photo |
| POST | `/api/sessions/create` | Private | Create a session with its questions |
| GET | `/api/sessions/my-sessions` | Private | List the logged-in user's sessions |
| GET | `/api/sessions/:id` | Private | Get a session with populated questions |
| DELETE | `/api/sessions/:id` | Private | Delete a session and its questions |
| POST | `/api/questions/add` | Private | Add more questions to an existing session |
| POST | `/api/questions/:id/pin` | Private | Toggle pin on a question |
| POST | `/api/questions/:id/note` | Private | Update a question's note |
| POST | `/api/ai/generate-questions` | Private | Generate a Q&A set via Gemini |
| POST | `/api/ai/generate-explanation` | Private | Generate a concept explanation for a question |

## Notes

- `.env` files are git-ignored — never commit real credentials.
- If secrets were ever committed to this repo, rotate them (`MONGO_URI`, `JWT_SECRET`, `GEMINI_API_KEY`) rather than assuming removal from history is enough.
