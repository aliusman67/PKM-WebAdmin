# CLAUDE.md — Monitoring Ujian

## Tech stack

### Web & API
- **React 19** + **Next.js 15** (App Router) — frontend web dan backend API (Route Handlers) dalam satu aplikasi
- **PostgreSQL 16** — basis data
- **Prisma 6** — ORM (`prisma/schema.prisma`, migrasi di `prisma/migrations/`)
- **Redis 7** — cache, rate limit, anti-replay, pub/sub SSE (opsional; fallback in-memory bila `REDIS_URL` kosong)
- **TypeScript** + Node.js 22

### Mobile
- **Flutter** — aplikasi siswa & pengawas (Android/iOS), proyek terpisah; konsumsi API via JWT Bearer + header signature HMAC
