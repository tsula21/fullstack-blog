# Roadmap

The order matters more than anything else here:

> **database → API → auth → frontend → deploy → CI → polish**

Each layer is fully testable *before the next one exists*. You never debug two
unknowns at once. If the API is broken you find out with a REST client, not by
staring at a blank React page wondering which of the four layers lied to you.

| #  | Phase | Time | Done when |
| -- | ----- | ---- | --------- |
| 0  | Setup — Node, VS Code, GitHub account, repo | ½ day | `node -v` works, repo pushes |
| 1  | Design on paper — data model + endpoint list | ½ day | [DESIGN.md](./DESIGN.md) exists |
| 2  | Database — Neon, Prisma schema, first migration | 2–3 days | Migration applied, rows visible in Studio |
| 3  | Backend API — Express CRUD, no frontend | 1 week | All 8 endpoints pass by hand in a REST client |
| 4  | Auth — register/login/logout, hashing, sessions | 1 week | Cookie round-trips, protected route rejects guests |
| 5  | Frontend — React + Vite against your own API | 1 week | Both pages work, ugly is fine |
| 6  | Deploy backend — Render, env vars, public URL | 2–3 days | `curl https://…/api/health` returns 200 |
| 7  | Deploy frontend — Vercel, then fight CORS/cookies | 1–2 days | Login works from the deployed site |
| 8  | CI/CD — GitHub Actions on every push | 1–2 days | Green check on a PR |
| 9  | Polish — logging, errors, rate limiting, README | 2–3 days | You'd show it to someone |
| 10 | Teardown — delete everything, keep the repo | 1 hour | $0 bill, code preserved |

Total: roughly six weeks of evenings. Slipping is normal; skipping is not.

---

## Phase 0 — Setup · ½ day

Install Node LTS, VS Code, and Git. Create the GitHub account and the repo, push
an empty commit.

GitHub is **not optional** for this project. Vercel and Render both deploy *from a
GitHub repo*, and Actions runs on push. Without it there is no deployment story at all.

**Done when:** `node -v`, `npm -v`, `git --version` all print versions, and an
empty commit is visible on GitHub.

---

## Phase 1 — Design on paper · ½ day

Two questions, answered before any code:

1. What tables exist, and what columns does each one have?
2. What is the exact list of HTTP endpoints, with method, path, request body, and response?

Both answers live in [DESIGN.md](./DESIGN.md). Two tables, eight endpoints.

**Done when:** you can name every column and every endpoint without looking.

---

## Phase 2 — Database first · 2–3 days ✅

The database is the foundation. Everything above it is a translation layer over
what you decide here, so it goes first.

1. Create a free Neon project. Copy the pooled connection string into `server/.env`.
2. `npm i -D prisma && npm i @prisma/client`, then write `prisma/schema.prisma`.
3. `npx prisma migrate dev --name init` — this both writes the SQL file *and*
   applies it.
4. `npx prisma studio` — add a user and a post by hand. Then try creating a post
   with an `authorId` that doesn't exist and watch the foreign key reject it.
