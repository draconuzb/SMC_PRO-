# Modme CRM — Project Architecture

## System Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        MODME CRM                                │
│                  Educational Center CRM                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌─────────────┐    ┌──────────────┐    ┌─────────────────┐   │
│   │  React SPA  │───▶│  NestJS API  │───▶│  PostgreSQL 16  │   │
│   │  Port 5173  │    │  Port 3000   │    │  Port 5432      │   │
│   │  Ant Design │    │  Prisma ORM  │    │  37 tables      │   │
│   └─────────────┘    └──────┬───────┘    └─────────────────┘   │
│                             │                                   │
│                             ▼                                   │
│                      ┌─────────────┐                            │
│                      │  Redis 7    │                            │
│                      │  Port 6379  │                            │
│                      └─────────────┘                            │
└─────────────────────────────────────────────────────────────────┘
```

## Monorepo Structure

```
modme-crm/
│
├── package.json                 # Root workspace config
├── pnpm-workspace.yaml          # pnpm workspace definitions
├── turbo.json                   # Turborepo build pipelines
├── docker-compose.yml           # PostgreSQL + Redis services
├── .env.example                 # Environment variables template
│
├── packages/
│   └── shared/                  # @modme/shared — shared between FE & BE
│       └── src/
│           ├── types/           # TypeScript interfaces
│           │   ├── auth.ts      #   Role, LoginRequest/Response, JwtPayload
│           │   ├── branch.ts    #   Branch, CreateBranchDto
│           │   ├── student.ts   #   Student, StudentGroup, CreateStudentDto
│           │   ├── teacher.ts   #   Teacher, TeacherSalary
│           │   ├── group.ts     #   Group, DayType, GroupStatus
│           │   ├── lead.ts      #   Lead, LeadStatus, CreateLeadDto
│           │   ├── finance.ts   #   Payment, Withdrawal, Expense, Debtor
│           │   ├── attendance.ts #  AttendanceRecord, BulkAttendanceDto
│           │   ├── settings.ts  #   Course, Room, Tag, Holiday, SmsSettings
│           │   └── gamification.ts # Product, Order
│           ├── constants/       # Enums & labels
│           │   ├── roles.ts     #   ROLE_LABELS, ADMIN_ROLES
│           │   ├── lead-status.ts # LEAD_COLUMNS, LEAD_SOURCES
│           │   ├── days.ts      #   DAY_TYPE_LABELS, WEEKDAYS, ODD/EVEN
│           │   ├── payment.ts   #   PAYMENT_METHOD_LABELS
│           │   └── currency.ts  #   UZS, +998, PHONE_LENGTH
│           └── utils/           # Helper functions
│               ├── phone.ts     #   formatPhone, normalizePhone, isValidUzPhone
│               ├── currency.ts  #   formatCurrency, formatCurrencyWithLabel
│               └── date.ts      #   formatDate, formatDateTime, getMonthYear
│
├── apps/
│   ├── api/                     # NestJS Backend
│   └── web/                     # React Frontend
```

## Backend Architecture (apps/api/)

```
apps/api/
├── prisma/
│   ├── schema.prisma            # 37 database models
│   ├── seed.ts                  # Default data (branch, CEO, rooms, course)
│   └── migrations/              # Auto-generated migrations
│
└── src/
    ├── main.ts                  # App bootstrap, CORS, Swagger, ValidationPipe
    ├── app.module.ts            # Root module — imports all 26 modules
    │
    ├── common/                  # Shared infrastructure
    │   ├── decorators/
    │   │   ├── current-user.decorator.ts    # @CurrentUser() — extract user from JWT
    │   │   ├── current-branch.decorator.ts  # @CurrentBranch() — extract x-branch-id header
    │   │   └── roles.decorator.ts           # @Roles('CEO','ADMIN') — metadata decorator
    │   ├── guards/
    │   │   ├── jwt-auth.guard.ts            # JWT Bearer token validation
    │   │   └── roles.guard.ts               # Role-based access control
    │   ├── interceptors/
    │   │   └── branch-scope.interceptor.ts  # Multi-tenancy: auto-scope by branchId
    │   └── dto/
    │       └── pagination.dto.ts            # page, limit, search, sortBy, sortOrder
    │
    └── modules/                 # Feature modules (26 total)
        ├── prisma/              # [GLOBAL] PrismaClient singleton
        ├── auth/                # [IMPLEMENTED] Login, JWT, refresh tokens
        ├── branch/              # [IMPLEMENTED] Branch CRUD, multi-tenancy
        ├── user/                # [IMPLEMENTED] User CRUD, role management
        ├── teacher/             # [STUB] Teacher profiles, salary calculation
        ├── student/             # [STUB] Student profiles, balance, enrollment
        ├── group/               # [STUB] Group management, student enrollment
        ├── course/              # [STUB] Course & subcourse CRUD
        ├── lead/                # [STUB] Lead Kanban, status transitions
        ├── attendance/          # [STUB] Bulk attendance, reports
        ├── finance/             # [STUB] Payments, withdrawals, expenses, salaries, debtors
        ├── reminder/            # [STUB] Reminders: overdue/today/future
        ├── rating/              # [STUB] Student ratings by group/period
        ├── report/              # [STUB] Conversion funnel, attendance, logs
        ├── gamification/        # [STUB] Shop products, orders, coins
        ├── sms/                 # [STUB] SMS integration (Eskiz.uz)
        ├── voip/                # [STUB] VoIP call tracking
        ├── room/                # [STUB] Room CRUD
        ├── tag/                 # [STUB] Tag CRUD
        ├── grade/               # [STUB] Grade settings & records
        ├── exam/                # [STUB] Exam CRUD & results
        ├── form/                # [STUB] Dynamic form builder
        ├── blog/                # [STUB] Blog/news CRUD
        ├── holiday/             # [STUB] Holiday CRUD
        ├── schedule/            # [STUB] Aggregate schedule view
        └── settings/            # [STUB] General/SMS/VoIP settings
