# WebMin — Sistem Monitoring Ujian (ANBK-style)

Sistem monitoring ujian sekolah: **dashboard admin & pengawas (web)** + **aplikasi mobile Android khusus siswa**. Pengawas memantau lewat web, bukan aplikasi mobile. Real-time (SSE), anti-kecurangan (rate limit, anti-replay HMAC), dan tahan jaringan tidak stabil (mobile offline-first).

| Platform | Lokasi | Dokumentasi setup |
|---|---|---|
| Web + API (Next.js 15) — admin & pengawas | root repo (`src/`, `prisma/`) | dokumen ini |
| Mobile Android (Flutter) — siswa saja | `mobile/` | [`mobile/README.md`](mobile/README.md) |

Detail lengkap tech stack & konvensi ada di [`CLAUDE.md`](CLAUDE.md).

---

## 1. Prasyarat (Web)

- **Node.js 22 LTS** + npm (atau pnpm)
- **Docker + docker-compose** — untuk Postgres 16, Redis 7, MinIO
- PostgreSQL 16 & Redis 7 (bisa lewat Docker, atau instance sendiri)
- Git

## 2. Scaffold Next.js 15 (App Router)

> Web app menempati **root repo**. Jalankan dari root, lalu pindahkan hasil scaffold ke root (jangan buat folder `web/`).

```bash
npx create-next-app@15 . --typescript --tailwind --eslint --app --src-dir --turbopack --use-npm
```

## 3. Dependensi

```bash
# UI
npm i @radix-ui/react-slot class-variance-authority clsx tailwind-merge lucide-react
npx shadcn@latest init

# State & data fetching
npm i @tanstack/react-query @tanstack/react-table zustand

# Form & validasi (schema Zod dipakai bersama client <-> Route Handler)
npm i react-hook-form zod @hookform/resolvers

# Auth & keamanan
npm i jose

# Realtime & backend services
npm i ioredis bullmq

# Database
npm i @prisma/client
npm i -D prisma

# Utilitas
npm i date-fns pino pino-pretty
```

Dev dependencies testing:

```bash
npm i -D vitest @testing-library/react @testing-library/jest-dom jsdom \
  @playwright/test
```

## 4. Struktur folder

```
├── src/
│   ├── app/                    # App Router (halaman + Route Handler /api/v1/*)
│   ├── components/             # Komponen React (shadcn/ui di components/ui/)
│   ├── lib/
│   │   ├── schemas/            # Schema Zod bersama (client & API)
│   │   ├── api-client.ts       # Wrapper fetch (retry + signature HMAC)
│   │   ├── auth.ts             # JWT (jose) + session Redis
│   │   ├── redis.ts            # Koneksi Redis (fallback in-memory jika REDIS_URL kosong)
│   │   ├── rate-limit.ts       # Rate limit + anti-replay (nonce)
│   │   └── logger.ts           # pino
│   ├── hooks/
│   ├── stores/                 # Zustand (sesi ujian, filter dashboard)
│   └── middleware.ts           # RBAC: ADMIN, PENGGAWA, PENGAWAS, SISWA
├── prisma/
│   ├── schema.prisma
│   └── migrations/
├── docker-compose.yml
└── mobile/                     # Proyek Flutter — aplikasi siswa saja (lihat mobile/README.md)
```

## 5. Environment variables

Salin `.env.example` → `.env` (jangan commit `.env`):

```env
# Database & cache
DATABASE_URL="postgresql://webmin:webmin@localhost:5432/webmin?schema=public"
REDIS_URL="redis://localhost:6379"          # kosongkan untuk fallback in-memory

# Auth & keamanan
JWT_SECRET="<rahasia acak, min 32 char>"
HMAC_SECRET="<kunci signature X-Signature mobile>"

# Storage (bukti pelanggaran & export jawaban)
S3_ENDPOINT="http://localhost:9000"
S3_BUCKET="webmin-bukti"
S3_ACCESS_KEY="minioadmin"
S3_SECRET_KEY="minioadmin"

# Observability
SENTRY_DSN=""
LOG_LEVEL="debug"
```

## 6. Layanan lokal via Docker

`docker-compose.yml` (dev) — Postgres, Redis, MinIO:

```yaml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: webmin
      POSTGRES_PASSWORD: webmin
      POSTGRES_DB: webmin
    ports: ["5432:5432"]
    volumes: [pgdata:/var/lib/postgresql/data]
  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]
  minio:
    image: minio/minio
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    ports: ["9000:9000", "9001:9001"]
    volumes: [miniodata:/data]
volumes:
  pgdata:
  miniodata:
```

```bash
docker compose up -d
```

## 7. Prisma

```bash
npx prisma init --datasource-provider postgresql
npx prisma migrate dev --name init        # migrasi baru SELALU lewat CLI, jangan edit manual
npx prisma studio                          # inspect DB
```

Aturan (lihat CLAUDE.md): migrasi yang sudah di-apply **tidak boleh diedit manual**.

## 8. Skrip `package.json`

```json
{
  "scripts": {
    "dev": "next dev --turbopack",
    "build": "prisma generate && next build",
    "start": "next start",
    "lint": "next lint && tsc --noEmit",
    "test": "vitest run",
    "test:e2e": "playwright test",
    "db:migrate": "prisma migrate dev",
    "db:deploy": "prisma migrate deploy",
    "db:studio": "prisma studio"
  }
}
```

## 9. Jalankan

```bash
npm run dev        # http://localhost:3000
```

- Dashboard: `/dashboard` — dipakai **admin & pengawas** (pengawas tidak punya aplikasi mobile)
- API mobile: `/api/v1/*` — konsumen tunggalnya **aplikasi siswa**; **wajib** header `X-Signature` (HMAC-SHA256), `X-Nonce`, `X-Timestamp` (ditolak jika selisih waktu > 30 detik)
- SSE realtime: `/api/v1/stream` (Redis pub/sub, fallback polling 5 detik) — pemantauan live pengawas di dashboard

## 10. Testing

```bash
npm run test               # Vitest + Testing Library (unit)
npx playwright install     # sekali saja
npm run test:e2e           # Playwright (E2E)
```

## 11. Mobile (siswa) & rilis Play Store

Satu aplikasi Android saja — **`id.smpn5tangerang.ujian_siswa`** — tanpa product flavor dan tanpa Firebase/FCM. Setup Flutter, signing, dan alur rilis Google Play (internal → closed testing → production) dijelaskan di **[mobile/README.md](mobile/README.md)**.
