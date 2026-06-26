# Prisma + Express + TypeScript Backend — Project Setup Guide

A step-by-step note for setting up a backend project using **Express.js**, **TypeScript**, and **Prisma ORM** with **PostgreSQL**. Use this as a reference template whenever starting a new project with this stack.

---

## 📁 Final Folder Structure

```
PROJECT-PRISMA-PRESS-BACKEND/
├── prisma/
│   ├── migrations/
│   │   ├── 20220620090709_init/
│   │   └── migration_lock.toml
│   └── schema/
│       ├── enums.prisma
│       ├── profile.prisma
│       ├── schema.prisma
│       └── user.prisma
│
├── src/
│   ├── config/
│   │   └── index.ts
│   │
│   ├── lib/
│   │   └── prisma.ts
│   │
│   ├── modules/
│   │   └── user/
│   │       ├── user.controller.ts
│   │       ├── user.interface.ts
│   │       ├── user.route.ts
│   │       └── user.service.ts
│   │
│   ├── app.ts
│   └── server.ts
│
├── .env.example
├── .gitignore
├── package-lock.json
├── package.json
├── prisma.config.ts
└── tsconfig.json
```

Every other API resource (e.g. `post`, `comment`, `auth`) will get its own folder inside `src/modules/`, following the same pattern as the `user` module.

---

## Setting up the Project with Typescript, Express, Prisma And DB Connection

### Step 1: Initialize Git

```bash
git init
```

Create a `.gitignore` file in the project root and add:

```
node_modules
dist
build
```

This prevents dependencies and build output from being committed to GitHub.

---

## Step 2: Initialize the Node.js + TypeScript Project

Following the **Prisma Docs → Quickstart → Prisma Postgres** guide:

```bash
npm init
npm install typescript tsx @types/node --save-dev
npx tsc --init
```

`npm init` will ask a series of questions. Example answers used for this project:

| Prompt | Value |
|---|---|
| package name | `prisma-press-backend` |
| version | `0.0.1` |
| description | `server.ts` |
| entry point | `index.js` (default, not actually used since we run via `src/server.ts`) |
| type | `module` (so we can use ESM `import/export` syntax) |
| license | `ISC` |

This generates a `package.json` like:

```json
{
  "name": "prisma-press-backend",
  "version": "0.0.1",
  "description": "server.ts",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "author": "",
  "license": "ISC",
  "type": "module"
}
```

---

## Step 3: Configure `tsconfig.json`

```json
{
  "compilerOptions": {
    "outDir": "./dist",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "target": "ES2023",
    "types": ["node"],
    "sourceMap": true,
    "declaration": true,
    "declarationMap": true,
    "noUncheckedIndexedAccess": true,
    "strict": true,
    "isolatedModules": true,
    "noUncheckedSideEffectImports": true,
    "moduleDetection": "force",
    "skipLibCheck": true
  },
  // "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

### What each option means:

- **`outDir: "./dist"`** — where compiled JavaScript files go when you run `tsc` (build output folder).
- **`module: "ESNext"`** — use the latest ES module syntax (`import`/`export`) when compiling.
- **`moduleResolution: "bundler"`** — tells TypeScript to resolve imports the way modern bundlers do (works well with ESM + `tsx`).
- **`target: "ES2023"`** — compile down to ES2023 JavaScript features (modern Node.js supports this natively).
- **`types: ["node"]`** — include Node.js global types (e.g. `process`, `__dirname`).
- **`sourceMap: true`** — generates `.map` files so you can debug TypeScript directly even after compilation.
- **`declaration: true`** — generates `.d.ts` type declaration files for your compiled code.
- **`declarationMap: true`** — generates source maps for the declaration files too.
- **`noUncheckedIndexedAccess: true`** — when accessing an object/array by index (e.g. `arr[0]`), TypeScript treats the result as possibly `undefined` — forces you to handle missing values safely.
- **`strict: true`** — turns on all strict type-checking rules (recommended for catching bugs early).
- **`isolatedModules: true`** — ensures every file can be compiled independently (required by tools like `tsx`/`esbuild`).
- **`noUncheckedSideEffectImports: true`** — flags side-effect-only imports (e.g. `import "./file"`) if the file doesn't actually exist, catching typos.
- **`moduleDetection: "force"`** — forces every file to be treated as a module, even if it has no imports/exports.
- **`skipLibCheck: true`** — skips type-checking of `.d.ts` files inside `node_modules` (speeds up compilation).
- **`include`** — left commented out for now. We comment this out temporarily because `prisma.config.ts` (created in the next step) lives outside `src/`, and TypeScript would otherwise complain that it's not included in the project. Once everything is wired up, you can re-enable `"include": ["src/**/*"]` if needed, or simply leave it out so all `.ts` files in the project are picked up.
- **`exclude`** — folders TypeScript should ignore (`node_modules`, `dist`).

---

## Step 4: Initialize Prisma

```bash
npx prisma
npx prisma init --output ../generated/prisma
```

This creates a `prisma.config.ts` file like:

```ts
// This file was generated by Prisma, and assumes you have installed the following:
// npm install --save-dev prisma dotenv
import "dotenv/config";
import { defineConfig } from "prisma/config";