```

### Module Pattern

Every module follows the same structure:

```
modules/<name>/
├── <name>.module.ts             # NestJS @Module declaration
├── <name>.controller.ts         # REST endpoints with Swagger docs
├── <name>.service.ts            # Business logic + Prisma queries
└── dto/                         # Request validation (class-validator)
    ├── create-<name>.dto.ts
    ├── update-<name>.dto.ts
    └── query-<name>.dto.ts
```

### Request Flow

```
Client Request
    │
    ▼
┌─────────────────┐
│   JWT Guard      │  ← Validates Bearer token
├─────────────────┤
│   Roles Guard    │  ← Checks user.role against @Roles()
├─────────────────┤
│  BranchScope     │  ← Reads x-branch-id header, sets req.branchId
│  Interceptor     │     CEO can access all branches
├─────────────────┤
│  ValidationPipe  │  ← Validates DTO with class-validator
├─────────────────┤
│   Controller     │  ← Route handler, extracts @CurrentUser, @CurrentBranch
├─────────────────┤
│   Service        │  ← Business logic
├─────────────────┤
│   Prisma ORM     │  ← Database queries (auto-scoped by branchId)
├─────────────────┤
│  PostgreSQL      │  ← Data storage
└─────────────────┘
```

### Authentication Flow

```
┌──────────┐     POST /auth/login       ┌──────────┐
│  Client  │ ──────────────────────────▶ │  Server  │
│          │     {phone, password}       │          │
│          │                             │          │
│          │  ◀──────────────────────── │          │
│          │  {accessToken(15m),         │          │
│          │   refreshToken(7d),         │          │
│          │   user}                     │          │
│          │                             │          │
│          │  GET /api/* + Bearer token  │          │
│          │  + x-branch-id header       │          │
│          │ ──────────────────────────▶ │          │
│          │                             │          │
│          │  401 Unauthorized           │          │
│          │ ◀────────────────────────── │          │
│          │                             │          │
│          │  POST /auth/refresh         │          │
│          │  {refreshToken}             │          │
│          │ ──────────────────────────▶ │          │
│          │                             │          │
│          │  {new accessToken,          │          │
│          │   new refreshToken}         │          │
│          │ ◀────────────────────────── │          │
└──────────┘                             └──────────┘
```

## Database Schema (37 models)

```
┌─────────────────────────────────────────────────────────────────────┐
│                         BRANCH (Multi-Tenant Root)                  │
│  id, name, address, phone, isActive                                 │
└────────┬──────────┬──────────┬──────────┬──────────┬───────────────┘
         │          │          │          │          │
    ┌────▼───┐ ┌───▼────┐ ┌──▼───┐ ┌───▼───┐ ┌───▼────┐
    │  User  │ │ Course │ │ Room │ │  Tag  │ │  Lead  │
    │  Branch│ │        │ │      │ │       │ │        │
    └───┬────┘ └───┬────┘ └──┬───┘ └───┬───┘ └───┬────┘
        │          │         │         │          │
   ┌────┴─────┐    │         │    ┌────┴───┐      │ convert
   │          │    │         │    │        │      ▼
┌──▼───┐ ┌───▼────▼─────────▼──┐ │  ┌─────▼──────────┐
│Teach-│ │       GROUP          │ │  │    STUDENT      │
│ er   │ │  name, dayType,      │ │  │  balance, coins │
│      │─┤  startTime, room,    ├─┤  │                 │
└──┬───┘ │  capacity            │ │  └──┬──┬──┬──┬──┬──┘
   │     └──────────┬───────────┘ │     │  │  │  │  │
   │                │             │     │  │  │  │  │
   │     ┌──────────▼──────────┐  │     │  │  │  │  │
   │     │    GroupStudent     │◀─┼─────┘  │  │  │  │
   │     │  price, status,     │  │        │  │  │  │
   │     │  startDate          │  │        │  │  │  │
   │     └─────────────────────┘  │        │  │  │  │
   │                              │        │  │  │  │
   │  ┌────────────┐  ┌──────────┐│  ┌─────▼┐ │  │  │
   │  │ Attendance │  │  Grade   ││  │ Pay- │ │  │  │
   │  │ date,status│  │  Record  ││  │ ment │ │  │  │
   │  └────────────┘  └──────────┘│  └──────┘ │  │  │
   │                              │           │  │  │
   │  ┌────────────┐  ┌──────────┐│  ┌────────▼┐ │  │
   │  │   Exam     │  │ Online   ││  │ Comment │ │  │
   │  │   Result   │  │ Lesson   ││  └─────────┘ │  │
   │  └────────────┘  └──────────┘│              │  │
   │                              │  ┌───────────▼┐ │
   │  ┌────────────┐              │  │ SmsRecord  │ │
   │  │  Salary    │◀─────────────┘  └────────────┘ │
   │  │ month,year │                                │
   │  └────────────┘                 ┌──────────────▼┐
   │                                 │  CallRecord   │
   │  ┌────────────┐                 └───────────────┘
   │  │  Teacher   │
   │  │ Attendance │
   │  └────────────┘
   │
   │  ┌────────────┐
   └──│  Work      │
      │  Schedule  │
      └────────────┘

┌─────────────────────────────────────────────┐
│  STANDALONE TABLES (branch-scoped)          │
├─────────────────────────────────────────────┤
│  Payment, Withdrawal, Expense               │
│  Reminder, Rating, Order, Product           │
│  SmsSettings, VoipSettings, Holiday         │
│  Form, BlogPost, Discount, Log             │
└─────────────────────────────────────────────┘
```

### Key Relationships

| Relationship          | Type        | Via                      |
|-----------------------|-------------|--------------------------|
| Branch ↔ User         | Many-to-Many| UserBranch               |
| Student ↔ Group       | Many-to-Many| GroupStudent (price,status)|
| Course → Group        | One-to-Many | group.courseId            |
| Teacher → Group       | One-to-Many | group.teacherId          |
| Room → Group          | One-to-Many | group.roomId             |
| Lead → Student        | One-to-One  | student.leadId           |
| Lead ↔ Tag            | Many-to-Many| LeadTag                  |
| Group ↔ Tag           | Many-to-Many| GroupTag                 |

## Frontend Architecture (apps/web/)

```
apps/web/src/
├── main.tsx                     # ReactDOM.createRoot entry
├── App.tsx                      # Providers: QueryClient → Auth → Antd → Router
│
├── lib/                         # Core infrastructure
│   ├── axios.ts                 # Axios instance + JWT interceptor + refresh logic
│   ├── queryClient.ts           # TanStack React Query config
│   └── i18n.ts                  # i18next (uz/en/ru)
│
├── styles/
│   └── theme.ts                 # Ant Design 5 theme customization
│
├── features/                    # Feature-specific API + state
│   └── auth/
│       ├── api.ts               # loginApi, refreshTokenApi, logoutApi, getMeApi
│       ├── store.ts             # AuthContext + AuthProvider (JWT + localStorage)
│       └── hooks.ts             # useAuth(), useRequireAuth()
│
├── components/                  # Shared components
│   └── ProtectedRoute.tsx       # Auth guard — redirects to /login if unauthenticated
│
├── layouts/
│   ├── AuthLayout.tsx           # Centered card layout (login page)
│   ├── DashboardLayout.tsx      # Sidebar + Header + Content (Ant Layout)
│   └── components/
│       ├── Sidebar.tsx          # Navigation menu (all 30+ items)
│       └── Header.tsx           # Branch selector + Language + User menu
│
├── routes/
│   └── index.tsx                # React Router v6 config (lazy-loaded pages)
│
├── pages/                       # 33 page components
│   ├── auth/LoginPage.tsx           # ✅ Full implementation
│   ├── dashboard/DashboardPage.tsx  # Stub
│   ├── leads/LeadsKanbanPage.tsx    # Stub
│   ├── teachers/
│   │   ├── TeacherListPage.tsx      # Stub
│   │   └── TeacherProfilePage.tsx   # Stub
│   ├── groups/
│   │   ├── GroupListPage.tsx        # Stub
│   │   └── GroupDetailPage.tsx      # Stub
│   ├── students/
│   │   ├── StudentListPage.tsx      # Stub
│   │   └── StudentProfilePage.tsx   # Stub
│   ├── reminders/RemindersPage.tsx  # Stub
│   ├── rating/RatingPage.tsx        # Stub
│   ├── attendance/AttendanceReportPage.tsx       # Stub
│   ├── teacher-attendance/TeacherAttendancePage.tsx # Stub
│   ├── finance/
│   │   ├── PaymentsPage.tsx         # Stub
│   │   ├── WithdrawPage.tsx         # Stub
│   │   ├── ExpensesPage.tsx         # Stub
│   │   ├── SalariesPage.tsx         # Stub
│   │   └── DebtorsPage.tsx          # Stub
│   ├── reports/
│   │   ├── ConversionPage.tsx       # Stub
│   │   ├── AttendanceReportsPage.tsx # Stub
│   │   ├── LeadsReportsPage.tsx     # Stub
│   │   ├── StudentsLeftPage.tsx     # Stub
│   │   └── LogsPage.tsx            # Stub
│   ├── gamification/
│   │   ├── OrdersPage.tsx           # Stub
│   │   └── ShopPage.tsx             # Stub
│   └── settings/
│       ├── SmsSettingsPage.tsx      # Stub
│       ├── VoipSettingsPage.tsx     # Stub
│       ├── GradeSettingsPage.tsx    # Stub
│       ├── CeoSettingsPage.tsx      # Stub
│       ├── OfficeSettingsPage.tsx   # Stub
│       ├── FormsPage.tsx            # Stub
│       ├── BlogPage.tsx             # Stub
│       └── TagsPage.tsx             # Stub
│
└── public/locales/              # i18n translations
    ├── uz/translation.json      # O'zbek
    ├── en/translation.json      # English
    └── ru/translation.json      # Русский
```

### Frontend Provider Stack

```
<QueryClientProvider>           ← TanStack Query (server state)
  <AuthProvider>                ← JWT tokens + user profile + branch
    <ConfigProvider theme={}>   ← Ant Design theme
      <BrowserRouter>           ← React Router v6
        <AppRoutes />           ← Lazy-loaded route tree
      </BrowserRouter>
    </ConfigProvider>
  </AuthProvider>
</QueryClientProvider>
```

### Route Structure

```
/login                          → AuthLayout > LoginPage
/                               → ProtectedRoute > DashboardLayout
├── /dashboard                  → DashboardPage
├── /leads                      → LeadsKanbanPage
├── /teachers                   → TeacherListPage
├── /teachers/:id               → TeacherProfilePage
├── /groups                     → GroupListPage
├── /groups/:id                 → GroupDetailPage
├── /students                   → StudentListPage
├── /students/:id               → StudentProfilePage
├── /reminders                  → RemindersPage
├── /rating                     → RatingPage
├── /attendance                 → AttendanceReportPage
├── /teacher-attendance         → TeacherAttendancePage
├── /finance
│   ├── /payments               → PaymentsPage
│   ├── /withdraw               → WithdrawPage
│   ├── /expenses               → ExpensesPage
│   ├── /salaries               → SalariesPage
│   └── /debtors                → DebtorsPage
├── /reports
│   ├── /conversion             → ConversionPage
│   ├── /attendance             → AttendanceReportsPage
│   ├── /leads                  → LeadsReportsPage
│   ├── /students-left          → StudentsLeftPage
│   └── /logs                   → LogsPage
├── /gamification
│   ├── /orders                 → OrdersPage
│   └── /shop                   → ShopPage
└── /settings
    ├── /sms                    → SmsSettingsPage
    ├── /voip                   → VoipSettingsPage
    ├── /grade                  → GradeSettingsPage
    ├── /ceo                    → CeoSettingsPage
    ├── /office                 → OfficeSettingsPage
    ├── /forms                  → FormsPage
    ├── /blog                   → BlogPage
    └── /tags                   → TagsPage
```

## API Endpoints

### Auth (✅ Implemented)
| Method | Endpoint            | Auth  | Description                |
|--------|---------------------|-------|----------------------------|
| POST   | /auth/login         | No    | Login with phone + password|
| POST   | /auth/refresh       | No    | Refresh access token       |
| POST   | /auth/logout        | JWT   | Invalidate refresh token   |
| GET    | /auth/me            | JWT   | Get current user + branches|

### Branches (✅ Implemented)
| Method | Endpoint            | Roles      | Description         |
|--------|---------------------|------------|---------------------|
| GET    | /branches           | CEO, ADMIN | List all branches   |
| POST   | /branches           | CEO        | Create branch       |
| GET    | /branches/:id       | Any        | Get branch          |
| PATCH  | /branches/:id       | CEO, ADMIN | Update branch       |
| DELETE | /branches/:id       | CEO        | Soft delete branch  |

### Users (✅ Implemented)
| Method | Endpoint            | Auth | Description               |
|--------|---------------------|------|---------------------------|
| GET    | /users              | JWT  | List users (branch-scoped)|
| POST   | /users              | JWT  | Create user in branch     |
| GET    | /users/:id          | JWT  | Get user detail           |
| PATCH  | /users/:id          | JWT  | Update user               |
| DELETE | /users/:id          | JWT  | Soft delete user          |

### Stub Modules (25 modules — endpoints defined, logic TODO)

| Module       | Key Endpoints                                           |
|--------------|---------------------------------------------------------|
| Teachers     | GET /teachers, GET /teachers/:id, GET /teachers/:id/salary |
| Students     | GET /students, GET/PATCH /students/:id, GET /students/:id/groups |
| Groups       | GET/POST /groups, GET/PATCH /groups/:id, POST /groups/:id/students |
| Courses      | GET/POST /courses, GET/PATCH/DELETE /courses/:id        |
| Leads        | GET/POST /leads, PATCH /leads/:id/status, POST /leads/:id/convert |
| Attendance   | POST /attendance/bulk, GET /attendance/report           |
| Finance      | GET/POST /finance/payments, /withdrawals, /expenses, /salaries, /debtors |
| Reminders    | GET/POST /reminders, PATCH /reminders/:id/complete      |
| Ratings      | GET /ratings                                            |
| Reports      | GET /reports/conversion, /attendance, /leads, /students-left, /logs |
| Gamification | GET/POST /gamification/products, /orders                |
| SMS          | POST /sms/send, GET /sms/history                        |
| VoIP         | GET /voip/calls                                         |
| Rooms        | GET/POST /rooms, GET/PATCH/DELETE /rooms/:id            |
| Tags         | GET/POST /tags, PATCH/DELETE /tags/:id                  |
| Grade        | GET/PATCH /grade/settings                               |
| Exams        | GET/POST /exams, GET /exams/:id/results                 |
| Forms        | GET/POST /forms, GET/PATCH/DELETE /forms/:id            |
| Blog         | GET/POST /blog, GET/PATCH/DELETE /blog/:id              |
| Holidays     | GET/POST /holidays, DELETE /holidays/:id                |
| Schedule     | GET /schedule                                           |
| Settings     | GET/PATCH /settings/general, /sms, /voip                |

## Multi-Tenancy Strategy

```
┌─────────────────────────────────────────────┐
│ Every API request includes:                  │
│   Authorization: Bearer <jwt>                │
│   x-branch-id: <number>                     │
├─────────────────────────────────────────────┤
│                                              │
│  BranchScopeInterceptor:                     │
│  ├── Reads x-branch-id from header           │
│  ├── CEO role → can access ANY branch        │
│  ├── Other roles → must have valid branchId  │
│  └── Attaches branchId to req.branchId       │
│                                              │
│  Service layer:                              │
│  └── prisma.student.findMany({               │
│        where: { branchId: req.branchId }     │
│      })                                      │
│                                              │
│  Shared DB, data isolated by branchId column │
└─────────────────────────────────────────────┘
```

## Tech Stack

| Layer         | Technology                        | Purpose                    |
|---------------|-----------------------------------|----------------------------|
| Frontend      | React 18 + TypeScript             | UI framework               |
| UI Library    | Ant Design 5                      | Components, theme          |
| Routing       | React Router 6                    | Client-side navigation     |
| Server State  | TanStack Query 5                  | API caching, mutations     |
| Forms         | React Hook Form + Zod             | Validation                 |
| Kanban        | @hello-pangea/dnd                 | Drag-and-drop (Leads)      |
| Charts        | @ant-design/charts                | Dashboard graphs           |
| i18n          | i18next                           | uz/en/ru translations      |
| HTTP          | Axios                             | API client + interceptors  |
| Backend       | NestJS 10                         | REST API framework         |
| ORM           | Prisma 5                          | Database access            |
| Auth          | Passport JWT                      | Token authentication       |
| Database      | PostgreSQL 16                     | Primary data store         |
| Cache         | Redis 7                           | Future: caching, queues    |
| Monorepo      | pnpm workspaces + Turborepo       | Multi-package management   |
| Containerized | Docker Compose                    | PostgreSQL + Redis         |

## Development Commands

```bash
# Start infrastructure
docker-compose up -d              # PostgreSQL + Redis

# Install dependencies
pnpm install                      # All packages

# Database
pnpm db:migrate                   # Run Prisma migrations
pnpm db:seed                      # Seed default data
pnpm db:studio                    # Open Prisma Studio GUI

# Development
pnpm dev                          # Start both API + Web
pnpm dev:api                      # Start only backend (port 3000)
pnpm dev:web                      # Start only frontend (port 5173)

# Build
pnpm build                        # Build all packages

# API Documentation
# Open http://localhost:3000/api/docs for Swagger UI
```

## Default Seed Data

| Entity  | Data                                              |
|---------|---------------------------------------------------|
| Branch  | "Main Branch", Tashkent, +998901234567            |
| CEO     | Admin CEO, +998900000000, password: admin123      |
| Rooms   | "Room 1" (cap 20), "Room 2" (cap 15)             |
| Course  | "English", 500,000 UZS                            |

## Implementation Status

| Phase | Module                    | Status        |
|-------|---------------------------|---------------|
| A     | Auth + JWT + Refresh      | ✅ Complete    |
| A     | Branch CRUD               | ✅ Complete    |
| A     | User CRUD + RBAC          | ✅ Complete    |
| A     | BranchScope Interceptor   | ✅ Complete    |
| A     | Frontend Login + Auth     | ✅ Complete    |
| A     | Frontend Protected Routes | ✅ Complete    |
| A     | Frontend BranchSelector   | ✅ Complete    |
| B     | Courses / Rooms / Tags    | ⬜ Stub        |
| C     | Teachers + Groups         | ⬜ Stub        |
| D     | Students                  | ⬜ Stub        |
| E     | Leads (Kanban)            | ⬜ Stub        |
| F     | Attendance + Schedule     | ⬜ Stub        |
| G     | Finance                   | ⬜ Stub        |
| H     | Dashboard                 | ⬜ Stub        |
| I     | Reminders + Rating        | ⬜ Stub        |
| J     | Reports                   | ⬜ Stub        |
| K     | SMS + VoIP                | ⬜ Stub        |
| L     | Gamification              | ⬜ Stub        |
| M     | Grade / Exam / Forms / Blog| ⬜ Stub       |
| N     | CEO Settings + Polish     | ⬜ Stub        |
