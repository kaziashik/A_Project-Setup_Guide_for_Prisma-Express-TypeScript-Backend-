# JWT Authentication — Access & Refresh Tokens + Cookies

> Part 4–5 of my backend auth notes. Covers: signing access/refresh tokens,
> moving secrets to a config file, building a reusable token utility, and
> storing tokens in cookies.

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
