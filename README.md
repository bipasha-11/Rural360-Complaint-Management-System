# Rural360-Complaint-Management-System

A full-stack, production-oriented grievance redressal portal for Gram Panchayat operations. Citizens can file, track, and provide feedback on complaints. Clerks, Staff, and Admins manage workflows, assign tasks, and monitor resolution metrics.

---

## Quick Start Summary

For a local development setup, the backend and frontend can be started independently:

```bash
# Backend
cd backend
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
python run.py
```

```bash
# Frontend (in a second terminal)
cd frontend
npm install
npm run dev
```

The web app is typically available at http://localhost:5173, while the Flask API runs at http://localhost:5000.

> If you are setting up the database for the first time, create the required Oracle schema before starting the backend.

---

## Technology Stack

| Layer        | Technology                                     |
|--------------|------------------------------------------------|
| Frontend     | React 18 (Vite), Vanilla CSS, Axios            |
| Backend      | Python Flask, Flask-JWT-Extended, Flask-Bcrypt |
| Validation   | Marshmallow                                    |
| Database     | Oracle Database (cx_Oracle)                    |
| Auth         | JWT tokens, OTP via SMS gateway                |

---

## Folder Structure

```
PANCHAYAT SYSTEM DOSC/
├── backend/
│   ├── app/
│   │   ├── __init__.py          # Flask app factory
│   │   ├── config.py            # Environment config
│   │   ├── extensions.py        # JWT, Bcrypt singletons
│   │   ├── models/
│   │   │   └── schemas.py       # Marshmallow validation schemas
│   │   ├── routes/
│   │   │   ├── auth.py          # OTP, Register, Login, Forgot-password
│   │   │   ├── complaints.py    # CRUD + status transitions
│   │   │   ├── assignments.py   # Assign staff to complaints
│   │   │   ├── feedback.py      # Citizen feedback
│   │   │   └── admin.py         # Analytics endpoints
│   │   └── utils/
│   │       ├── db.py            # cx_Oracle parameterized queries
│   │       ├── otp.py           # OTP generation & SMS dispatch
│   │       └── decorators.py    # Role-based JWT decorators
│   ├── .env                     # Environment variables (DO NOT COMMIT)
│   ├── requirements.txt
│   └── run.py
├── database/
│   └── schema.sql               # Oracle SQL Plus DDL script
├── frontend/
│   ├── src/
│   │   ├── api/axios.js         # Axios instance with JWT interceptor
│   │   ├── components/          # Header, Navbar, ProtectedRoute
│   │   ├── context/             # AuthContext
│   │   ├── pages/               # All page components
│   │   └── index.css            # Global government-style stylesheet
│   ├── index.html
│   └── vite.config.js           # Vite + proxy to Flask
└── README.md
```

---

## Database Setup (Oracle SQL Plus)

1. Open SQL Plus and connect as a privileged user:
   ```sql
   sqlplus your_user/your_password@localhost:1521/ORCL
   ```

2. Run the schema script:
   ```sql
   @path\to\database\schema.sql
   ```

   This creates:
   - Tables: `USERS`, `OTP_VERIFICATION`, `COMPLAINTS`, `ASSIGNMENTS`, `STATUS_HISTORY`, `FEEDBACK`
   - Sequences for all primary keys
   - Indexes on frequently queried columns
   - Trigger for auto-updating `COMPLAINTS.updated_at`
   - View `COMPLAINT_SUMMARY` for admin queries
   - Sample Admin, Staff, and Clerk seed data

3. Update passwords for seed users:
   ```sql
   UPDATE USERS SET password_hash = 'bcrypt_hash_here' WHERE phone_number = '9000000001';
   COMMIT;
   ```
   > Use Python `bcrypt.generate_password_hash('YourPassword').decode()` to generate the hash.

---

## Backend Setup

### Prerequisites
- Python 3.10+
- Oracle Instant Client installed and added to PATH
- pip

### Steps

```bash
cd backend

# Create virtual environment
python -m venv venv
venv\Scripts\activate        # Windows
# source venv/bin/activate   # Linux/macOS

# Install dependencies
pip install -r requirements.txt

# Configure environment
# Edit .env — fill in DB_DSN, DB_USER, DB_PASSWORD, JWT_SECRET_KEY
# Example:
#   DB_DSN=localhost:1521/ORCL
#   DB_USER=panchayat_user
#   DB_PASSWORD=SecurePass@123
#   JWT_SECRET_KEY=change-this-random-256-bit-key

# Start Flask
python run.py
```

