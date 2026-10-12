# BST-Internal-Hackathon--Team-Hackstorm
# BST Smart Exam Planner

![Exam Planner Thumbnail](./docs/thumbnail.png)

*A smart examination management platform for Bosscoder School of Technology (BST). Coordinators can allocate seats, schedule invigilators, understand conflicts in plain language, and replan live when a room or invigilator is suddenly unavailable. Students just enter their roll number to find their room and seat. Built with a simple, classy blue-and-white theme with light and dark modes.*

## Full App Flow in Short
Coordinator uploads students, rooms and staff (CSV/Excel) → backend validates the data → solver generates seating and invigilator roster → conflicts are explained in plain language → coordinator approves and publishes → students look up their room and seat by roll number → if an emergency happens, the replanner fixes only what is affected and publishes a new version.

## Demo Links

- **Live App:** `<add-deployed-url>`
- **Student Lookup:** `<add-deployed-url>/lookup`
- **Demo Video:** `<add-video-url>`

## Demo Login Credentials

| Role | Email | Password |
|------|-------|----------|
| Admin / Coordinator | `<add-email>` | `<add-password>` |
| Invigilator | `<add-email>` | `<add-password>` |

Students do not need to log in. They only enter their roll number.

> Use dummy data only. Never put real student records or real passwords in this README.

## Features Implemented

> Tick only what is actually working. Move the rest to the roadmap.

- [ ] Role-based login for admin, coordinator and invigilator
- [ ] Exam, subject, shift, room and seat-layout management
- [ ] CSV / Excel import with validation (missing fields, duplicate roll numbers)
- [ ] Automatic seat allocation within room capacity
- [ ] Anti-cheating rule: avoid same-subject students sitting adjacent (soft rule)
- [ ] Conflict-free invigilator scheduling with workload limits
- [ ] Explainable conflicts: what failed, why, and what to do next
- [ ] Live emergency replanning (room cancelled, invigilator absent)
- [ ] Old plan vs new plan comparison, approval and version history
- [ ] Student roll-number lookup with visual seat map
- [ ] PDF / Excel export of seating charts and duty rosters
- [ ] Light / dark mode with saved preference

## How I Approached the Problem
Exam planning is a constraint problem, not just a CRUD app. So I split it into two parts: a normal web app (data, roles, UI) and a **solver** that only decides *who sits where* and *who invigilates where*.

Rules are divided into two kinds:

- **Hard rules (must never break):** room capacity, one seat per student, one student per seat, an invigilator cannot be in two rooms at the same time, only published plans are visible to students.
- **Soft rules (try to satisfy):** keep same-subject students apart, balance invigilator workload, keep changes minimum during replanning.

If no valid plan exists, the system does not fail silently. It tells the coordinator exactly which hard rule is blocking, for example "Shift 2 has 120 students but usable seats are 96. Short by 24."

For emergencies, the replanner keeps everything that is not affected and only re-solves the broken part. This keeps disruption low and students are not moved for no reason.

## Trade-offs I Made

- Used a constraint solver instead of a greedy algorithm. It is more reliable for hard rules but needs one extra service to run.
- Replanning minimises changes rather than finding the "perfect" plan from scratch. This is more practical on exam day.
- Roll-number lookup is public but shows only published data and only that student's own seat. No other personal details are shown.
- Focused on the core planning flow first. Notifications (email / SMS) are left for later.

# App Architecture

```mermaid
flowchart TB
    F["Frontend (React + TypeScript)<br>Dashboard | Planner | Replan | Student Lookup"]
    API["API Layer (Node.js / Express)<br>Auth | Exams | Import | Plans | Lookup"]
    DB["Database (PostgreSQL)<br>Users | Exams | Rooms | Seats | Students | Staff | Plans"]
    SOLVER["Solver Service (Python + OR-Tools CP-SAT)<br>Seating | Invigilators | Replanning"]
    JOBS["Background Jobs<br>Import Processing | Solver Runs | Exports"]

    F --> API
    API --> DB
    API --> SOLVER
    API --> JOBS
    JOBS --> DB
    JOBS --> SOLVER
```

