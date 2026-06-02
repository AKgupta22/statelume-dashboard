# Statelume — Frontend

> Trust infrastructure for land ownership records. Cryptographically verifiable, tamper-proof, independently auditable.

[![CI](https://github.com/your-org/statelume/actions/workflows/ci.yml/badge.svg)](https://github.com/your-org/statelume/actions/workflows/ci.yml)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.x-blue)](https://www.typescriptlang.org/)
[![Next.js](https://img.shields.io/badge/Next.js-14-black)](https://nextjs.org/)

---

## What this is

Statelume's frontend is a single Next.js application serving three distinct portals:

| Portal | URL | Who uses it |
|--------|-----|-------------|
| **Public verify** | `/verify/[record-id]` | Anyone verifying a land record |
| **Officer portal** | `/dashboard`, `/issue`, `/records` | Sub-Registrar officers issuing records |
| **Govt admin portal** | `/admin/*` | Government administrators managing officers |

The verify portal is server-side rendered — a buyer scanning a QR code sees the full record instantly with no loading spinner. The officer and admin portals are client-side dashboards protected by JWT middleware.

---

## Tech stack

| Layer | Technology | Why |
|-------|-----------|-----|
| Framework | Next.js 14 App Router | SSR for verify page, file-based routing, middleware auth |
| Language | TypeScript | Complex data types — wallet addresses, SHA-256 hashes, record schemas |
| Styling | Tailwind CSS + Shadcn/ui | Utility-first styling, accessible pre-built components |
| Server state | TanStack Query v5 | Loading/error states, cache invalidation, background refetch |
| Client state | Redux Toolkit + Redux Persist | Auth only — user, token, role survive browser refresh |
| HTTP client | Axios | Configured instance with base URL and auth interceptor |
| Forms | React Hook Form + Zod | Type-safe validation on complex land record submission forms |
| Cache storage | localForage (IndexedDB) | TanStack Query cache persisted beyond page refresh |

---

## Project structure

```
src/
├── app/                          # Next.js App Router
│   ├── (public)/                 # Public routes — no auth required
│   │   ├── verify/
│   │   │   └── [id]/
│   │   │       └── page.tsx      # Server component — SSR record verification
│   │   └── layout.tsx
│   ├── (officer)/                # Officer routes — OFFICER role required
│   │   ├── dashboard/page.tsx
│   │   ├── issue/page.tsx        # Issue new land record
│   │   ├── records/
│   │   │   ├── page.tsx          # All issued records
│   │   │   └── [id]/page.tsx     # Single record detail
│   │   └── layout.tsx
│   ├── (admin)/                  # Admin routes — GOVT_ADMIN role required
│   │   ├── dashboard/page.tsx
│   │   ├── officers/page.tsx     # Manage officer accounts
│   │   ├── records/page.tsx      # All records across all officers
│   │   ├── audit/page.tsx        # Full audit log
│   │   └── layout.tsx
│   ├── login/page.tsx            # Shared login page
│   ├── unauthorized/page.tsx
│   ├── layout.tsx                # Root layout — providers
│   └── middleware.ts             # JWT auth + role enforcement
│
├── components/
│   ├── ui/                       # Shadcn/ui base components
│   ├── verify/                   # Verify page components
│   │   ├── VerifyResult.tsx
│   │   ├── HashDisplay.tsx
│   │   └── StatusBadge.tsx
│   ├── records/                  # Record-related components
│   │   ├── RecordCard.tsx
│   │   ├── RecordTable.tsx
│   │   └── IssueRecordForm.tsx
│   ├── officers/                 # Officer management components
│   └── shared/                   # Navbar, Sidebar, PageHeader
│
├── lib/
│   ├── api.ts                    # Axios instance — base URL + auth interceptor
│   ├── queryClient.ts            # TanStack Query client + IndexedDB persister
│   └── utils.ts                  # cn(), formatDate(), truncateHash()
│
├── store/
│   ├── index.ts                  # Redux store + Redux Persist config
│   ├── authSlice.ts              # User, token, role
│   └── draftSlice.ts             # Unsaved record form draft
│
├── hooks/
│   ├── useAuth.ts                # Current user from Redux
│   ├── useRecords.ts             # TanStack Query hooks for records
│   └── useOfficers.ts            # TanStack Query hooks for officers
│
├── types/
│   ├── record.ts                 # LandRecord, RecordStatus types
│   ├── user.ts                   # User, UserRole types
│   └── api.ts                    # API response envelope types
│
└── middleware.ts                 # Route protection at Edge
```

---

## Getting started

### Prerequisites

- Node.js 20+
- npm 10+
- Statelume backend running (see `/backend/README.md`)

### Installation

```bash
# 1. Clone the repo
git clone https://github.com/your-org/statelume.git
cd statelume/frontend

# 2. Install dependencies
npm install

# 3. Set up environment variables
cp .env.example .env.local
```

### Environment variables

```bash
# .env.local

# Backend API URL
NEXT_PUBLIC_API_URL=http://localhost:8000
API_URL=http://localhost:8000          # used by server components

# JWT
NEXT_PUBLIC_TOKEN_KEY=statelume_token  # localStorage key for token

# Phase 2 — blockchain (leave empty for now)
NEXT_PUBLIC_POLYGON_EXPLORER=https://amoy.polygonscan.com
NEXT_PUBLIC_CONTRACT_ADDRESS=
```

`NEXT_PUBLIC_*` variables are exposed to the browser. Variables without the prefix are server-only — never sent to the client.

### Run in development

```bash
npm run dev
# → http://localhost:3000
```

---

## Available scripts

```bash
npm run dev          # start dev server with hot reload
npm run build        # production build
npm run start        # start production server locally
npm run lint         # ESLint
npm run type-check   # TypeScript check without building
```

---

## Authentication flow

```
User submits login form
  ↓
POST /auth/login → FastAPI returns { access_token, refresh_token, user }
  ↓
Redux authSlice stores user + tokens
Redux Persist writes to localStorage
  ↓
Axios interceptor attaches Authorization header to every request
  ↓
Middleware reads token from cookie on every navigation
Redirects to /login if missing, /unauthorized if wrong role
```

Tokens are stored in both Redux (for client components) and an httpOnly-like cookie (for middleware to read on the server side).

---

## Data fetching patterns

### Verify page — server component (no TanStack Query)

The verify page is the most important page for performance. It uses a Next.js server component to fetch and render the record server-side:

```tsx
// app/(public)/verify/[id]/page.tsx
async function VerifyPage({ params }) {
  const record = await fetch(`${process.env.API_URL}/verify/${params.id}`, {
    cache: "no-store",           // always fetch fresh — verification must be real-time
  }).then(r => r.json())

  return <VerifyResult record={record} />
}
```

The user's browser receives fully rendered HTML. No loading state. No blank page.

### Officer dashboard — TanStack Query for mutations

```tsx
// 'use client' components use TanStack Query
const mutation = useMutation({
  mutationFn: (data: IssueRecordInput) => api.post("/records", data),
  onSuccess: () => {
    queryClient.invalidateQueries({ queryKey: ["records"] })
    router.refresh()             // re-run server components on the page
  },
})
```

### Cache persistence

TanStack Query's cache is persisted to IndexedDB via localForage. Officers returning to the dashboard see cached records instantly while fresh data loads in the background.

To clear the cache on logout:

```ts
queryClient.clear()
await localForage.clear()
store.dispatch(logout())
```

---

## State management

Two tools, two jobs, zero overlap:

```
Redux Toolkit + Persist    →  who you are (auth, role, token)
TanStack Query + IndexedDB →  what data exists (records, officers)
```

Redux Persist uses `localStorage` (small, auth only). TanStack Query uses `localForage` (IndexedDB, handles larger datasets). Both are cleared on logout.

---

## Running with Docker

```bash
# from the repo root
docker compose up

# frontend available at http://localhost:3000
# backend at http://localhost:8000
# pgAdmin at http://localhost:5050
```

To build the frontend image independently:

```bash
docker build -t statelume-frontend ./frontend
docker run -p 3000:80 statelume-frontend
```

---

## CI/CD

Every push triggers the CI pipeline:

1. **Type check** — `tsc --noEmit` catches type errors
2. **Lint** — ESLint enforces code style
3. **Build** — `next build` fails if anything breaks

On merge to `main`, the deploy pipeline builds a Docker image, pushes to GitHub Container Registry, and deploys to Railway automatically.

See `.github/workflows/ci.yml` and `.github/workflows/deploy.yml`.

---

## Roadmap

- [x] Project scaffold and folder structure
- [x] Auth — login, JWT, role-based middleware
- [ ] Public verify page
- [ ] Officer portal — issue record form
- [ ] Officer portal — records list and detail
- [ ] Govt admin portal — officer management
- [ ] Govt admin portal — audit log
- [ ] Phase 2 — blockchain verification on verify page
- [ ] Phase 2 — Polygonscan deep link from certificate

---

## License

Private — Statelume POC. Not licensed for public use.