5. **Read the generated SQL.** It is the only place you see what Prisma actually
   did. Walkthrough in [NOTES.md](./NOTES.md#reading-the-init-migration).

Use **Neon, not Render's free Postgres** — a Render free database is deleted 30
days after creation, which is exactly the wrong lifespan for a six-week project.

**Done when:** `prisma/migrations/` is committed, `.env` is not, and Studio shows
your hand-made rows.

---

## Phase 3 — Backend API · 1 week

Express + Prisma. No frontend, no auth. You test it entirely with a REST client
(Thunder Client, Bruno, Postman, or a `.http` file in VS Code).

Build, in this order:

1. `GET /api/health` — returns `{ ok: true }`. Proves the server boots.
2. `GET /api/posts` — reads real rows, newest first.
3. `POST /api/posts` — **with a hardcoded `authorId`** pasted from Studio.
4. `DELETE /api/posts/:id` — deletes anything, ownership comes in Phase 4.

That hardcoded `authorId` is not a shortcut you should feel bad about. Auth does
not exist yet. Phase 4 replaces the constant with `req.userId` and nothing else
about this code changes.

Also this phase: input validation, a JSON error shape, and a single
`errorHandler` middleware so no route ever leaks a stack trace.

**Done when:** every endpoint in [DESIGN.md §5](./DESIGN.md#5-api-contracts)
returns the documented status code for both the happy path and the bad input.

---

## Phase 4 — Auth · 1 week

The most valuable week in the roadmap. Auth is the best teacher of backend
concepts there is: hashing, statelessness, middleware, cookies, expiry, and the
difference between *who you claim to be* and *who the server says you are*.

Write it yourself. Do not reach for Auth0 or Clerk here.

1. `POST /api/auth/register` — validate, `bcrypt.hash(password, 12)`, insert,
   set cookie. Handle the unique-email collision as a 409, not a 500.
2. `POST /api/auth/login` — look up by email, `bcrypt.compare`, set cookie.
   Return the **same** error for wrong email and wrong password.
3. `GET /api/auth/me` — reads the cookie, returns the user or 401.
4. `POST /api/auth/logout` — clears the cookie.
5. `requireAuth` middleware — verifies the JWT, sets `req.userId`, else 401.

Then go back and wire `req.userId` into the post routes.

**Two rules that prevent most auth bugs:**

- **Identity comes from the token, never the payload.** `authorId` is `req.userId`.
  If a client can send you an author id, a client can post as anyone.
- **The JWT lives in an httpOnly cookie**, so JavaScript can't read it and neither
  can an XSS payload. This is also the thing that will bite you in Phase 7 —
  see [DESIGN.md §9](./DESIGN.md#9-the-cors--cookie-trap).

**Done when:** register → me → logout → me returns 200 → 200 → 204 → 401, and
deleting someone else's post 404s.

---

## Phase 5 — Frontend · 1 week

React + Vite. Two routes, thirteen files, ~80 lines of CSS. Ugly is fine — the
spec is [DESIGN.md §6–8](./DESIGN.md#6-frontend-routes).

Order: `client.ts` → `AuthContext` → `Navbar` → `Home` → `SignUpModal` → `Profile`.

The one thing everybody gets wrong: **`AuthContext` needs a `loading` flag.** On a
hard refresh of `/profile`, `GET /api/auth/me` has not come back yet, so `user`
is `null`, so `ProtectedRoute` decides you're a guest and bounces you home. The
flag makes the route render nothing until the answer arrives.

**Done when:** you can sign up, refresh the page, still be signed in, post, delete
your post, and see it all as a guest in a private window.

---

## Phase 6 — Deploy backend · 2–3 days

Render, free web service, deploying from GitHub.

- Root directory `server`, build `npm install && npx prisma generate && npm run build`,
  start `npm run start`.
- Set `DATABASE_URL`, `JWT_SECRET`, `CLIENT_ORIGIN`, `NODE_ENV=production` in the
  Render dashboard. **Never** commit them.
- Bind to `process.env.PORT`. Render assigns the port; hardcoding 3000 fails.
- Migrations in production are `npx prisma migrate deploy`, never `migrate dev`.
  `dev` can drop your data. `deploy` only applies pending files.

Free Render services sleep after 15 minutes idle and take ~50s to wake. That is
the free tier, not a bug.

**Done when:** `curl https://your-api.onrender.com/api/health` returns `{"ok":true}`.

---

## Phase 7 — Deploy frontend · 1–2 days

Vercel, root directory `client`, one env var: `VITE_API_URL` pointing at Render.

Then you fight CORS and cookies. You will. Budget the day for it — the causes and
fixes are written up in advance in [DESIGN.md §9](./DESIGN.md#9-the-cors--cookie-trap)
so you can read them while the errors are fresh instead of guessing.

**Done when:** you can log in on the deployed site, refresh, and still be logged in.

---

## Phase 8 — CI/CD · 1–2 days

One workflow file, `.github/workflows/ci.yml`: on push and PR, install, typecheck
both packages, run tests, build.

It won't save you time on a solo project. Do it anyway — it takes two hours, and
"I've never set up CI" is a bad sentence in an interview.

**Done when:** a PR shows a green check, and a deliberately broken type shows a red one.

---

## Phase 9 — Polish · 2–3 days

- Request logging (`morgan` or ten lines of your own middleware)
- One error handler, consistent JSON error shape, no stack traces in production
- `express-rate-limit` on `/api/auth/*` — brute force is the obvious attack
- `helmet` for security headers
- A README that a stranger could follow to run the thing

**Done when:** you'd link it in an application.

---

## Phase 10 — Teardown · 1 hour

Delete the Render service, the Vercel project, and the Neon project. Keep the
repo — it is the artifact. Confirm every dashboard reads $0.

Write the last NOTES.md entry: what you'd do differently.

---

## Afterwards

- **Project #2 in Next.js.** Now that you know what a backend is, having the
  framework hide it is a feature instead of a lie.
- **A real VPS, if you want that experience.** Hetzner CX23 is €5.49/mo billed
  *hourly*, so a weekend of nginx, systemd, and certbot costs about €1. Do it
  once so "it's on a server" stops being an abstraction.