## Flow Summary
*Step-by-step breakdown of the planning workflow*

### `01` Configure
Admin adds exams, subjects, shifts, rooms with seat layouts (rows x columns), and staff availability. Student and staff data can be imported from CSV / Excel.

### `02` Validate
Backend checks duplicate roll numbers, missing fields, and whether total seats are enough for registered students.

### `03` Generate
Solver service receives students, rooms and staff, and returns a seating plan and an invigilator roster as a **draft**.

### `04` Resolve
If the plan is not possible, the conflict explainer returns the blocking rule, the affected students / rooms, and suggested fixes.

### `05` Approve
Coordinator reviews the draft, then marks it **validated** and **approved**. Every change is saved in the audit log.

### `06` Publish
Only an approved plan can be published. After publishing, students can see their room and seat.

### `07` Replan
On emergency (room cancelled, invigilator absent), the replanner locks unaffected assignments, re-solves the affected part, and shows a **diff** between the old and new plan. After approval, it publishes as a new version.

## Plan Lifecycle

```mermaid
stateDiagram-v2
    [*] --> Draft
    Draft --> Conflict: Hard rule fails
    Conflict --> Draft: Fix data / apply suggestion
    Draft --> Validated: All hard rules pass
    Validated --> Approved: Coordinator approves
    Approved --> Published: Publish
    Published --> Replanning: Emergency
    Replanning --> Validated: New version generated
```

## Component Interaction Flow

```mermaid
sequenceDiagram
    participant C as Coordinator
    participant UI as Frontend
    participant API as Backend API
    participant S as Solver
    participant DB as PostgreSQL

    C->>UI: Click Generate Plan
    UI->>API: POST /api/plans/generate
    API->>DB: Load students, rooms, staff
    API->>S: Solve (hard + soft rules)
    S-->>API: Plan or conflict report
    API->>DB: Save draft version
    API-->>UI: Plan / explanation
    C->>UI: Approve and Publish
    UI->>API: POST /api/plans/:id/publish
    API->>DB: Mark published, write audit log
```

## Emergency Replanning Flow

```mermaid
flowchart LR
    A["Emergency reported<br>(room cancelled / invigilator absent)"] --> B["Find affected<br>students and duties"]
    B --> C["Lock everything<br>not affected"]
    C --> D["Re-solve affected part"]
    D --> E{"Valid plan?"}
    E -- Yes --> F["Show old vs new diff"]
    E -- No --> G["Explain shortage<br>+ suggest options"]
    F --> H["Coordinator approves"]
    H --> I["Publish new version"]
```

## Student Lookup Flow

```mermaid
sequenceDiagram
    participant St as Student
    participant UI as Frontend
    participant API as Backend API
    participant DB as PostgreSQL

    St->>UI: Enter roll number
    UI->>API: GET /api/lookup/:rollNo
    API->>DB: Find seat in published plan
    DB-->>API: Exam, room, seat, time
    API-->>UI: Only that student's data
    UI->>St: Room, floor, seat + highlighted seat map
```

## Tech Stack
*Proposed stack. Update the versions and remove anything not actually used.*

### Frontend
<table>
<tr>
<td><strong>React</strong><br><sub>UI</sub></td>
<td><strong>TypeScript</strong><br><sub>Type safety</sub></td>
<td><strong>Tailwind CSS</strong><br><sub>Blue-white theme + dark mode</sub></td>
<td><strong>Lucide</strong><br><sub>Icons</sub></td>
</tr>
</table>

### Backend
<table>
<tr>
<td><strong>Node.js / Express</strong><br><sub>API</sub></td>
<td><strong>PostgreSQL</strong><br><sub>Database</sub></td>
<td><strong>Prisma</strong><br><sub>ORM</sub></td>
<td><strong>JWT</strong><br><sub>Auth</sub></td>
</tr>
</table>

### Optimization and Data
<table>
<tr>
<td><strong>Google OR-Tools CP-SAT</strong><br><sub>Constraint solver</sub></td>
<td><strong>CSV / Excel parser</strong><br><sub>Import</sub></td>
<td><strong>PDF / XLSX export</strong><br><sub>Reports</sub></td>
</tr>
</table>

