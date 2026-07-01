# JWT Authentication — Access & Refresh Tokens + Cookies

> Parts 4–7 of my backend auth notes. Covers: signing access/refresh tokens,
> moving secrets to a config file, building a reusable token utility, storing
> tokens in cookies, fetching the logged-in user's profile, and an auth
> middleware for role-based access control (RBAC).

---

## Table of Contents

1. [Signing the Access & Refresh Tokens](#1-signing-the-access--refresh-tokens)
2. [Cleanup: One Shared Payload](#2-cleanup-one-shared-payload)
3. [Moving Secrets into a Config File](#3-moving-secrets-into-a-config-file)
4. [What `!` Means in TypeScript (Non-null Assertion)](#4-what--means-in-typescript-non-null-assertion)
5. [DRY Refactor: A Reusable `createToken` Utility](#5-dry-refactor-a-reusable-createtoken-utility)
6. [Setting the Tokens as Cookies](#6-setting-the-tokens-as-cookies)
7. [Cookie Option Reference](#7-cookie-option-reference)
8. [Gotchas & Things to Fix Later](#8-gotchas--things-to-fix-later)
9. [`.gitignore` Reminder](#9-gitignore-reminder)
10. [Getting the Logged-in User's Profile](#10-getting-the-logged-in-users-profile)
11. [Auth Middleware for Role-Based Access Control](#11-auth-middleware-for-role-based-access-control)
12. [Why This Logic Lives in Middleware](#12-why-this-logic-lives-in-middleware)
13. [Gotchas in Sections 10–11](#13-gotchas-in-sections-10-11)

---

## 1. Signing the Access & Refresh Tokens

`jwt.sign(payload, secret, options)` takes three things:

- **payload** — the data you want to embed in the token (user info, etc.)
- **secret** — the key used to sign (and later verify) the token
- **options** — extra settings, like expiry time

```js
const accessToken = jwt.sign(
  {
    id: user.id,
    name: user.name,
    email: user.email,
    role: user.role,
  },
  "acccessecret",
  { expiresIn: "1d" }
);

const refreshToken = jwt.sign(
  {
    id: user.id,
    name: user.name,
    email: user.email,
    role: user.role,
  },
  "refreToken",
  { expiresIn: "7d" }
);

return { accessToken, refreshToken };
```

**Why two tokens?**
- `accessToken` — short-lived (1 day here), sent with every request to prove who you are.
- `refreshToken` — long-lived (7 days here), used **only** to get a new access token once the old one expires, so the user doesn't have to log in again every day.

Login response looks like this:

```json
{
  "success": true,
  "statusCode": 200,
  "message": "User logged in successfully",
  "data": {
    "accessToken": "eyJhbGciOi...",
    "refreshToken": "eyJhbGciOi..."
  }
}
```

---

## 2. Cleanup: One Shared Payload

Both tokens use the exact same payload — no need to write it twice.

```js
const jwtPayload = {
  id: user.id,
  name: user.name,
  email: user.email,
  role: user.role,
};

const accessToken = jwt.sign(jwtPayload, "acccessecret", { expiresIn: "1d" });
const refreshToken = jwt.sign(jwtPayload, "refreToken", { expiresIn: "7d" });
```

---

## 3. Moving Secrets into a Config File

Hardcoding secrets directly in the code (`"acccessecret"`) is bad practice —
they should come from environment variables, never be committed to git, and
be easy to change per environment (dev/staging/prod).

**`.env`**
```env
PORT=5000
APP_URL=http://localhost:5000
DATABASE_URL="postgres://<username>:<password>@<host>:5432/<database>?sslmode=require"
BCRYPT_SALT_ROUNDS=10

JWT_ACCESS_SECRET='acccessecret'
JWT_REFRESH_SECRET='refreToken'
JWT_ACCESS_EXPIRES_IN='1d'
JWT_REFRESH_EXPIRES_IN='7d'
```

> ⚠️ Use long, random strings for real secrets — `acccessecret` is fine for
> learning, but in a real project generate something like
> `openssl rand -hex 32` and never reuse it across access/refresh.

**`config/index.ts`**
```ts
import dotenv from "dotenv";
import path from "path";

dotenv.config({ path: path.join(process.cwd(), ".env") });

export default {
  port: process.env.PORT,
  database_url: process.env.DATABASE_URL,
  bcrypt_salt_rounds: process.env.BCRYPT_SALT_ROUNDS,
  app_url: process.env.APP_URL,

  jwt_access_secret: process.env.JWT_ACCESS_SECRET!,
  jwt_refresh_secret: process.env.JWT_REFRESH_SECRET!,
  jwt_access_expires_in: process.env.JWT_ACCESS_EXPIRES_IN!,
  jwt_refresh_expires_in: process.env.JWT_REFRESH_EXPIRES_IN!,
};
```

Usage:
```ts
const accessToken = jwt.sign(jwtPayload, config.jwt_access_secret, {
  expiresIn: config.jwt_access_expires_in,
} as SignOptions);

const refreshToken = jwt.sign(jwtPayload, config.jwt_refresh_secret, {
  expiresIn: config.jwt_refresh_expires_in,
} as SignOptions);

return { accessToken, refreshToken };
```

---

## 4. What `!` Means in TypeScript (Non-null Assertion)

```ts
jwt_access_secret: process.env.JWT_ACCESS_SECRET!,
```

`process.env.JWT_ACCESS_SECRET` has the type `string | undefined`, because
TypeScript can't know for sure that the env variable will actually exist at
runtime. The `!` is the **non-null assertion operator** — it tells TypeScript:

> "Trust me, this will not be `undefined` or `null`. Stop warning me about it."

It does **not** check anything at runtime, it just silences the compiler. If
the env variable is genuinely missing, `config.jwt_access_secret` will
actually be `undefined`, and `jwt.sign()` will throw a runtime error.

A safer alternative used in real projects:

```ts
function getEnv(key: string): string {
  const value = process.env[key];
  if (!value) {
    throw new Error(`Missing required environment variable: ${key}`);
  }
  return value;
}

jwt_access_secret: getEnv("JWT_ACCESS_SECRET"),
```

This fails fast with a clear error message instead of silently passing
`undefined` into `jwt.sign()`. Good upgrade for later — `!` is fine while
you're learning.

---

## 5. DRY Refactor: A Reusable `createToken` Utility

Access token and refresh token creation is the same logic, just different
secret + expiry. Pull it into a shared utility.

**`utils/jwt.ts`**
```ts
import jwt, { JwtPayload, SignOptions } from "jsonwebtoken";

const createToken = (
  payload: JwtPayload,
  secret: string,
  expiresIn: SignOptions["expiresIn"]
) => {
  return jwt.sign(payload, secret, { expiresIn } as SignOptions);
};

export const jwtUtils = {
  createToken,
};
```

**Usage:**
```ts
const accessToken = jwtUtils.createToken(
  jwtPayload,
  config.jwt_access_secret,
  config.jwt_access_expires_in as SignOptions["expiresIn"]
);

const refreshToken = jwtUtils.createToken(
  jwtPayload,
  config.jwt_refresh_secret,
  config.jwt_refresh_expires_in as SignOptions["expiresIn"]
);
```

---

## 6. Setting the Tokens as Cookies

Done in `auth.controller.ts` — **not** in the service. The controller is
responsible for the HTTP request/response, so anything related to headers,
status codes, or cookies belongs there. The service just returns data; it
shouldn't know about `res`.

```ts
res.cookie("accessToken", accessToken, {
  httpOnly: true,
  secure: false,
  sameSite: "none",
  maxAge: 1000 * 60 * 60 * 24, // 24 hours
});

res.cookie("refreshToken", refreshToken, {
  httpOnly: true,
  secure: false,
  sameSite: "none",
  maxAge: 1000 * 60 * 60 * 24, // 24 hours
});
```

---

## 7. Cookie Option Reference

| Option | Value here | What it actually does |
|---|---|---|
| `httpOnly` | `true` | The cookie can't be read or touched by JavaScript in the browser (`document.cookie` won't show it). This is the main defense against **XSS** stealing your tokens. Always `true` for auth tokens. |
| `secure` | `false` | If `true`, the cookie is only ever sent over **HTTPS**. `false` means it'll also be sent over plain HTTP — convenient for local dev, but should be `true` in production. |
| `sameSite` | `"none"` | Controls whether the cookie is sent on **cross-site** requests. `"none"` = sent always (needed when frontend and backend are on different domains/ports). `"lax"` = sent on top-level navigation only. `"strict"` = never sent cross-site. |
| `maxAge` | `1000 * 60 * 60 * 24` | How long the cookie lives, **in milliseconds**, before the browser deletes it. `1000 * 60 * 60 * 24` = 1000ms × 60s × 60min × 24hr = 1 day. |

---

## 8. Gotchas & Things to Fix Later

- **`secure: false` + `sameSite: "none"` doesn't work in modern browsers.**
  Browsers require `secure: true` whenever `sameSite: "none"` is used —
  otherwise they silently drop the cookie. For local dev over HTTP, use
  `sameSite: "lax"` instead, and switch to `secure: true, sameSite: "none"`
  once you're on HTTPS in production. Good pattern:
  ```ts
  secure: config.node_env === "production",
  sameSite: config.node_env === "production" ? "none" : "lax",
  ```

- **`refreshToken` cookie's `maxAge` doesn't match its real expiry.**
  The refresh token itself expires in `7d`, but the cookie is set to expire
  in `1` day (`1000 * 60 * 60 * 24`). The cookie will disappear from the
  browser a full 6 days before the token inside it actually expires. It
  should be:
  ```ts
  maxAge: 1000 * 60 * 60 * 24 * 7, // 7 days, matches refresh token expiry
  ```

- **Naming**: `JWT_REFRESH_SECRET='refreToken'` — the *value* of the secret
  is literally the string `refreToken`, which reads like it's a token, not a
  secret key. Just a naming trap to watch out for, not a bug.

---

## 9. `.gitignore` Reminder

Before pushing this repo, make sure `.env` is **never** committed:

```gitignore
.env
.env.*
!.env.example
```

Commit an `.env.example` instead, with placeholder values only, so anyone
cloning the repo knows what variables they need to set:

```env
PORT=
APP_URL=
DATABASE_URL=
BCRYPT_SALT_ROUNDS=
JWT_ACCESS_SECRET=
JWT_REFRESH_SECRET=
JWT_ACCESS_EXPIRES_IN=
JWT_REFRESH_EXPIRES_IN=
```

---

## 10. Getting the Logged-in User's Profile

Goal: a `GET /me` route that reads the access token from the cookie, verifies
it, and returns that user's profile. Step by step:

**Step 1 — Get the cookie out of the request.**
`req.cookies` only exists if `cookie-parser` middleware is registered on the
app (otherwise `req.cookies` is `undefined`). Add this once, in `app.ts`:
```ts
import cookieParser from "cookie-parser";
app.use(cookieParser());
```

**Step 2 — Pull out the access token.**
```ts
const { accessToken } = req.cookies;
```
The browser automatically attaches cookies on every request to the same
domain, so the frontend doesn't need to manually send this — that's the
whole point of using cookies instead of, say, localStorage.

**Step 3 — Verify the token.**
```ts
const verifyToken = (token: string, secret: string) => {
  try {
    const verifiedToken = jwt.verify(token, secret);
    return verifiedToken;
  } catch (error: any) {
    console.log("Token Verification failed:", error);
    throw new Error(error.message);
  }
};
```
`jwt.verify` checks two things at once: the **signature** (was this token
actually signed with our secret, i.e. not forged) and the **expiry** (has
`expiresIn` passed). If either check fails, it throws — which is why this is
wrapped in `try/catch` and re-thrown as a normal `Error`.

**Step 4 — Narrow the TypeScript type.**
```ts
if (typeof verifiedToken === "string") {
  throw new Error(verifiedToken);
}
```
`jwt.verify()`'s return type is `JwtPayload | string`. It only returns a
plain `string` in an edge case — if the token had been signed with a raw
string payload instead of an object — which never happens in our app since
we always sign `jwtPayload` (an object). This check exists purely to satisfy
TypeScript so it knows `verifiedToken` is a `JwtPayload` object (with `.id`,
`.role`, etc.) from this point onward.

**Step 5 — Use the decoded `id` to fetch the real user from the database.**
```ts
const profile = await userService.getMyProfileFromDB(verifiedToken.id);
```
```ts
const getMyProfileFromDB = async (userId: string) => {
  const user = await prisma.user.findUniqueOrThrow({
    where: { id: userId },
    omit: { password: true },
    include: { profile: true },
  });
  return user;
};
```
- `findUniqueOrThrow` — throws automatically if no user matches that id,
  instead of silently returning `null` (less `if (!user)` boilerplate).
- `omit: { password: true }` — explicitly excludes the password hash from
  the result, so it never accidentally leaks into the API response.
- `include: { profile: true }` — joins the related `profile` table/record.

**Step 6 — Send the response.**
```ts
res.send({
  success: true,
  StatusCodes: httpsStatus.OK,
  message: "User Profile fetched successfully",
  data: { profile },
});
```

**Full flow, in one sentence:** cookie → `cookie-parser` reads it onto
`req.cookies` → pull out `accessToken` → `jwt.verify` checks it's genuine and
not expired → decoded payload gives you the `id` → query the DB with that
`id` → return the profile.

---

## 11. Auth Middleware for Role-Based Access Control

**Step 1 — Tell TypeScript that `req.user` is allowed to exist.**
```ts
declare global {
  namespace Express {
    interface Request {
      user?: {
        email: string;
        name: string;
        id: string;
        role: Role;
      };
    }
  }
}
```
Express's built-in `Request` type has no `user` field by default, so writing
`req.user = {...}` later would be a TypeScript error. This block uses
**declaration merging** to extend Express's own `Request` interface
globally, adding an optional `user` property everywhere in the project.

**Step 2 — `auth` is a middleware *factory*, not a middleware.**
```ts
const auth = (...requiredRoles: Role[]) => {
  return catchAsync(async (req: Request, res: Response, next: NextFunction) => {
    // ...
  });
};
```
`auth` itself isn't middleware — it's a function that *returns* one. The
`...requiredRoles` rest parameter lets you call it flexibly per route:
- `auth()` → no roles required, just "must be logged in"
- `auth(Role.ADMIN)` → only admins
- `auth(Role.ADMIN, Role.USER)` → either role

**Step 3 — Get the token from cookie or header, whichever exists.**
```ts
const token = req.cookies.accessToken
  ? req.cookies.accessToken
  : req.headers.authorization?.startsWith("Bearer")
    ? req.headers.authorization?.split(" ")[1]
    : req.headers.authorization;
```
This supports two kinds of clients with one middleware: a browser frontend
(sends the cookie automatically) and a non-browser client like Postman or a
mobile app (sends `Authorization: Bearer <token>` instead, since it has no
cookie jar). `"Bearer <token>".split(" ")[1]` strips the `"Bearer "` prefix
to get just the token string.

**Step 4 — Verify the token, then check the role.**
```ts
const verifiedToken = jwtutils.verifyToken(token, config.jwt_access_secret);
const { email, name, id, role } = verifiedToken.data as JwtPayload;

if (requiredRoles.length && !requiredRoles.includes(role)) {
  throw new Error("Forbidden. You don't have permission to access this resource");
}
```
If `requiredRoles` is empty (e.g. `auth()`), `requiredRoles.length` is `0`,
so the whole check is skipped — any logged-in user passes. If roles *were*
specified, the user's `role` must be one of them, or it's a `403 Forbidden`.

**Step 5 — Re-check the user against the database, not just the token.**
```ts
const user = await prisma.user.findUnique({ where: { id, email, name, role } });
if (!user) throw new Error("user not found. Please log in again.");
if (user.activeStatus === "BLOCKED") {
  throw new Error("your Account has been blocked. please contact support");
}
```
A valid JWT only proves the token *was* issued correctly — it says nothing
about whether the account still exists or has since been blocked/deleted.
This DB check catches: account deleted after the token was issued, or
account blocked by an admin while the access token is still "valid" (since
it can't be revoked once issued — it just expires naturally).

**Step 6 — Attach the verified user to the request, then continue.**
```ts
req.user = { email, name, id, role };
next();
```
Every controller *after* this middleware can now read `req.user` directly,
already verified, without re-checking the token itself.

**Step 7 — Use it on a route.**
```ts
router.get("/me", auth(Role.ADMIN, Role.AUTHOR, Role.USER), userController.getMyProfile);
```

---

## 12. Why This Logic Lives in Middleware

Auth/RBAC checking is pulled out into middleware instead of being written
inside every controller, for a few reasons:

- **Reusability.** One `auth()` function protects every route that needs it —
  `/me`, `/posts`, `/admin/users`, whatever — instead of copy-pasting the
  same token-checking code into every single controller.
- **Runs before the controller, on purpose.** Express runs middleware in
  order: `auth()` → controller. If the token is missing, invalid, or the
  role doesn't match, the request is rejected *before* it ever reaches the
  controller or touches business logic — no wasted DB queries for unrelated
  data, no risk of a controller forgetting to check auth.
- **Single source of truth.** If you need to change *how* auth works later
  (switch from cookies to headers only, add a new check, change the error
  message), you edit one file — not every controller that happens to need
  auth.
- **Separation of concerns.** The controller's job is "fetch this profile and
  send it back." It shouldn't also be responsible for parsing tokens,
  checking roles, and querying for blocked status — that's a different
  concern, and middleware is exactly the place Express gives you for that.
- **Parameterized per-route.** Because `auth(...requiredRoles)` is a
  factory, the *same* middleware function can enforce different rules on
  different routes (`auth(Role.ADMIN)` here, `auth()` there) without writing
  separate code for each.

---

## 13. Gotchas in Sections 10–11

- **`verifyToken`'s return shape is inconsistent between section 10 and 11.**
  In section 10, `verifyToken` returns the decoded payload directly (or
  throws):
  ```ts
  const verifiedToken = jwtutils.verifyToken(accessToken, config.jwt_access_secret);
  if (typeof verifiedToken === "string") { throw new Error(verifiedToken); }
  // verifiedToken.id used directly
  ```
  But in section 11, the code calls it as if it returns a result *object*
  with `.success`, `.error`, and `.data`:
  ```ts
  if (!verifiedToken.success) { throw new Error(verifiedToken.error); }
  const { email, name, id, role } = verifiedToken.data as JwtPayload;
  ```
  These can't both be true for the same `verifyToken` function. Pick one
  shape and use it everywhere — e.g. simplest is "throws on failure, returns
  the payload directly on success" (section 10's version), and then section
  11 should be:
  ```ts
  const verifiedToken = jwtutils.verifyToken(token, config.jwt_access_secret) as JwtPayload;
  const { email, name, id, role } = verifiedToken;
  ```

- **`prisma.user.findUnique({ where: { id, email, name, role } })` will likely
  throw a Prisma error.** `findUnique` only accepts fields that are actually
  unique in the schema (a single `@unique` field, or an exact `@@unique([...])`
  composite). `name` and `role` aren't unique on their own, so combining all
  four into `where` only works if there's a matching `@@unique([id, email,
  name, role])` constraint — which is unusual. This almost certainly just
  needs to be:
  ```ts
  const user = await prisma.user.findUnique({ where: { id } });
  ```

- **Errors are thrown as plain `Error`, not HTTP-aware errors.** `throw new
  Error("Forbidden...")` doesn't carry a status code, so unless your global
  error handler specifically maps that message to `403`, it'll likely fall
  back to a generic `500`. Worth using (or building) an `AppError` class that
  carries a `statusCode` alongside the message, so "Forbidden" actually comes
  back as `403` and "not logged in" as `401`.

