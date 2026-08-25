# PH Healthcare Backend — Setup Roadmap & Folder Structure

Your phase plan is close to right — the overall four-phase shape (foundation → auth → shared services → business logic) is correct, and matches how experienced teams actually sequence a backend build. Two real issues found, both fixed below, plus the full roadmap mapped against your actual folder structure so you can use this as a literal setup checklist.

---

## 🔧 Corrections found

### 1. Real ordering bug: Step 6 (Public Auth APIs) is listed *before* Step 7 (Redis & Email)

This matters because your actual registration flow is **OTP-gated** — `register` stages the user in Redis and sends an OTP email, and only `verifyEmail` (which reads that Redis data) actually creates the `User` row. That means `Register` **directly depends on Redis and Nodemailer already being connected** — you can't build or test a working register endpoint without them.

**Fix: swap the order — Redis + Email infrastructure comes first, then the public auth APIs are built using them.**

### 2. Missing: Google OAuth isn't mentioned anywhere in the plan

Your actual `lib/` folder already has `googleAuth.ts`, but the phase plan's auth section only covers email/password. Since Google sign-in is a login-provider alongside credentials, it belongs in Phase 2, built right after (or alongside) the credentials-based login — not tucked away as an unplanned extra later.

**Everything else in your plan is correctly ordered** — seeding the Super Admin before building the public register/login APIs is actually fine here (not a bug), since your `seed.ts` calls Prisma directly with its own `bcrypt.hash(...)` call — it doesn't depend on the register endpoint existing at all, just on the schema and bcrypt being available, both of which are done by that point.

---

## ✅ Corrected Roadmap

## 🏗️ Phase 1: Foundation & Project Initialization

### Step 1 — Requirement Analysis & Database Schema Design
Finalize product spec, map the ERD, write the initial `schema.prisma` (`User`, `Role`, and every other core model).

### Step 2 — Project Setup (Boilerplate & Tooling)
TypeScript + Express + Prisma Client init, `.env.example` + `tsconfig.json`, Biome install & config.

### Step 3 — Centralized Error Handling & Utils
`AppError`, `globalErrorHandler`, `catchAsync`, `sendResponse`.

### Step 4 — Global Data Validation (Zod)
`validateRequest` middleware factory.

---

## 🔐 Phase 2: Authentication & Core Infrastructure

### Step 5 — Database Seeding (Super Admin)
`prisma/seed.ts` — creates the Super Admin directly via Prisma, independent of any API endpoint.

### Step 6 — Redis & Email Infrastructure *(moved up — was Step 7)*
Connect Redis, connect Nodemailer, build the OTP-storage pattern. **This has to exist before Register, not after**, since Register writes to Redis and sends mail as part of its own logic.

### Step 7 — Public Authentication APIs *(moved down — was Step 6)*
Register (OTP-gated, using Step 6's infrastructure), Verify Email + welcome email, Login (credentials), JWT issuance.

### Step 8 — Google OAuth *(newly added)*
`lib/googleAuth.ts` — verify Google ID tokens, find-or-create `User`, issue the same JWT shape as credentials login so downstream code (middleware, `req.user`) doesn't need to know or care which provider the user signed in with.

### Step 9 — Auth Middlewares
`checkAuth.ts` (token verification + role guard).

---

## 📁 Phase 3: Shared Core Services

### Step 10 — Storage Service (Multer + Cloudinary)
Memory-buffer Multer config, Cloudinary wrapper, reusable single-file and multi-file upload helpers.

### Step 11 — Profile Management API
Update-profile / avatar-upload endpoint — first real feature that proves auth + storage work together correctly.

---

## 💼 Phase 4: Business Logic & Advanced Features

### Step 12 — Feature-Specific Domain APIs
Your actual product modules: `doctor` (applications, verification), `appointment`, and anything else specific to PH Healthcare. This is where a team splits up and works in parallel, since infrastructure is done.

### Step 13 — Payment Gateway Integration
bKash (`lib/bkash.ts`) — deliberately built *after* Step 12, since payment needs a real appointment/booking to attach to; building it earlier means testing against fake data.

### Step 14 — Cron Jobs & Background Tasks
`lib/cron.ts` — e.g. deleting unverified doctor applications after 1 hour. Comes last because it references tables/statuses from features built in Step 12.

---

## 👥 Professional Team Workflows
(unchanged from your original — this part was already correct)
1. **API-First Design Contracts** — full endpoint/payload spec (Swagger/Postman) before coding, so frontend can build in parallel.
2. **Strict Git Branching** — `feat/auth-otp`, `feat/bkash-payment`, etc., named after this roadmap's steps.
3. **Environment Isolation** — never mix Local/Staging/Production config or data.

---

## 📂 Folder Structure — Mapped to Each Step

This is your actual project tree, annotated with which step creates each piece — use it as a literal build order for the boilerplate.

```
L2B7-Project-PH-Healthcare-Backend/
├── prisma/
│   ├── migrations/                    ← Step 1 (created by first `migrate dev`)
│   └── schema/                        ← Step 1
│
├── src/
│   └── app/
│       ├── config/
│       │   └── index.ts               ← Step 2 (env loading + validation)
│       │
│       ├── interfaces/
│       │   └── index.ts               ← Step 2 (shared TS types, e.g. IRequestUser)
│       │
│       ├── lib/
│       │   ├── prisma.ts              ← Step 2 (Prisma client instance)
│       │   ├── redis.ts               ← Step 6
│       │   ├── nodemailer.ts          ← Step 6
│       │   ├── googleAuth.ts          ← Step 8
│       │   ├── cloudinary.ts          ← Step 10
│       │   ├── multer.ts              ← Step 10
│       │   ├── bkash.ts               ← Step 13
│       │   └── cron.ts                ← Step 14
│       │
│       ├── middleware/
│       │   ├── validateRequest.ts     ← Step 4
│       │   ├── globalErrorHandler.ts  ← Step 3
│       │   ├── notFound.ts            ← Step 3
│       │   └── checkAuth.ts           ← Step 9
│       │
│       ├── module/
│       │   ├── auth/                  ← Step 7 (+ Step 8 Google routes)
│       │   ├── user/                  ← Step 11 (profile management)
│       │   ├── doctor/                ← Step 12
│       │   └── appointment/           ← Step 12 (+ Step 13 payment wiring)
│       │
│       ├── templates/                 ← Step 6 (OTP/welcome/reset .ejs files)
│       │
│       └── utils/
│           ├── AppError.ts            ← Step 3
│           ├── catchAsync.ts          ← Step 3
│           ├── sendResponse.ts        ← Step 3
│           ├── jwt.ts                 ← Step 7
│           └── seed.ts                ← Step 5
│
├── app.ts                             ← Step 2 (skeleton), wired up incrementally after
├── server.ts                          ← Step 2
├── .env / .env.example                ← Step 2
├── .gitignore                         ← Step 2
├── biome.json                         ← Step 2
├── prisma.config.ts                   ← Step 1
├── tsconfig.json                      ← Step 2
├── package.json                       ← Step 2
├── README.md                          ← Step 2 (skeleton), filled in throughout
└── Project Requirements.md            ← Step 1
```

**How to use this practically:** work top-to-bottom through the numbered steps above; each step tells you exactly which file(s) in this tree get created or filled in. By the time you reach Step 14, every folder in your actual project should be populated and the tree above should match your real repo one-to-one.
