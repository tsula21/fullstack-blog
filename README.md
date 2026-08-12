# fullstack-learning

A deliberately small Reddit-shaped blog, built one layer at a time to learn how a
real full-stack app fits together — database, API, auth, frontend, deployment, CI.

The app itself is trivial on purpose. Users sign up, post text, delete their own
posts. Guests can read everything. No likes, no comments, no styling ambitions.
**The point is the plumbing, not the product.**

## Stack

| Layer     | Choice                        | Why |
| --------- | ----------------------------- | --- |
| Language  | TypeScript everywhere         | One language, so the concepts are the only new thing |
| Backend   | Node + Express 5              | Explicit and unopinionated — you see every layer |
| ORM       | Prisma 7                      | Typed queries, real migration files you can read |
| Database  | PostgreSQL on Neon (free)     | Free tier that does not expire (Render's does, after 30 days) |
| Frontend  | React + Vite                  | Separate from the backend on purpose — see below |
| Auth      | bcrypt + JWT in httpOnly cookie | Written by hand; it is the best teacher of backend concepts |
| Hosting   | Vercel (web) + Render (API) + Neon (db) | $0 total |
| CI        | GitHub Actions                | Typecheck + tests on every push |

**Why React + Vite and not Next.js:** Next.js merges the frontend and backend into
one deployable. That is great for productivity and bad for understanding what a
backend actually *is*. Build it the hard way once; use Next.js for project #2.

## Documents

| File | What's in it |
| ---- | ------------ |
| [ROADMAP.md](./ROADMAP.md) | The 11 phases, in order, with time estimates and a definition of done for each |
| [DESIGN.md](./DESIGN.md)   | The spec — data model, all 8 API contracts, frontend routes and files, decisions log |
| [NOTES.md](./NOTES.md)     | Running learning log: things that broke, why, and what the fix actually was |

## Current status

- [x] **Phase 0** — Setup: Node, VS Code, GitHub, repo
- [x] **Phase 1** — Design on paper (see [DESIGN.md](./DESIGN.md))
- [x] **Phase 2** — Database: Neon project, Prisma schema, `20260812115626_init` migration applied
- [ ] **Phase 3** — Backend API ← *you are here*
- [ ] **Phase 4** — Auth
- [ ] **Phase 5** — Frontend
- [ ] **Phase 6** — Deploy backend
- [ ] **Phase 7** — Deploy frontend
- [ ] **Phase 8** — CI/CD
- [ ] **Phase 9** — Polish
- [ ] **Phase 10** — Teardown

## Repo layout

```
fullstack-learning/
├── README.md            this file
├── ROADMAP.md           the plan
├── DESIGN.md            the spec
├── NOTES.md             the learning log
├── server/              Node + Express + Prisma API
│   ├── prisma/
│   │   ├── schema.prisma
│   │   └── migrations/
│   │       └── 20260812115626_init/migration.sql
│   ├── prisma.config.ts
│   ├── tsconfig.json
│   └── src/             (Phase 3)
└── client/              React + Vite app (Phase 5 — empty for now)
```

## Running the backend

```bash
cd server
npm install
npx prisma generate       # regenerates the client into src/generated/prisma
npm run dev               # tsx watch src/index.ts
```

Inspect the database in a browser:

```bash
cd server
npx prisma studio         # http://localhost:5555
```

### Environment

`server/.env` is gitignored and must never be committed. It needs:

```ini
DATABASE_URL="postgresql://...neon.tech/neondb?sslmode=require&channel_binding=require"
JWT_SECRET="..."          # Phase 4 — any long random string
CLIENT_ORIGIN="http://localhost:5173"
PORT=3000
```

Prisma 7 does **not** read `.env` automatically. `prisma.config.ts` imports
`dotenv/config` to make it work — that is what that file is for.

> If `npx prisma` ever fails with **P1001: Can't reach database server**, the Neon
> compute is almost certainly just asleep, not broken. See [NOTES.md](./NOTES.md#p1001-on-neon).

## The rule that makes this worth doing

An AI can write this entire app in an afternoon. If you let it, you end up with a
deployed site and no new understanding.

**Write each piece badly yourself first, get it working, then ask for a review.**
The review is where the learning is. The generation is where it isn't.
