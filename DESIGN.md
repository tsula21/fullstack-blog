# Design spec

Everything the app is, before it is built. Contracts only — no implementation.
If something here turns out wrong during a phase, change *this file* first, then
the code.

**Contents:** [1 Scope](#1-scope) · [2 Actors](#2-actors) · [3 Data model](#3-data-model) ·
[4 Validation](#4-validation) · [5 API contracts](#5-api-contracts) · [6 Frontend routes](#6-frontend-routes) ·
[7 Frontend files](#7-frontend-files) · [8 Layout & CSS](#8-layout--css) · [9 The CORS + cookie trap](#9-the-cors--cookie-trap) ·
[10 Backend files](#10-backend-files) · [11 Environment](#11-environment) · [12 Build order](#12-build-order) ·
[13 Decisions](#13-decisions)

---

## 1 Scope

A Reddit-shaped text board, stripped to the bone.

**In scope**

- Sign up, sign in, sign out
- Create a post (body text only)
- Delete **your own** post
- Read all posts — including as a signed-out guest
- View your own profile, read-only

**Explicitly out of scope** (do not build these, even if tempted)

Likes · comments · other users' profiles · editing posts · titles · images ·
tags · search · pagination UI · password reset · email verification · avatars ·
dark mode · a design system.

Styling is not a goal. Plain, legible, and equally usable on a phone is the whole
bar. Markup should be simple enough to write by hand.

---

## 2 Actors

| Actor | Can |
| ----- | --- |
| **Guest** | Read the post feed. Sign in. Sign up. |
| **User** | Everything a guest can, plus: create posts, delete own posts, view own profile. |

There is no admin, no moderation, no roles column. If a role ever appears, this
section is where it gets added first.

---

## 3 Data model

Two tables. That is the entire model.

### User

| Column | Type | Constraints | Notes |
| ------ | ---- | ----------- | ----- |
| `id` | `String` | PK, `@default(uuid())` | UUID, not an int — safe to expose in URLs, no row-count leak |
| `name` | `VarChar(50)` | required | Display name. Not unique. |
| `email` | `VarChar(254)` | required, **unique** | 254 is the RFC 5321 maximum. Stored lowercased. |
| `passwordHash` | `Text` | required | bcrypt output, cost 12. **Never** the password. |
| `dateOfBirth` | `Date` | required | A calendar date, no time, no timezone |
| `createdAt` | `Timestamp(3)` | `@default(now())` | An instant in time — timezone matters |

### Post

| Column | Type | Constraints | Notes |
| ------ | ---- | ----------- | ----- |
| `id` | `String` | PK, `@default(uuid())` | |
| `body` | `VarChar(500)` | required | The whole post. No title. |
| `createdAt` | `Timestamp(3)` | `@default(now())`, indexed DESC | Index matches the only query: newest first |
| `authorId` | `String` | FK → `User.id`, `ON DELETE CASCADE` | Delete a user, their posts go with them |

### Schema

```prisma
model User {
  id           String   @id @default(uuid())
  name         String   @db.VarChar(50)
  email        String   @unique @db.VarChar(254)
  passwordHash String
  dateOfBirth  DateTime @db.Date
  createdAt    DateTime @default(now())

  posts        Post[]
}

model Post {
  id        String   @id @default(uuid())
  body      String   @db.VarChar(500)
  createdAt DateTime @default(now())

  authorId  String
  author    User     @relation(fields: [authorId], references: [id], onDelete: Cascade)

  @@index([createdAt(sort: Desc)])
}
```

`dateOfBirth` is `DATE` and `createdAt` is `TIMESTAMP(3)` **on purpose**. A
birthday is the same day everywhere on earth; a signup happened at one absolute
instant. Storing a birthday as a timestamp is how people born on the 1st end up
displaying as the 31st for users west of you.

---

## 4 Validation

Validate on the server. Always. Client-side checks are a courtesy to the user,
never a security boundary — anything the browser enforces, `curl` ignores.

| Field | Rule | Failure |
| ----- | ---- | ------- |
| `name` | trimmed, 1–50 chars | 400 |
| `email` | contains `@`, ≤254 chars, lowercased before storing | 400 |
| `email` | already registered | **409**, not 400 |
| `dateOfBirth` | parses as `YYYY-MM-DD`, is in the past, age ≥ 13 | 400 |
| `password` | 8–128 chars | 400 |
| `confirmPassword` | **client-side only** — never sent to the server | — |
| `body` | trimmed, 1–500 chars | 400 |

`confirmPassword` exists to catch typos in the browser. The server has no use for
it and should not receive it.

### Error shape

Every non-2xx response, without exception:

```jsonc
{
  "error": "Human-readable message",
  "fields": { "email": "Already registered" }   // optional, only for 400/409
}
```

One shape means the frontend has one error path instead of nine.

---

## 5 API contracts

Base path `/api`. All bodies are JSON. All authenticated calls send the session
cookie automatically — no `Authorization` header anywhere.

Eight endpoints: four auth, three posts, one health.

### `GET /api/health`

Liveness probe. No auth, no database.

`200` → `{ "ok": true }`

---

### `POST /api/auth/register`

```jsonc
// request
{
  "name": "Irakli",
  "email": "me@example.com",
  "dateOfBirth": "1998-04-21",
  "password": "hunter2hunter2"
}
```

| Status | Meaning |
| ------ | ------- |
| `201` | Created. Sets the session cookie. Returns the public user. |
| `400` | Validation failed |
| `409` | Email already registered |

```jsonc
// 201
{ "id": "8f3…", "name": "Irakli", "email": "me@example.com",
  "dateOfBirth": "1998-04-21", "createdAt": "2026-08-12T11:56:26.031Z" }
```

`passwordHash` never appears in a response body. Not once, not ever.

---

### `POST /api/auth/login`

```jsonc
{ "email": "me@example.com", "password": "hunter2hunter2" }
```

| Status | Meaning |
| ------ | ------- |
| `200` | Sets the cookie, returns the public user |
| `401` | `{ "error": "Invalid email or password" }` |

Unknown email and wrong password return the **identical** 401. Distinguishing
them hands an attacker a list of which emails are registered.

---

### `POST /api/auth/logout`

No body. Clears the cookie.

`204` → no content. Succeeds even if you weren't signed in.

---

### `GET /api/auth/me`

Reads the cookie. This is how the frontend knows who you are after a refresh.

| Status | Meaning |
| ------ | ------- |
| `200` | The public user object |
| `401` | No cookie, or expired/invalid token |

A 401 here is **normal**, not an error to log noisily — every guest hits it once
on page load.

---

### `GET /api/posts`

Public. The feed. Newest first, capped at 50.

`200`:

```jsonc
[
  { "id": "a1…", "body": "hello world",
    "createdAt": "2026-08-12T12:00:00.000Z",
    "author": { "id": "8f3…", "name": "Irakli" } }
]
```

The author object carries `id` and `name` only. The frontend compares
`post.author.id` against the signed-in user's id to decide whether to render a
delete button. Never include the author's email.

Ordering is `createdAt DESC` — which is exactly what the index in §3 exists for.

---

### `POST /api/posts` 🔒

```jsonc
{ "body": "hello world" }
```

| Status | Meaning |
| ------ | ------- |
| `201` | The created post, in feed shape |
| `400` | Empty or >500 chars |
| `401` | Not signed in |

**`authorId` is not in the request body and must never be accepted from one.** It
is `req.userId`, set by `requireAuth` from the verified token. If the client can
name the author, the client can post as anybody.

---

### `DELETE /api/posts/:id` 🔒

| Status | Meaning |
| ------ | ------- |
| `204` | Deleted |
| `401` | Not signed in |
| `404` | No such post **or it isn't yours** |

Do the ownership check and the delete as one atomic statement:

```ts
const { count } = await prisma.post.deleteMany({
  where: { id: req.params.id, authorId: req.userId },
});
return count === 0 ? res.status(404).json({ error: "Post not found" })
                   : res.status(204).end();
```

`count === 0` covers both "doesn't exist" and "not yours" — and returning the
same 404 for both means you don't leak the existence of other people's posts.

This is better than find-then-check-then-delete: it is one round trip and there
is no window between the check and the delete for anything to change.

### Cookie

| Property | Dev | Production |
| -------- | --- | ---------- |
| name | `token` | `token` |
| `httpOnly` | `true` | `true` |
| `secure` | `false` | `true` |
| `sameSite` | `lax` | `none` |
| `maxAge` | 7 days | 7 days |
| `path` | `/` | `/` |

The dev/prod difference is not cosmetic — see §9.

---

## 6 Frontend routes

**Two routes.** Sign-in lives in the navbar and sign-up in a modal, so there is
no `/login` page and no `/register` page at all.

| Route | Access | Contents |
| ----- | ------ | -------- |
| `/` | public | Composer (signed in only) + the post feed |
| `/profile` | 🔒 protected | Your name, email, date of birth, join date — read-only |

`/profile` is *your* profile. Clicking an author's name goes nowhere; there are
no other-user profiles in this version.

### Navbar

Always visible, on both routes.

- **Guest:** site name · `email` input · `password` input · **Sign in** button · **Sign up** button (opens the modal)
- **User:** site name · `Home` · `Profile` · `Hi, {name}` · **Sign out** button

### Sign-up modal

A native `<dialog>` element. No library, no portal, no focus-trap code — the
browser gives you the backdrop, `Esc` to close, and focus management for free.

Five fields, all required:

| Field | Input |
| ----- | ----- |
| Name | `<input type="text" maxlength="50">` |
| Email | `<input type="email" maxlength="254">` |
| Date of birth | `<input type="date">` — the OS date picker, no library |
| Password | `<input type="password" minlength="8">` |
| Confirm password | `<input type="password">` — compared in the browser, never sent |

On success: close the dialog, set the user in context, stay on `/`.

### Composer

On `/`, only when signed in. A `<textarea maxlength="500">`, a character counter,
and a **Publish** button that disables while the request is in flight. New post
gets prepended to the list without a refetch.

### Post

```
Irakli · 12 Aug 2026, 14:05                    [Delete]
hello world
```

The `[Delete]` button renders only when `post.author.id === user.id`. That is a
convenience, **not** the security check — the server checks again, and the server
is the one that counts.

---

## 7 Frontend files

Thirteen files. `ProtectedRoute` lives inside `App.tsx` rather than earning a file.

```
client/
├── index.html                      1
└── src/
    ├── main.tsx                    2   mount, BrowserRouter, AuthProvider
    ├── App.tsx                     3   routes + ProtectedRoute
    ├── styles.css                  4   the whole stylesheet, ~80 lines
    ├── api/
    │   ├── client.ts               5   the ONLY file containing fetch
    │   ├── auth.ts                 6   register / login / logout / me
    │   └── posts.ts                7   list / create / remove
    ├── context/
    │   └── AuthContext.tsx         8   user, loading, and the auth actions
    ├── components/
    │   ├── Navbar.tsx              9   sign-in form or user menu
    │   ├── SignUpModal.tsx        10   the <dialog>
    │   └── PostCard.tsx           11   one post + conditional delete
    └── pages/
        ├── Home.tsx               12   composer + feed
        └── Profile.tsx            13   read-only fields
```

### `api/client.ts` — one chokepoint

**No other file is allowed to call `fetch`.** Every request goes through one
wrapper that owns the base URL, `credentials: "include"`, the JSON headers, and
turning a non-2xx response into a thrown `Error` carrying the server's message.

This is worth it for one reason: in Phase 7, every request suddenly needs a
different base URL and cookie handling. You edit one file instead of ten.

```ts
const BASE = import.meta.env.VITE_API_URL ?? "http://localhost:3000";

export async function api<T>(path: string, init?: RequestInit): Promise<T> {
  const res = await fetch(`${BASE}/api${path}`, {
    ...init,
    credentials: "include",                       // ← the cookie. Non-negotiable.
    headers: { "Content-Type": "application/json", ...init?.headers },
  });
  if (res.status === 204) return undefined as T;
  const data = await res.json().catch(() => ({}));
  if (!res.ok) throw new Error(data.error ?? `Request failed (${res.status})`);
  return data as T;
}
```

### `AuthContext.tsx` — and the `loading` flag

```ts
{ user: PublicUser | null,
  loading: boolean,            // ← the important one
  signIn, signUp, signOut }
```

On mount it calls `GET /api/auth/me` and sets `loading` false when the answer
arrives, 200 or 401.

**Why `loading` exists:** without it, a hard refresh on `/profile` renders with
`user === null` — because `/me` is still in flight — so `ProtectedRoute` concludes
you're a guest and redirects you home. You get bounced off your own profile every
single refresh. `loading` makes the route render nothing until the truth is known.
Everyone hits this bug. You get to skip it.

```tsx
function ProtectedRoute({ children }) {
  const { user, loading } = useAuth();
  if (loading) return null;                       // ← say nothing until you know
  return user ? children : <Navigate to="/" replace />;
}
```

---

## 8 Layout & CSS

One stylesheet, roughly 80 lines, no framework, no preprocessor, no CSS modules.

The entire responsive strategy is **`flex-wrap` plus one media query at 640px**.

```css
:root { --gap: 12px; --border: #ddd; }

body   { margin: 0; font: 16px/1.5 system-ui, sans-serif; color: #111; }
main   { max-width: 640px; margin: 0 auto; padding: var(--gap); }

.nav   { display: flex; flex-wrap: wrap; gap: var(--gap);
         align-items: center; padding: var(--gap);
         border-bottom: 1px solid var(--border); }
.nav form { display: flex; flex-wrap: wrap; gap: 8px; }
.spacer{ margin-left: auto; }

input, textarea, button { font: inherit; padding: 8px; }
textarea { width: 100%; box-sizing: border-box; min-height: 80px; }

.post  { border: 1px solid var(--border); padding: var(--gap);
         margin-bottom: var(--gap); }
.meta  { font-size: 14px; color: #666;
         display: flex; justify-content: space-between; gap: 8px; }
.error { color: #b00; font-size: 14px; }

@media (max-width: 640px) {
  .nav, .nav form { flex-direction: column; align-items: stretch; }
  .spacer { margin-left: 0; }
}
```

That's it. On desktop the navbar is a row; under 640px it stacks. **No hamburger
menu** — a hamburger is three days of work to hide four links.

Rules of thumb for the markup:

- Semantic elements over divs: `<nav> <main> <article> <form> <dialog>`
- Every input gets a `<label>` — mobile keyboards and screen readers both need it
- `type="email"` and `type="date"` so phones show the right keyboard and picker
- No fixed pixel widths anywhere; `max-width` on the container does all the work

---

## 9 The CORS + cookie trap

Read this in Phase 7 while the errors are on screen. It is the single most
frustrating part of the whole roadmap, and it's frustrating because the browser
fails *silently* — no error, the cookie just isn't there.

In development, `localhost:5173` → `localhost:3000` is same-site and everything
works by accident. In production, `yourapp.vercel.app` → `yourapi.onrender.com`
are **different sites**, and every one of these must be true at once:

| # | Requirement | Where | Symptom if wrong |
| - | ----------- | ----- | ---------------- |
| 1 | `credentials: "include"` on every fetch | `api/client.ts` | Cookie never sent; `/me` always 401 |
| 2 | `cors({ origin: CLIENT_ORIGIN, credentials: true })` | backend | Browser blocks the response |
| 3 | Origin is an **exact** string, not `*` | backend | `*` is illegal with credentials — hard fail |
| 4 | `sameSite: "none"` on the cookie | backend | Cookie silently dropped cross-site |
| 5 | `secure: true` alongside it | backend | `sameSite:none` without `secure` is rejected |
| 6 | Both sides on HTTPS | Vercel + Render | `secure` cookies don't travel over http |
| 7 | No trailing slash in `CLIENT_ORIGIN` | env var | Exact-match fails; looks like #2 |

Debug it in this order, in DevTools:

1. **Network → the request → Request Headers.** Is `Cookie:` present? No → problem is #1 or #4.
2. **Network → the login response → Response Headers.** Is `Set-Cookie:` there? Present but the cookie isn't stored → #4, #5, or #6.
3. **Console.** An explicit CORS message → #2, #3, or #7.
4. **Application → Cookies.** If it's stored under the wrong domain, check `path` and that you never set `domain`.

Set `sameSite` and `secure` from `NODE_ENV` so dev and prod each get what they need:

```ts
const isProd = process.env.NODE_ENV === "production";
res.cookie("token", jwt, {
  httpOnly: true,
  secure: isProd,
  sameSite: isProd ? "none" : "lax",
  maxAge: 7 * 24 * 60 * 60 * 1000,
});
```

---

## 10 Backend files

```
server/
├── prisma/
│   ├── schema.prisma
│   └── migrations/20260812115626_init/migration.sql
├── prisma.config.ts                loads dotenv; Prisma 7 needs this
└── src/
    ├── index.ts                    listen on process.env.PORT
    ├── app.ts                      express(), cors, json, cookieParser, routes, errorHandler
    ├── db.ts                       one PrismaClient, exported
    ├── lib/
    │   ├── validate.ts             the §4 rules, one function per payload
    │   └── tokens.ts               sign / verify / cookie options
    ├── middleware/
    │   ├── requireAuth.ts          cookie → verify → req.userId, else 401
    │   └── errorHandler.ts         last app.use — the only place that formats errors
    └── routes/
        ├── health.ts
        ├── auth.ts                 register, login, logout, me
        └── posts.ts                list, create, remove
```

`db.ts` exports a **single** `PrismaClient`. Constructing one per request opens a
new connection pool per request and exhausts Neon's connection limit fast.

`errorHandler` is registered last and is the only code that writes an error
response. Routes throw; the handler formats. In production it logs the stack and
sends `{ "error": "Internal server error" }` — never the stack itself.

---

## 11 Environment

| Variable | Where | Example |
| -------- | ----- | ------- |
| `DATABASE_URL` | server | `postgresql://…-pooler…neon.tech/neondb?sslmode=require&channel_binding=require` |
| `JWT_SECRET` | server | 32+ random bytes. Different in dev and prod. |
| `CLIENT_ORIGIN` | server | `http://localhost:5173` → `https://yourapp.vercel.app` |
| `PORT` | server | `3000` locally; **Render sets it** — always read it, never hardcode |
| `NODE_ENV` | server | `production` on Render |
| `VITE_API_URL` | client | `http://localhost:3000` → `https://yourapi.onrender.com` |

Only `VITE_`-prefixed variables reach the browser in Vite, and everything that
reaches the browser is **public**. Never prefix a secret with `VITE_`.

`.env` is gitignored in both packages. Commit a `.env.example` with the keys and
empty values so the next person knows what's required.

---

## 12 Build order

A checklist. Nothing here needs the thing below it to exist.

**Phase 2 — database** ✅
- [x] Neon project, connection string in `.env`
- [x] `schema.prisma` with `User` and `Post`
- [x] `npx prisma migrate dev --name init`
- [x] Rows created by hand in Studio; foreign key verified by trying a bad `authorId`
- [x] `prisma/migrations/` committed, `.env` not

**Phase 3 — API, no auth**
- [ ] `db.ts`, `app.ts`, `index.ts`, `GET /api/health`
- [ ] `GET /api/posts` — real rows, newest first, author `id`+`name` only
- [ ] `POST /api/posts` — **hardcoded `authorId`**, validation, 400 on bad input
- [ ] `DELETE /api/posts/:id` — no ownership check yet
- [ ] `errorHandler`, consistent error shape
- [ ] Every endpoint exercised in a REST client, happy path and bad input

**Phase 4 — auth**
- [ ] `bcrypt`, `jsonwebtoken`, `cookie-parser` installed
- [ ] `register` (409 on duplicate email) · `login` (identical 401s) · `logout` · `me`
- [ ] `requireAuth` sets `req.userId`
- [ ] Replace the hardcoded `authorId` with `req.userId`
- [ ] `DELETE` becomes `deleteMany({ id, authorId })`

**Phase 5 — frontend**
- [ ] Vite scaffold, `react-router-dom`, `api/client.ts`
- [ ] `AuthContext` **with the `loading` flag**, `ProtectedRoute`
- [ ] `Navbar`, `SignUpModal`, `Home` + composer, `PostCard`, `Profile`
- [ ] `styles.css` and the 640px media query
- [ ] Guest check in a private window: feed visible, no composer, no delete buttons

---

## 13 Decisions

Made without asking. Push back on any of them.

| # | Decision | Reasoning |
| - | -------- | --------- |
| 1 | **Post is body-only** — no title | Reddit has titles; a first backend doesn't need two text fields to teach the same thing |
| 2 | **Own profile only** | Other-user profiles mean a `/users/:id` route and a public-fields decision. Out of scope. |
| 3 | **UUID primary keys** | Safe in URLs; sequential ints leak how many users you have |
| 4 | **Feed capped at 50, no pagination UI** | Pagination is a real topic and deserves its own project |
| 5 | **`VarChar(500)` on body, `VarChar(50)` on name** | The database is the last line of defence. Enforce limits where they can't be bypassed. |
| 6 | **`dateOfBirth` is `DATE`, `createdAt` is `TIMESTAMP`** | A birthday has no timezone; a signup does |
| 7 | **`ON DELETE CASCADE`** | Delete an account, the posts go too. The alternative is orphan rows nobody cleans up. |
| 8 | **JWT in an httpOnly cookie, not localStorage** | localStorage is readable by any script on the page — one XSS and the token is gone |
| 9 | **No refresh tokens; 7-day expiry** | Refresh-token rotation is a whole subject. Log in again after a week. |
| 10 | **Same 401 for unknown email and wrong password** | Different errors enumerate registered accounts |
| 11 | **Duplicate email is 409, not 400** | It's a state conflict, not malformed input, and the frontend shows a different message |
| 12 | **`deleteMany` for ownership** | Atomic, one round trip, no check-then-act race |
| 13 | **404 for another user's post, not 403** | 403 confirms the post exists. 404 tells them nothing. |
| 14 | **Sign-in in the navbar, sign-up in a `<dialog>`** | Your request. Nice side effect: two routes total. |
| 15 | **Native `<dialog>`, not a modal library** | Backdrop, `Esc`, and focus handling for free |
| 16 | **`confirmPassword` never leaves the browser** | The server can't use it; sending it is one more password in one more log |
| 17 | **Age ≥ 13** | You're collecting a birth date; do something with it. Also keeps you out of COPPA territory. |
| 18 | **One `fetch` chokepoint** | Phase 7 changes the base URL and credentials for every call at once |
| 19 | **Plain CSS, one file, no framework** | Tailwind would be faster and would teach you Tailwind |
| 20 | **No hamburger menu** | `flex-wrap` and one media query already handle a phone |
| 21 | **Neon, not Render Postgres** | Render's free database is deleted after 30 days |
| 22 | **No tests until Phase 8** | Testing a design that's still moving teaches you to hate tests |