Flask runs on `http://localhost:5000`

> **OTP in Development:** If `SMS_API_KEY` is not set in `.env`, OTPs are printed to the Flask console. Use them directly during development.

---

## Frontend Setup

### Prerequisites
- Node.js 18+

### Steps

```bash
cd frontend
npm install
npm run dev
```

React runs on `http://localhost:5173`  
All `/api/*` requests are proxied to Flask via `vite.config.js`.

---

## API Endpoints Reference

### Auth (`/api/auth`)
| Method | Endpoint             | Description                     | Auth |
|--------|----------------------|---------------------------------|------|
| POST   | `/send-otp`          | Send OTP to phone number        | No   |
| POST   | `/register`          | Register new citizen (OTP req.) | No   |
| POST   | `/login`             | Login, returns JWT              | No   |
| POST   | `/forgot-password`   | Reset password via OTP          | No   |

### Complaints (`/api/complaints`)
| Method | Endpoint              | Description                | Auth      |
|--------|-----------------------|----------------------------|-----------|
| POST   | `/`                   | Submit new complaint       | Citizen   |
| GET    | `/`                   | List complaints (filtered) | Any       |
| GET    | `/<id>`               | Get detail + history       | Any       |
| PATCH  | `/<id>/status`        | Update status              | Non-Citizen |

### Assignments (`/api/assignments`)
| Method | Endpoint          | Description           | Auth          |
|--------|-------------------|-----------------------|---------------|
| POST   | `/<complaint_id>` | Assign staff          | Admin/Clerk   |
| GET    | `/`               | List assignments      | Staff/Admin   |

### Feedback (`/api/feedback`)
| Method | Endpoint          | Description             | Auth    |
|--------|-------------------|-------------------------|---------|
| POST   | `/<complaint_id>` | Submit feedback (1–5 ★) | Citizen |
| GET    | `/<complaint_id>` | Get feedback            | Any     |

### Admin (`/api/admin`)
| Method | Endpoint               | Description                | Auth        |
|--------|------------------------|----------------------------|-------------|
| GET    | `/stats`               | Dashboard KPI counts       | Admin/Clerk |
| GET    | `/category-breakdown`  | Complaints by category     | Admin/Clerk |
| GET    | `/status-breakdown`    | Complaints by status       | Admin/Clerk |
| GET    | `/avg-resolution-time` | Avg days to close          | Admin/Clerk |
| GET    | `/staff-performance`   | Per-staff completion rate  | Admin/Clerk |
| GET    | `/staff-list`          | Active staff members       | Admin/Clerk |

---

## Complaint Workflow

```
Submitted → Under Review → Pending Approval → Approved → Assigned → In Progress → Completed → Closed
```

| Role   | Allowed Transitions                                    |
|--------|--------------------------------------------------------|
| Clerk  | Submitted → Under Review, Under Review → Pending Approval |
| Admin  | Pending Approval → Approved, Approved → Assigned, Completed → Closed |
| Staff  | Assigned → In Progress, In Progress → Completed        |
| Citizen| No status changes (can only submit & view)             |

---

## User Roles

| Role    | Access                                               |
|---------|------------------------------------------------------|
| Citizen | Submit, track, and give feedback on own complaints   |
| Clerk   | Review complaints, advance through early stages      |
| Staff   | View assigned tasks, update progress                 |
| Admin   | Full access: approve, assign, close, view analytics  |

---

## Security Notes

- All database queries use parameterized `cx_Oracle` binds (no raw string interpolation)
- Passwords hashed with `bcrypt` (cost factor 12)
- JWT tokens expire in 1 hour; stored in `localStorage`
- OTP rate-limited (1 request/minute per phone number)
- CORS restricted to `http://localhost:5173` in development
- Role enforcement at both frontend (route guards) and backend (decorator middleware)

---

## Known Limitations / Academic Context

- SMS dispatch is a stub — replace `SMS_API_URL` and `SMS_API_KEY` with a real provider (e.g., Textlocal, Fast2SMS)
- File upload endpoint is not implemented (PDF/JPEG requirement noted in spec but excluded per scope)
- Oracle Instant Client must be installed separately on the host machine
- The `ASSIGNMENTS.deadline` check constraint uses `SYSDATE` at DDL time and is advisory — enforce in application layer too
