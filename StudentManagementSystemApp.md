# Campus SMS — Student Management System

A full-stack Student Management System with two modules (**User** and **Admin**) covering
student data CRUD, attendance, academic records, administrative tasks and role management.
Built with TanStack Start (React 19 + TypeScript), Tailwind CSS v4 and a Postgres backend
with authentication and row-level security.

---

## 1. Features

### Common (User module — read access)
- Email/password and Google sign-in, session persistence, protected routes
- Dashboard: total students, active/inactive counts, attendance rate, department distribution, recent tasks
- Student directory with search + department/year filters
- Attendance register view by date
- Academic records view (marks, subject, term, certificates)
- Task board view (todo / in progress / done)

### Admin module (role-gated)
- Create, update and delete students
- Mark and edit daily attendance (present / absent / late + remarks)
- Add academic records (marks, max marks, subject, term, notes)
- Create and move tasks across statuses, set priority and due dates
- **Administration page**: assign/revoke the `admin` role for any user, plus a
  "Claim admin access" bootstrap for the first account

All write permissions are enforced in the database (RLS policies), not only in the UI.

---

## 2. Tech stack

| Layer | Technology |
| --- | --- |
| Framework | TanStack Start v1 (React 19, SSR, server functions) |
| Routing | TanStack Router (file-based, `src/routes`) |
| Build tool | Vite 7 |
| Language | TypeScript |
| Styling | Tailwind CSS v4 + shadcn/ui |
| Data layer | TanStack Query (React Query) |
| Backend | Postgres + Auth + RLS (Supabase via Lovable Cloud) |
| Icons | lucide-react |
| Toasts | sonner |

---

## 3. Project structure

```text
src/
├── components/
│   ├── AppShell.tsx            # Sidebar + topbar layout, nav, role badge, sign out
│   └── ui/                     # shadcn/ui primitives (button, card, dialog, table, ...)
├── hooks/
│   ├── useAuth.tsx             # Auth context: session, user, roles, isAdmin, signOut
│   └── use-mobile.tsx
├── integrations/supabase/
│   ├── client.ts               # Browser client (auto-generated)
│   └── types.ts                # Generated database types
├── lib/
│   ├── sms.ts                  # Domain types + fetchers (students, attendance, records, tasks)
│   └── utils.ts
├── routes/
│   ├── __root.tsx              # HTML shell, providers, SEO head, error/404 boundaries
│   ├── index.tsx               # Public landing page
│   ├── auth.tsx                # Sign in / sign up
│   └── _authenticated/
│       ├── route.tsx           # Auth gate (redirects to /auth when signed out)
│       ├── dashboard.tsx       # Overview + statistics
│       ├── students.tsx        # Student CRUD
│       ├── attendance.tsx      # Daily attendance register
│       ├── records.tsx         # Academic records
│       ├── tasks.tsx           # Task board
│       └── administration.tsx  # Role management
└── styles.css                  # Design tokens (colors, fonts, radii) + Tailwind theme
```

### Route map

| Path | Access | Purpose |
| --- | --- | --- |
| `/` | public | Landing page |
| `/auth` | public | Sign in / sign up |
| `/dashboard` | authenticated | Statistics overview |
| `/students` | authenticated (write: admin) | Student directory + CRUD |
| `/attendance` | authenticated (write: admin) | Daily register |
| `/records` | authenticated (write: admin) | Academic records |
| `/tasks` | authenticated (write: admin) | Task board |
| `/administration` | authenticated (actions: admin) | Roles & admin claim |

---

## 4. Database schema

```text
profiles(id -> auth.users, full_name, email, created_at)
user_roles(id, user_id -> auth.users, role: app_role)   unique(user_id, role)
students(id, roll_no unique, full_name, email, phone, dob, gender, address,
         course, department, year, section, status, created_by, created_at, updated_at)
attendance(id, student_id -> students, date, status: attendance_status,
           remarks, marked_by, created_at)
student_records(id, student_id -> students, record_type, title, subject, term,
                marks, max_marks, notes, created_by, created_at)
tasks(id, title, description, status: task_status, priority: task_priority,
      due_date, student_id -> students, created_by, created_at, updated_at)
```

Enums:
- `app_role`: `admin` | `user`
- `attendance_status`: `present` | `absent` | `late`
- `task_status`: `todo` | `in_progress` | `done`
- `task_priority`: `low` | `medium` | `high`

### Security model
- Roles live in a **separate `user_roles` table** (never on `profiles`) to prevent privilege escalation.
- `has_role(_user_id, _role)` is a `SECURITY DEFINER STABLE` SQL function used inside policies to
  avoid recursive RLS evaluation.
- RLS is enabled on every table: authenticated users may `SELECT`; `INSERT/UPDATE/DELETE`
  require `has_role(auth.uid(), 'admin')`.
- A sign-up trigger inserts the `profiles` row and the default `user` role.
- `claim_first_admin()` promotes the caller to `admin` only while no admin exists yet.

---

## 5. Running locally

Requirements: Node.js 20+ and npm.

```sh
git clone <your-repo-url>
cd <repo-name>
npm install
npm run dev            # http://localhost:8080
```

Environment variables (`.env`, provided by the backend integration):

```env
VITE_SUPABASE_URL=<project url>
VITE_SUPABASE_PUBLISHABLE_KEY=<publishable key>
VITE_SUPABASE_PROJECT_ID=<project id>
```

Only publishable keys are used in the browser; privileged keys never reach the client.

| Command | Description |
| --- | --- |
| `npm run dev` | Dev server with HMR |
| `npm run build` | Production build |
| `npm run lint` | ESLint |

---

## 6. First-run setup

1. Open `/auth` and create an account (email + password, or Google).
2. Go to **Administration** and press **Claim admin access** — the first user becomes admin.
3. Add students in **Students**, then mark attendance, add records and create tasks.
4. Later sign-ups get the `user` role (read-only) until an admin promotes them.

---

## 7. Implementation notes

- **Data access**: reads go through typed helpers in `src/lib/sms.ts`, cached by TanStack Query
  with one query key per entity; mutations invalidate the affected keys.
- **Auth**: `AuthProvider` subscribes to `onAuthStateChange`, hydrates the initial session and
  loads roles from `user_roles`; `isAdmin` drives conditional UI.
- **Route protection**: `src/routes/_authenticated/route.tsx` runs `beforeLoad` and redirects
  unauthenticated visitors to `/auth`, gating every child route in one place.
- **Design system**: semantic tokens in `src/styles.css` (academic navy + amber palette,
  Space Grotesk headings, DM Sans body); components use tokens only, no hardcoded colors.
- **SEO**: each route exports `head()` with a unique title, description and Open Graph tags.

---

## 8. Possible extensions

- CSV import/export of students and attendance
- Per-student profile page with attendance chart and marks trend
- Report-card PDF generation
- Class/section timetable module
- Email notifications for due tasks

---

## 9. License

Released for academic/project-submission use. Add a license file before wider distribution.