export default defineConfig({
  schema: "prisma/schema.prisma",
  migrations: {
    path: "prisma/migrations",
  },
  datasource: {
    url: process.env["DATABASE_URL"],
  },
});
```

> **Note:** The first time you see this file, TypeScript may show a red underline under `process.env["DATABASE_URL"]` because `prisma.config.ts` sits **outside** the `src/` folder, and your `tsconfig.json` `include` is scoped to `src/**/*`. That's why we commented out the `include` line in Step 3 — so TypeScript checks the whole project, including this config file.

---

## Step 5: Create a Prisma Postgres Database

1. Go to the **Prisma** dashboard and create a new project.
2. Skip the "deploy" step for now.
3. On the dashboard, click **Open Connect Setup**.
4. Copy the **connection string** shown there.
5. Paste it into a `.env` file in your project root:

```
DATABASE_URL="YOUR_CONNECTION_STRING_HERE"
```

---

## Step 6: Install Dependencies

### Dev Dependencies

```bash
npm install -D @types/cookie-parser @types/cors @types/express @types/jsonwebtoken @types/node @types/pg prisma tsx typescript
```

| Package | Purpose |
|---|---|
| `@types/cookie-parser` | Type definitions for `cookie-parser` |
| `@types/cors` | Type definitions for the `cors` middleware |
| `@types/express` | Type definitions for the Express.js framework |
| `@types/jsonwebtoken` | Type definitions for `jsonwebtoken` |
| `@types/node` | Type definitions for Node.js built-in modules |
| `@types/pg` | Type definitions for the PostgreSQL (`pg`) driver |
| `prisma` | Prisma CLI and schema management tool |
| `tsx` | Run TypeScript files directly without pre-compiling |
| `typescript` | TypeScript compiler (`tsc`) for type-checking and builds |

### Dependencies

```bash
npm install @prisma/adapter-pg @prisma/client bcryptjs cookie-parser cors dotenv express http-status jsonwebtoken pg
```

| Package | Purpose |
|---|---|
| `@prisma/adapter-pg` | Prisma driver adapter for PostgreSQL |
| `@prisma/client` | Prisma Client for running database queries |
| `bcryptjs` | Hash and compare passwords securely |
| `cookie-parser` | Parse cookies from incoming requests |
| `cors` | Enable Cross-Origin Resource Sharing |
| `dotenv` | Load environment variables from `.env` |
| `express` | Web framework for building the API/server |
| `http-status` | Named HTTP status codes (e.g. `httpStatus.OK`) |
| `jsonwebtoken` | Create and verify JWT auth tokens |
| `pg` | PostgreSQL driver for Node.js |

---

## Step 7: Set Up Express

Following the [Express docs](https://expressjs.com/), but written using **ES Modules** (`import`/`export`) instead of CommonJS, since `package.json` has `"type": "module"`.

### `src/server.ts`

```ts
import "dotenv/config";
import app from "./app";
import config from "./config";
import { prisma } from "./lib/prisma";

const PORT = config.port;

async function main() {
    try {
        await prisma.$connect();
        console.log("Connected to the database successfully.");
        app.listen(PORT, () => {
            console.log(`Server is running on port ${PORT}`);
        });
    } catch (error) {
        console.error("Error starting the server:", error);
        await prisma.$disconnect();
        process.exit(1);
    }
}

main();
```

This async `main()` function connects to the database first — if that fails, the server never starts, and we exit cleanly instead of running with a broken DB connection.

### `src/app.ts`

```ts
import cookieParser from "cookie-parser";
import cors from "cors";
import express, { Application, Request, Response } from "express";
import config from "./config";
import { userRoutes } from "./modules/user/user.route";

