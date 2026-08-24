# Healthcare Appointment & Follow-up Manager

A full-stack healthcare appointment platform with separate **Patient, Doctor, and Admin portals**.

## Features

- Role-based authentication: `patient`, `doctor`, `admin`
- Doctor profile management:
  - specialization
  - working hours
  - slot duration
  - leave days
- Patient doctor search and appointment booking
- Transaction-safe double-booking prevention
- Temporary slot-hold mechanism
- Doctor leave conflict handling + patient notifications
- AI pre-visit symptom summary:
  - urgency: Low / Medium / High
  - chief complaint
  - three suggested questions
- AI post-visit patient-friendly summary
- Prescription storage and medication reminders
- Email notifications with retry support
- Google Calendar OAuth 2.0 integration
- Appointment reschedule/cancellation calendar synchronization
- Background jobs using `node-cron`
- PostgreSQL database
- React frontend + Express backend

> **Important:** This project is a software engineering demonstration. AI output is not a diagnosis and must not replace professional clinical judgment.

---

## Project structure

```text
healthcare-appointment-followup-manager/
├── backend/
│   ├── src/
│   │   ├── config/
│   │   ├── middleware/
│   │   ├── routes/
│   │   ├── services/
│   │   ├── jobs/
│   │   ├── utils/
│   │   ├── app.js
│   │   └── server.js
│   ├── db/schema.sql
│   ├── .env.example
│   └── package.json
├── frontend/
│   ├── src/
│   │   ├── components/
│   │   ├── pages/
│   │   ├── services/
│   │   ├── App.jsx
│   │   └── main.jsx
│   ├── .env.example
│   └── package.json
├── docs/
│   ├── API.md
│   ├── SYSTEM_DESIGN.md
│   └── LLM_PROMPTS.md
├── render.yaml
└── README.md
```

---

# 1. Requirements

- Node.js 20+
- PostgreSQL 14+
- Google Cloud project for Calendar OAuth
- An LLM API key
- Email SMTP credentials (Gmail SMTP, SendGrid SMTP, Mailgun SMTP, etc.)

---

# 2. Backend setup

```bash
cd backend
npm install
cp .env.example .env
```

Create a PostgreSQL database, then run:

```bash
psql "$DATABASE_URL" -f db/schema.sql
```

Start development:

```bash
npm run dev
```

Backend default:

```text
http://localhost:5000
```

Health check:

```text
GET http://localhost:5000/api/health
```

---

# 3. Frontend setup

```bash
cd frontend
npm install
cp .env.example .env
npm run dev
```

Frontend:

```text
http://localhost:5173
```

Set:

```env
VITE_API_URL=http://localhost:5000/api
```

---

# 4. Environment variables

See:

- `backend/.env.example`
- `frontend/.env.example`

Required backend variables:

```env
DATABASE_URL=
JWT_SECRET=
LLM_API_KEY=
LLM_MODEL=
SMTP_HOST=
SMTP_PORT=
SMTP_USER=
SMTP_PASS=
MAIL_FROM=
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
GOOGLE_REDIRECT_URI=
FRONTEND_URL=
```

---

# 5. Google Calendar OAuth setup

1. Open Google Cloud Console.
2. Create/select a project.
3. Enable **Google Calendar API**.
4. Configure OAuth consent screen.
5. Create an OAuth 2.0 Web Application client.
6. Add the redirect URI:

```text
http://localhost:5000/api/calendar/oauth/callback
```

For production, use:

```text
https://YOUR-BACKEND-DOMAIN/api/calendar/oauth/callback
```

7. Put the client ID and secret in `.env`.
8. Log in as a patient or doctor.
9. Open:

```text
GET /api/calendar/oauth/start
```

The application stores Google access/refresh tokens securely in the database.

---

# 6. Email setup

The project uses Nodemailer and SMTP.

Example Gmail SMTP configuration:

```env
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your-email@gmail.com
SMTP_PASS=your-app-password
MAIL_FROM=Clinic <your-email@gmail.com>
```

For SendGrid or Mailgun, use their SMTP host/credentials instead.