## Backend Components

- **Routes**
    - `auth` for login and role checks
    - `exams` for exams, subjects, shifts
    - `rooms` for rooms and seat layouts
    - `import` for CSV / Excel upload and validation
    - `plans` for generate, validate, approve, publish, replan, versions
    - `lookup` for the public roll-number lookup (published data only)
- **Services**
    - `allocationService` prepares solver input and saves seating output
    - `invigilatorService` handles roster and substitute suggestions
    - `conflictService` converts solver failures into readable explanations
    - `replanService` locks unaffected assignments and computes the diff
    - `exportService` generates PDF and Excel reports
- **Middleware**
    - JWT verification and role-based access (admin / coordinator / invigilator)
    - Input validation and rate limiting on the public lookup route
- **Observability**
    - Audit log for every plan change (who, what, when)
    - Plan version history with rollback to an older version

## Project Structure
*Suggested layout. Adjust to match your actual repo.*

```
bst-exam-planner/
├── README.md
├── docs/
│   └── thumbnail.png
├── backend/
│   ├── package.json
│   ├── prisma/
│   │   └── schema.prisma
│   └── src/
│       ├── index.ts
│       ├── routes/
│       │   ├── auth.ts
│       │   ├── exams.ts
│       │   ├── rooms.ts
│       │   ├── import.ts
│       │   ├── plans.ts
│       │   └── lookup.ts
│       ├── services/
│       │   ├── allocationService.ts
│       │   ├── invigilatorService.ts
│       │   ├── conflictService.ts
│       │   ├── replanService.ts
│       │   └── exportService.ts
│       ├── middleware/
│       │   ├── auth.ts
│       │   └── validate.ts
│       └── utils/
├── solver/
│   ├── requirements.txt
│   └── app/
│       ├── main.py
│       ├── seating.py
│       ├── invigilators.py
│       └── replan.py
└── frontend/
    ├── package.json
    └── src/
        ├── pages/
        │   ├── Dashboard.tsx
        │   ├── Planner.tsx
        │   ├── Replan.tsx
        │   └── Lookup.tsx
        ├── components/
        │   ├── SeatMap.tsx
        │   ├── ConflictPanel.tsx
        │   ├── PlanDiff.tsx
        │   └── ThemeToggle.tsx
        ├── hooks/
        ├── services/
        └── types/
```

## Quick Start

### Prerequisites

<table>
<tr>
<td><strong>Node.js</strong><br><sub>v18 or higher</sub></td>
<td><strong>PostgreSQL</strong><br><sub>Database server</sub></td>
<td><strong>Python</strong><br><sub>3.10+ for solver</sub></td>
</tr>
</table>

### `01` Backend Setup

```bash
cd backend
npm install
cp .env.example .env
npx prisma migrate dev
npm run dev
```

#### Backend (.env)
```bash
DATABASE_URL="postgresql://username:password@localhost:5432/exam_planner"
JWT_SECRET="change-this-secret"
PORT=3001
FRONTEND_URL="http://localhost:5173"
SOLVER_URL="http://localhost:8000"
```

### `02` Solver Setup

```bash
cd solver
pip install -r requirements.txt
uvicorn app.main:app --port 8000
```

### `03` Frontend Setup

```bash
cd frontend
npm install
cp .env.example .env.local
npm run dev
```

#### Frontend (.env.local)
```bash
VITE_API_URL=http://localhost:3001
```

### Application URLs

**Backend API:** `http://localhost:3001`
**Solver:** `http://localhost:8000`
**Frontend:** `http://localhost:5173`

> Never commit `.env` files, passwords, API keys or real student data to GitHub.

## API Reference

### GET Endpoints

| Endpoint | Description |
|----------|-------------|
| `/api/exams` | List exams |
| `/api/rooms` | List rooms and seat layouts |
| `/api/plans/:id` | Get a plan with seating and roster |
| `/api/plans/:id/versions` | Version history of a plan |
| `/api/plans/:id/diff?with=:versionId` | Compare two plan versions |
| `/api/lookup/:rollNo` | Student room and seat (published plans only) |