const app: Application = express();

app.use(cors({
    origin: config.app_url,
    credentials: true,
}));

app.use(express.json());
app.use(express.urlencoded({ extended: true }));
app.use(cookieParser());

app.get("/", (req: Request, res: Response) => {
    res.send("Hello, World!");
});

app.use("/api/users", userRoutes);

export default app;
```

---

## Step 8: Add NPM Scripts

In `package.json`:

```json
"scripts": {
  "dev": "tsx watch src/server.ts",
  "build": "tsc",
  "start": "node dist/server.js"
}
```

- **`dev`** — runs the server with `tsx watch`, which auto-restarts on file changes (great for development).
- **`build`** — compiles TypeScript to JavaScript into `dist/` using `tsc`.
- **`start`** — runs the compiled JavaScript (used in production).

Run the dev server:

```bash
npm run dev
```

---

## Step 9: Connect Prisma Client to the App

Create `src/lib/prisma.ts`:

```ts
import { PrismaPg } from "@prisma/adapter-pg";
import "dotenv/config";
import { PrismaClient } from "../../generated/prisma/client";

const connectionString = `${process.env.DATABASE_URL}`;

const adapter = new PrismaPg({ connectionString });
const prisma = new PrismaClient({ adapter });

export { prisma };
```

> **Note:** At this point you'll get an error on `import { PrismaClient } from "../../generated/prisma/client";` — because the Prisma Client hasn't been generated yet. That's expected. We need to:
> 1. Write the Prisma schema/models (Step 11).
> 2. Run the migration (Step 12).
> 3. Run `npx prisma generate` to actually create the client at `generated/prisma`.

---

## Step 10: Create the Config File

Centralize all environment variables in one place for cleaner imports across the app.

Create `src/config/index.ts`:

```ts
import dotenv from "dotenv";
import path from "path";

dotenv.config({ path: path.join(process.cwd(), ".env") });

export default {
    port: process.env.PORT,
    database_url: process.env.DATABASE_URL,
    app_url: process.env.APP_URL,
    bcrypt_salt_rounds: process.env.BCRYPT_SALT_ROUNDS,
    jwt_access_secret: process.env.JWT_ACCESS_SECRET,
    jwt_refresh_secret: process.env.JWT_REFRESH_SECRET,
    jwt_access_expires_in: process.env.JWT_ACCESS_EXPIRES_IN,
    jwt_refresh_expires_in: process.env.JWT_REFRESH_EXPIRES_IN,
}
```

### Explanation: `dotenv.config({ path: path.join(process.cwd(), ".env") })`

- `process.cwd()` returns the **current working directory** — i.e. the folder you ran `npm run dev` from (normally your project root).
- `path.join(process.cwd(), ".env")` builds an absolute path to the `.env` file, regardless of which file is importing this config or how deep it is in the folder structure.
- `dotenv.config({ path: ... })` loads the variables from that `.env` file into `process.env`.

This is more reliable than just calling `dotenv.config()` with no path, because that default behavior depends on the current working directory matching where `.env` actually lives — using `process.cwd()` explicitly avoids "it works on my machine but not when run from another folder" bugs.

---

## Step 11: Middleware Setup in `app.ts`

```ts
app.use(cors({
    origin: config.app_url,
    credentials: true,
}));

