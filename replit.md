# CampusLink

Native Kotlin Android campus management for students and administrators, backed by Firebase and Room offline storage.

## Run & Operate

- `pnpm --filter @workspace/api-server run dev` — run the API server (port 5000)
- `pnpm run typecheck` — full typecheck across all packages
- `pnpm run build` — typecheck + build all packages
- `pnpm --filter @workspace/api-spec run codegen` — regenerate API hooks and Zod schemas from the OpenAPI spec
- `pnpm --filter @workspace/db run push` — push DB schema changes (dev only)
- Required env: `DATABASE_URL` — Postgres connection string
- Android app source and Firebase setup: `student-campus-android/README.md`
- Native Android builds require Android Studio, JDK 17, and Android SDK 35; this workspace has no Kotlin mobile preview.

## Stack

- pnpm workspaces, Node.js 24, TypeScript 5.9
- API: Express 5
- DB: PostgreSQL + Drizzle ORM
- Validation: Zod (`zod/v4`), `drizzle-zod`
- API codegen: Orval (from OpenAPI spec)
- Build: esbuild (CJS bundle)

## Where things live

- `student-campus-android/app` — Kotlin, Jetpack Compose, Room, Firebase client
- `student-campus-android/firebase` — Firestore/Storage rules, indexes, Cloud Functions, and emulator seed script
- `artifacts/api-server` — shared Express API service
- `artifacts/mockup-sandbox` — component preview and design canvas

## Architecture decisions

- Admin access is based on the Firebase Auth `admin` custom claim; a Firestore role field alone never grants administrator access.
- Payment writes go through callable functions so fee balances and payment history update atomically.
- Student-scoped records use explicit UID fields or recipient lists so Firestore queries and security rules match.

## Product

Students can access their own courses, attendance, timetable, assignment submissions, announcements, fees, payment history, results, profile, and notifications. Administrators can manage student accounts and academic records. Room and WorkManager support offline viewing and deferred writes.

## User preferences

_Populate as you build — explicit user instructions worth remembering across sessions._

## Gotchas

_Populate as you build — sharp edges, "always run X before Y" rules._

## Pointers

- See the `pnpm-workspace` skill for workspace structure, TypeScript setup, and package details