### POST Endpoints

| Endpoint | Description |
|----------|-------------|
| `/api/auth/login` | Login for staff roles |
| `/api/import/students` | Import students from CSV / Excel |
| `/api/import/staff` | Import staff from CSV / Excel |
| `/api/plans/generate` | Generate seating and invigilator plan |
| `/api/plans/:id/approve` | Approve a validated plan |
| `/api/plans/:id/publish` | Publish an approved plan |
| `/api/plans/:id/replan` | Replan after room cancellation or absence |
| `/api/plans/:id/export` | Export PDF / Excel |

> Protected endpoints need `Authorization: Bearer <token>`. The lookup endpoint is public but returns published data for one roll number only.

## Database Schema

<table>
<tr>
<td><strong>users</strong><br><sub>Staff accounts and roles</sub></td>
<td><strong>exams</strong><br><sub>Exam, subject, date, shift</sub></td>
<td><strong>rooms</strong><br><sub>Building, floor, layout</sub></td>
<td><strong>seats</strong><br><sub>Row, column, status</sub></td>
</tr>
<tr>
<td><strong>students</strong><br><sub>Roll number, subjects</sub></td>
<td><strong>staff</strong><br><sub>Availability, workload limit</sub></td>
<td><strong>plans</strong><br><sub>Versions and status</sub></td>
<td><strong>assignments</strong><br><sub>Seat and duty mapping</sub></td>
</tr>
<tr>
<td><strong>audit_logs</strong><br><sub>Who changed what</sub></td>
</tr>
</table>

## Rules Used by the Planner

| Type | Rule |
|------|------|
| Hard | Room capacity is never exceeded |
| Hard | One student gets exactly one seat, and one seat holds one student |
| Hard | An invigilator cannot have two duties at the same time |
| Hard | Only approved plans can be published |
| Soft | Same-subject students should not sit adjacent |
| Soft | Invigilator workload should be balanced |
| Soft | Replanning should change as few assignments as possible |

## Example Scenario

100 students are registered and one room (30 seats) is cancelled.

1. Affected students and the invigilators of that room are identified.
2. Remaining usable capacity is recalculated.
3. The solver finds a valid arrangement, keeping unaffected students where they are.
4. If seats are not enough, the system shows the exact shortage and suggestions (use another room, add a shift, etc.).
5. Coordinator compares old vs new plan, approves, and publishes a new version.

The system never exceeds room capacity and never publishes an invalid plan.

## Deployment

| Service | Platform |
|---------|----------|
| Frontend | `<Vercel / Netlify>` |
| Backend | `<Railway / Render>` |
| Database | `<Managed PostgreSQL>` |
| Solver | `<Railway / Render / Docker>` |

## Project Limitations and Assumptions

- Built for BST's exam setup. Other institutions would need configuration changes.
- Rooms are assumed to be rectangular grids of rows and columns.
- Staff availability must be entered or imported before generating a roster.
- Email / SMS notifications are not included yet.
- Very large exams may need longer solver time limits.
- Security and reliability depend on the final deployment and testing.

## Roadmap

- [ ] Exam, room and seat-layout management
- [ ] Student and staff data import
- [ ] Seat allocation
- [ ] Invigilator scheduling
- [ ] Conflict explanations
- [ ] Emergency replanning with plan diff
- [ ] Roll-number lookup with seat map
- [ ] PDF / Excel exports
- [ ] Approval workflow and versioning
- [ ] Light / dark mode
- [ ] Notifications (email / SMS)
- [ ] End-to-end testing and deployment

## Contributing

1. Fork the repository
2. Create a feature branch
3. Make and test your changes
4. Open a pull request with a clear description

Please do not include real student records in issues, commits, screenshots or test data.

## License

Add a license (for example MIT) before sharing publicly. Until then, others do not have permission to reuse the code.

**Built for Bosscoder School of Technology.**
*Made by Manyam*