---

# 7. Demo accounts

After starting the server, create accounts through:

```text
POST /api/auth/register
```

The first admin can be created by registering with `role=admin` in development, or by directly updating the database.

For production, disable public admin registration and create admins through a secure seed/migration process.

---

# 8. Main API groups

```text
POST   /api/auth/register
POST   /api/auth/login

GET    /api/doctors
POST   /api/doctors
PATCH  /api/doctors/:id

GET    /api/doctors/:id/slots
POST   /api/doctors/:id/leave
DELETE /api/doctors/:id/leave/:leaveId

POST   /api/appointments/hold
POST   /api/appointments
GET    /api/appointments
PATCH  /api/appointments/:id/reschedule
POST   /api/appointments/:id/cancel

POST   /api/appointments/:id/previsit-summary
POST   /api/appointments/:id/visit

GET    /api/calendar/oauth/start
GET    /api/calendar/oauth/callback

GET    /api/health
```

See `docs/API.md` for request/response examples.

---

# 9. Double-booking prevention

The appointment table contains:

```sql
UNIQUE (doctor_id, appointment_start)
```

Booking happens inside a PostgreSQL transaction.

The server:

1. Validates the slot.
2. Locks the doctor row.
3. Checks doctor leave.
4. Checks active slot holds.
5. Attempts the unique appointment insert.
6. Commits only after all checks succeed.

If two users attempt the same slot at the same time, PostgreSQL allows only one appointment to win.

---

# 10. Slot hold

The patient can create a temporary hold before confirming.

Default hold duration:

```env
SLOT_HOLD_MINUTES=5
```

Expired holds are automatically cleaned by a background job.

The hold table has a unique constraint on:

```text
doctor_id + appointment_start
```

This prevents two simultaneous active holds for the same slot.

---

# 11. Doctor leave handling

When an admin creates leave:

1. The leave is stored.
2. Existing confirmed appointments in the leave window are selected.
3. Those appointments are marked `CANCELLED`.
4. A notification record is created for each affected patient/doctor.
5. Email delivery is attempted asynchronously.
6. Calendar events are deleted where possible.

If notification delivery fails, the notification remains in the database with a retry count.

---

# 12. Notification reliability

Notifications use an outbox-style database table.

States:

```text
PENDING
SENT
FAILED
```

A background worker retries failed email jobs with exponential backoff.

This prevents a temporary SMTP outage from breaking appointment booking.

---

# 13. LLM failure handling

LLM calls are isolated in `services/llmService.js`.

If the model fails:

- appointment booking still succeeds
- the failure is logged
- a safe fallback summary is generated
- `ai_status=FAILED` is stored
- the user can retry AI generation later

The application never treats an LLM response as a prerequisite for saving a medical appointment.

---

# 14. Production deployment

A sample `render.yaml` is included.

Typical deployment:

### Database

Create a PostgreSQL database on Render.

### Backend

Create a web service:

```text
Root Directory: backend
Build Command: npm install
Start Command: npm start
```

### Frontend

Create a static site:

```text
Root Directory: frontend
Build Command: npm install && npm run build
Publish Directory: dist
```

Set:

```env
VITE_API_URL=https://YOUR-BACKEND-DOMAIN/api
```

Update backend:

```env
FRONTEND_URL=https://YOUR-FRONTEND-DOMAIN
GOOGLE_REDIRECT_URI=https://YOUR-BACKEND-DOMAIN/api/calendar/oauth/callback
```

A real hosted URL cannot be supplied from this source package alone because deployment requires access to your hosting and Google Cloud accounts.

---

# 15. Security notes

For a production healthcare deployment, additionally implement:

- HTTPS everywhere
- encrypted secrets
- secure cookie/session strategy
- audit logging
- database encryption/backups
- strict CORS
- rate limiting
- CSRF protection where applicable
- input validation
- least-privilege database users
- healthcare/privacy compliance appropriate to the deployment jurisdiction
- proper clinical governance for AI-generated content

Do not put real patient data into development/demo environments.