app.use(express.json());
app.use(express.urlencoded({ extended: true }));
app.use(cookieParser());
```

### What each middleware does:

- **`cors({ origin, credentials })`** — Cross-Origin Resource Sharing. By default, browsers block requests from a different origin (e.g. frontend on `localhost:3000` calling backend on `localhost:5000`). This middleware allows requests **only from the specified `origin`** (your frontend's URL), and `credentials: true` allows cookies/auth headers to be sent along with cross-origin requests.
- **`express.json()`** — parses incoming requests with a `Content-Type: application/json` body, so you can access `req.body` as a JS object.
- **`express.urlencoded({ extended: true })`** — parses incoming requests with URL-encoded payloads (e.g. traditional HTML form submissions). `extended: true` allows parsing of nested objects/arrays in the form data.
- **`cookieParser()`** — parses the `Cookie` header on incoming requests and populates `req.cookies`, so you can read cookies (e.g. for refresh tokens) easily.

---

## Step 12: Multi-File Prisma Schema

By default Prisma uses a single `schema.prisma` file. If you prefer splitting models across multiple files, Prisma supports a **multi-file schema** setup:

```
prisma/
├── migrations/
├── schema/
│   ├── enums.prisma
│   ├── profile.prisma
│   ├── schema.prisma
│   └── user.prisma
```

To enable this:

1. Create a `prisma/schema/` folder.
2. Move `schema.prisma` (with the `generator` and `datasource` blocks) into that folder.
3. Split your models into separate `.prisma` files inside the same folder (e.g. `user.prisma`, `profile.prisma`, `enums.prisma`). Prisma automatically merges all `.prisma` files inside the `schema/` folder when generating/migrating.
4. Update `prisma.config.ts` so the `schema` path points to the folder instead of the single file:

```ts
export default defineConfig({
  schema: "prisma/schema",
  // ...
});
```

This keeps large schemas organized as your number of models grows.

---

## Step 13: Write the Prisma Models

### `prisma/schema/schema.prisma`

```prisma
// This is your Prisma schema file,
// learn more about it in the docs: https://pris.ly/d/prisma-schema

// Get a free hosted Postgres database in seconds: `npx create-db`

generator client {
  provider = "prisma-client"
  output   = "../../generated/prisma"
}

datasource db {
  provider = "postgresql"
}
```

### `prisma/schema/enums.prisma`

```prisma
enum ActiveStatus {
  ACTIVE
  BLOCKED
}

enum Role {
  USER
  AUTHOR
  ADMIN
}
```

### `prisma/schema/user.prisma`

```prisma
model User {
  id           String       @id @default(uuid())
  name         String       @db.VarChar(255)
  email        String       @unique
  password     String
  activeStatus ActiveStatus @default(ACTIVE)
  role         Role         @default(USER)
  createdAt    DateTime     @default(now())
  updatedAt    DateTime     @updatedAt
  profile      Profile?

  @@map("users")
}
```

### `prisma/schema/profile.prisma`

```prisma
model Profile {
  id           String   @id @default(uuid())
  profilePhoto String?
  bio          String?

  userId String @unique
  user   User   @relation(fields: [userId], references: [id])

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  @@map("profiles")
}
```

---

## Step 14: Run Migrations & Generate the Client

```bash
npx prisma migrate dev
```

This creates the migration files and applies them to your database.

```bash
npx prisma studio
```

Opens a visual database browser in your browser to view/edit data.

```bash
npx prisma generate
```

Generates the Prisma Client based on your schema, creating the `generated/prisma` folder. This is what fixes the earlier import error in `src/lib/prisma.ts` — once generated, `PrismaClient` becomes a real, importable module.

---

## Step 15: Create Your First API Module (User)

Wire up the route in `app.ts`:

```ts
app.use("/api/users", userRoutes);
```

Create the module folder structure:

```
src/modules/user/
├── user.controller.ts   // handles req/res, calls service functions
├── user.interface.ts     // TypeScript types/interfaces for the module
├── user.route.ts         // Express router, defines endpoints
└── user.service.ts       // business logic, talks to Prisma/DB
```

Repeat this same folder pattern (`<name>.controller.ts`, `<name>.interface.ts`, `<name>.route.ts`, `<name>.service.ts`) for every new resource (e.g. `post`, `comment`, `auth`) inside `src/modules/<name>/`.

---

## ✅ Quick Recap Checklist

- [ ] `git init` + `.gitignore`
- [ ] `npm init` + install TypeScript + `tsc --init`
- [ ] Configure `tsconfig.json`
- [ ] `npx prisma init --output ../generated/prisma`
- [ ] Create Prisma Postgres DB → copy connection string into `.env`
- [ ] Install dev + prod dependencies
- [ ] Set up `server.ts` and `app.ts`
- [ ] Add `dev` / `build` / `start` scripts
- [ ] Create `src/lib/prisma.ts`
- [ ] Create `src/config/index.ts`
- [ ] Set up middleware (`cors`, `express.json`, `urlencoded`, `cookieParser`)
- [ ] (Optional) Switch to multi-file Prisma schema
- [ ] Write models (`User`, `Profile`, enums)
- [ ] `npx prisma migrate dev` → `npx prisma studio` → `npx prisma generate`
- [ ] Build first module (`user`) with controller/service/route/interface
