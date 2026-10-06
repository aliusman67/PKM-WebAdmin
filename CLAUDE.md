# CLAUDE.md — Monitoring Ujian SMPN 5 Tangerang

Sistem monitoring ujian (ANBK-style): dashboard admin/pengawas web + aplikasi mobile khusus siswa (pengawas tidak punya aplikasi mobile). Real-time, anti-kecurangan (rate limit, anti-replay), dan tahan jaringan tidak stabil (offline-first mobile).

## Tech stack

### Web & API — Next.js 15 (App Router)
- **React 19** + **Next.js 15** (App Router) — frontend web dan backend API (Route Handlers) dalam satu aplikasi
- **TypeScript 5.x** strict mode + **Node.js 22** (LTS)
- **Styling & UI**: Tailwind CSS 4, shadcn/ui (Radix UI), `lucide-react` icons
- **Form & validasi**: React Hook Form + **Zod** — schema Zod dipakai bersama antara client dan Route Handler (folder `src/lib/schemas/`)
- **State**: TanStack Query v5 (server state/caching), Zustand (client state — sesi ujian, filter dashboard)
- **Real-time**: **SSE** (Server-Sent Events) via Route Handler + Redis pub/sub untuk update status jawaban siswa live; fallback polling 5 detik
- **Auth**: JWT (library `jose`) + session di Redis (httpOnly cookie untuk web, Bearer untuk mobile), RBAC: `ADMIN`, `PENGGAWA`, `PENGAWAS`, `SISWA` via middleware `src/middleware.ts`
- **Komponen khusus**: TanStack Table (daftar siswa), Recharts (grafik rekap nilai), `date-fns` + locale `id`
- **HTTP klien**: `fetch` bawaan dengan wrapper `src/lib/api-client.ts` (retry, signature HMAC)

### Database & Backend Services
- **PostgreSQL 16** — basis data utama
- **Prisma 6** — ORM (`prisma/schema.prisma`, migrasi di `prisma/migrations/`)
- **Redis 7** — cache, rate limit, anti-replay (nonce), pub/sub SSE, antrian job (opsional; fallback in-memory bila `REDIS_URL` kosong)
- **BullMQ** — background job (rekap nilai, generate PDF laporan, kirim notifikasi ke dashboard web — bukan push)
- **Penyimpanan file**: MinIO / S3-compatible untuk bukti pelanggaran & export berkas jawaban
- **Logging**: `pino` (structured JSON)

### Mobile — Flutter (proyek terpisah di `mobile/`)
- **Flutter 3.x + Dart 3.x** — aplikasi **siswa saja** (**Android-only**, bukan iOS); pemantauan pengawas cukup lewat dashboard web, tidak ada aplikasi mobile pengawas. Konsumsi API via JWT Bearer + header signature HMAC
- **State management**: Riverpod v2 (`flutter_riverpod` + `riverpod_generator`)
- **Navigasi**: `go_router`
- **Networking**: `dio` + interceptor (auth refresh, HMAC signature, retry exponential backoff)
- **Model & serialisasi**: `freezed` + `json_serializable` (codegen `build_runner`)
- **Offline-first** (krusial — jaringan sekolah tidak stabil): `drift` (SQLite) sebagai local store + job queue sinkronisasi; auto-sync saat koneksi pulih, conflict resolution server-wins
- **Keamanan**: `flutter_secure_storage` (token JWT), `device_info_plus` (device binding), deteksi root/jailbreak untuk siswa
- **Notifikasi**: tanpa push notification — Firebase/FCM **tidak dipakai** di mobile. Alert & pemantauan live pengawas hanya di dashboard web (SSE + fallback polling)
- **Konfigurasi**: `--dart-define` untuk base URL & env (dev/staging/prod)
- **Distribusi**: dirilis ke **Google Play Store** (satu aplikasi saja: siswa)
  - Application ID: `id.smpn5tangerang.ujian_siswa` — tanpa product flavor
  - Signing: Play App Signing + keystore release (diimpan aman, JKS di luar repo, via secret CI)
  - Build: `flutter build appbundle --release` (AAB, wajib untuk Play); minSdk 21, targetSdk sesuai kebijakan Play terbaru
  - Rilis via **GitHub Actions + fastlane**: track internal → closed testing (perlu ≥12 tester aktif 14 hari untuk persetujuan akses Play) → production
  - Kepatuhan Play: permission minimal — hanya `INTERNET` (tanpa `POST_NOTIFICATIONS`/lokasi/kamera kecuali fitur proktor), privacy policy & data safety form wajib diisi, akun sekolah (G Suite) untuk Play Console

### Infrastruktur & DevOps
- **Docker + docker-compose** — Postgres, Redis, MinIO, app (deployment on-premise di server sekolah)
- **Nginx** — reverse proxy + SSL (Let's Encrypt)
- **CI/CD**: GitHub Actions (lint, test, build web & Flutter)
- **Error tracking**: Sentry (web + mobile)
- **Testing**:
  - Web: Vitest + Testing Library (unit), Playwright (E2E)
  - Mobile: `flutter_test` (unit/widget), `integration_test` (E2E)

## Konvensi
- Bahasa kode: identifier Inggris, komentar & UI dalam Bahasa Indonesia
- Semua endpoint mobile di bawah `/api/v1/`, wajib header `X-Signature` (HMAC-SHA256) + `X-Nonce` + `X-Timestamp` (tolak jika selisih waktu > 30 detik)
- Migrasi Prisma tidak boleh diedit manual setelah di-apply; gunakan `npx prisma migrate dev --name <nama>`
