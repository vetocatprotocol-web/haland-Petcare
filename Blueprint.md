HALAND PETCARE v2.1

SINGLE SOURCE OF TRUTH (SSOT)

Version: 2.1

Status: Production Architecture

Architecture: Local-First Clinic Operating System

Last Updated: 2026-06-23

---

1. RINGKASAN & TUJUAN

Haland PetCare adalah Clinic Operating System untuk klinik hewan berbasis arsitektur Local-First. Dokumen ini adalah Single Source of Truth (SSOT) untuk seluruh pengembangan: frontend, local database, backend/replication, infra, CI/CD, testing, keamanan, dan prosedur produksi.

Tujuan utama:
- Menjamin operasional klinik tetap berjalan tanpa internet.
- Menetapkan aturan teknis dan operasional yang tak dapat ditawar (non-negotiable).
- Menyediakan panduan lengkap agar produk siap produksi dan dapat dipelihara.

---

2. SKOP DOKUMEN (AUDIENCE)

- Pengembang frontend/back-end/infra
- QA dan tim SRE
- Product owner dan stakeholder teknis
- Tim operasi & klinik untuk runbooks

---

3. ATURAN NON-NEGOTIABLE (RINGKAS)

- Local database is Source of Truth: semua operasi CRUD ditulis ke database lokal terlebih dahulu.
- Cloud hanya sebagai replication, backup, monitoring, dan remote access.
- Offline-first: semua fitur utama harus berfungsi penuh tanpa internet.
- Audit wajib untuk setiap tindakan penting.
- Soft-delete untuk data operasional (tidak ada hard delete).
- Security-first: autentikasi, otorisasi, RLS, enkripsi, dan audit.

---

4. ARSITEKTUR SISTEM (TINGKAT TINGGI)

App (PWA) — Local App Core (Next.js PWA + React) — Local DB (PGlite) — Sync Queue — Replication Engine — Cloud DB (Supabase PostgreSQL)

- Frontend: Next.js PWA (SSR/ISR sesuai kebutuhan) dengan service worker untuk offline.
- Local DB: PGlite sebagai primary runtime DB di perangkat klinik.
- ORM & Models: Drizzle ORM, schema di-sync antara local & cloud.
- Sync Engine: antrean (sync_queue) --> worker background --> Supabase.
- Cloud: Supabase PostgreSQL sebagai replica / reporting / multi-device sync hub.
HALAND PETCARE v2.1

SINGLE SOURCE OF TRUTH (SSOT)

Version: 2.1

Status: Production Architecture

Architecture: Local-First Clinic Operating System

Last Updated: 2026-06-23

---

1. RINGKASAN & TUJUAN

Haland PetCare adalah Clinic Operating System untuk klinik hewan berbasis arsitektur Local-First. Dokumen ini adalah Single Source of Truth (SSOT) untuk seluruh pengembangan: frontend, local database, backend/replication, infra, CI/CD, testing, keamanan, dan prosedur produksi.

Tujuan utama:
- Menjamin operasional klinik tetap berjalan tanpa internet.
- Menetapkan aturan teknis dan operasional yang tak dapat ditawar (non-negotiable).
- Menyediakan panduan lengkap agar produk siap produksi dan dapat dipelihara.

---

2. SKOP DOKUMEN (AUDIENCE)

- Pengembang frontend/back-end/infra
- QA dan tim SRE
- Product owner dan stakeholder teknis
- Tim operasi & klinik untuk runbooks

---

3. ATURAN NON-NEGOTABLE (RINGKAS)

- Local database is Source of Truth: semua operasi CRUD ditulis ke database lokal terlebih dahulu.
- Cloud hanya sebagai replication, backup, monitoring, dan remote access.
- Offline-first: semua fitur utama harus berfungsi penuh tanpa internet.
- Audit wajib untuk setiap tindakan penting.
- Soft-delete untuk data operasional (tidak ada hard delete).
- Security-first: autentikasi, otorisasi, RLS, enkripsi, dan audit.

---

4. ARSITEKTUR SISTEM (TINGKAT TINGGI)

App (PWA) — Local App Core (Next.js PWA + React) — Local DB (PGlite) — Sync Queue — Replication Engine — Cloud DB (Supabase PostgreSQL)

- Frontend: Next.js PWA (SSR/ISR sesuai kebutuhan) dengan service worker untuk offline.
- Local DB: PGlite sebagai primary runtime DB di perangkat klinik.
- ORM & Models: Drizzle ORM, schema di-sync antara local & cloud.
- Sync Engine: antrean (sync_queue) --> worker background --> Supabase.
- Cloud: Supabase PostgreSQL sebagai replica / reporting / multi-device sync hub.

Diagram sederhana:

Next.js PWA
  │
  ▼
App Core (UI + Sync) <-> PGlite (haland_local)
  │
  ▼
Sync Queue -> Replication Worker -> Supabase (replica)

---

5. TEKNOLOGI UTAMA (REKOMENDASI & VERSI)

- Frontend: Next.js 15, React 18+, TypeScript 5+
- Styling: Tailwind CSS, shadcn/ui
- State: Zustand
- Validation: Zod
- Local DB: PGlite (sqlite embedded optimized)
- ORM: Drizzle ORM
- Cloud DB: Supabase PostgreSQL
- Offline: PWA, Service Worker, IndexedDB (fallback), Background Sync
- CI/CD: GitHub Actions (atau equivalent), Vercel untuk frontend
- Observability: Sentry (errors), Prometheus/Grafana (metrics), Loki/ELK (logs)

---

6. LINGKUNGAN (ENVIRONMENTS)

- Development: lokal device, local PGlite, mock supabase
- Staging: cloud replica di Supabase, preview deployment di Vercel
- Production: Supabase utama untuk replica & analytics, Vercel untuk frontend

Variabel lingkungan harus dipisah: `.env.development`, `.env.staging`, `.env.production`.

---

7. PANDUAN DATA & SKEMA

Prinsip:
- Primary key: gunakan UUID v4 untuk entitas utama (kecuali index internal)
- Timestamps: `created_at`, `updated_at`, mandatory
- Soft delete: `deleted_at` nullable untuk operational tables
- Versioning: `entity_version` (incremental) untuk sync/conflict detection
- Audit: simpan `old_data` & `new_data` ter-serialized di `audit_logs`
- Index: index pada field pencarian umum (customer.name, pet.code, queue.status)

Konvensi penamaan:
- table: snake_case (plural)
- fields: snake_case

Migrasi:
- Gunakan Drizzle migrations untuk skema cloud
- Sediakan migrasi transform untuk local PGlite saat upgrade versi schema

---

8. SYNC ENGINE & RESOLUSI KONFLIK

Flow:
1) Semua perubahan dicatat di `sync_queue` (operation: INSERT/UPDATE/DELETE) bersama payload dan `entity_version`.
2) Background worker memproses antrean dan mengirimkan perubahan ke Supabase dengan idempotency key dan checksum.
3) Jika konflik terdeteksi, jangan pakai Last-Write-Wins otomatis — buat `conflict_queue` dan tandai `status=PENDING` untuk resolusi manual/otomatis.

Conflict resolution strategy:
- Merge-first: jika per-field otomatis bisa digabung, lakukan merge (non-destructive).
- Jika merge tidak memungkinkan, buat entri konflik dengan `local_version`, `server_version`, dan metadata; tampilkan UI resolusi ke admin.

Retry & backoff: gunakan exponential backoff, limit retry_count, dan alert ketika antrean terlalu lama.

---

9. API CONTRACT & INTEGRATION

- API berbasis REST/HTTP JSON untuk replication dan integrasi eksternal.
- Semua endpoint harus idempotent untuk operasi yang bisa di-retry.
- Gunakan request/response schema yang tervalidasi (Zod) dan dokumentasikan di `docs/api-contract.md`.
- Versi API: `v1`, pertimbangkan version header.

Contoh endpoints (ringkas):
- `POST /sync/replicate` — kirim batch perubahan
- `GET /devices/:id/sync-status`
- `POST /auth/replicate-login` — untuk rekhabilitasi user jika perlu

---

10. AUTENTIKASI & SESSIONS OFFLINE

- Tabel: `local_users`, `local_sessions`.
- Flow: login -> local lookup -> password verify (PBKDF2/argon2) -> create local session -> access.
- Session expiry: adjustable, device-bound tokens.
- Saat replikasi ke cloud, jangan kirim password plain — kirim hash atau gunakan token re-auth flow.

---

11. FRONTEND - PWA & OFFLINE BEHAVIOR

Requirements:
- PWA installable, offline shell cached by service worker.
- UI patterns: skeleton loading, optimistic updates, sync status indicators, conflict banners.
- Background sync: upload sync_queue when online.
- Incremental hydration: prioritas ke critical UI (queue, POS, medical-record).

Accessibility & responsivity harus dipenuhi.

---

12. BACKEND & WORKERS

- Replication worker harus idempotent, menggunakan transactions ketika mengubah stock/inventory.
- Semua pembayaran & stock reduction hanya commit setelah payment success.
- Worker memproses sync_queue dan menulis ke Supabase, mengembalikan hasil (success/error) ke perangkat.

---

13. KEAMANAN (SECURITY)

- Principle: DENY ALL, allow specific access.
- RLS (Row Level Security) di Supabase: aktifkan dan buat role mapping.
- Enkripsi at-rest (cloud DB) dan in-transit (TLS).
- Secrets management: gunakan Secret Manager (Vercel/Supabase/Cloud provider).
- Audit trails: setiap aksi penting dicatat di `audit_logs`.

---

14. OBSERVABILITY & MONITORING

- Errors: Sentry
- Metrics: Prometheus/Grafana (request latency, sync queue length)
- Logs: structured logs (Loki/ELK)
- Alerts: threshold pada pending sync, failed payments, queue growth

---

15. BACKUP & DISASTER RECOVERY

- Backup schedule: daily snapshot + weekly full backup
- Retention: minimum 30 days (configurable)
- Backup targets: local file (on-site), Supabase Storage (cloud)
- Restore runbook: langkah-langkah untuk restore local dan cloud (uji recovery secara berkala)

---

16. CI/CD, RELEASE & ROLLBACK

Branching:
- `main`: production-ready
- `develop` / feature branches: development
- PR required, 1+ reviewer, must pass CI

Pipeline (ringkas):
1) Lint & typecheck
2) Unit tests
3) Integration tests (mocks)
4) Build
5) E2E tests (staging preview)
6) Deploy to staging
7) Canary/gradual deploy to production (if applicable)

Rollback: siap dengan deploy tag dan database migration rollback scripts.

---

17. TESTING MATRIX

- Unit tests: logic (models, utils)
- Integration tests: API contracts, DB migrations
- E2E: critical flows (login offline, POS payment, sync)
- Offline tests: simulate offline device and sync convergence
- Load tests: peak clinic load (concurrent queue, POS)

Automate tests in CI and require passing before merge.

---

18. PERFORMANCE TARGETS (SAMA)

- Dashboard: < 2s
- Customer Search: < 100ms
- Queue Load: < 300 ms
- Medical Record Save: < 500 ms
- POS Payment: < 500 ms
- Background Sync: non-blocking, prioritize user flow

---

19. DEFINITION OF DONE (DITERAPKAN)

Feature dianggap selesai hanya jika memenuhi semua item berikut:
- Validation
- Loading State
- Empty State
- Error State
- Audit Logged
- Offline Supported
- Sync Supported
- Role Protected
- RLS Protected
- Responsive
- Tested Offline
- Tested Sync
- No Console Error
- No Data Loss
- Performance Targets Met
- Documented (docs/ atau perubahan pada Blueprint)

---

20. ONBOARDING PENGEMBANG (LANGKAH CEPAT)

Clone & instalasi:

```bash
git clone <repo>
cd haland-Petcare
pnpm install # or npm/yarn sesuai proyek
```

Menjalankan dev (frontend + local DB):

```bash
pnpm dev # menjalankan Next.js PWA
# jalankan migration/seed untuk local PGlite (script ada di package.json)
pnpm run migrate:local
pnpm run seed:local
```

Testing:

```bash
pnpm test
pnpm test:e2e --config=playwright.config.ts
```

---

21. RUNBOOKS & INCIDENT RESPONSE (RINGKAS)

Prioritas insiden:
- P0: POS down / Data loss
- P1: Sync queue berhenti / retry failure
- P2: Slow queries / high latency

Contoh tindakan cepat:
- Periksa `sync_queue` length dan last_attempt_at
- Restart replication worker
- Restore dari backup jika data corrupt
- Notify on-call + buat incident ticket

---

22. DOKUMENTASI & PEMELIHARAAN

- Semua perubahan arsitektural wajib update dokumen ini terlebih dahulu.
- Tempat dokumentasi: [Blueprint.md](Blueprint.md), docs/database-schema.md, docs/api-contract.md, docs/sync-engine.md, docs/rls-policies.md
- Buat PR untuk setiap pembaruan dan sertakan changelog singkat.

---

23. CHANGE CONTROL

- Perubahan pada SSOT harus:
  1) Diusulkan lewat issue + RFC singkat
  2) Direview oleh 2 orang (architect + maintainer)
  3) Di-approve lalu di-merge dengan PR yang menyertakan update dokumentasi

---

24. NEXT STEPS & REKOMENDASI

- Review dokumen ini bersama tim teknis untuk menyepakati point-point utama.
- Lengkapi `docs/` dengan skema database lengkap, API contract terperinci, dan runbook restore.
- Implementasikan CI checks untuk menegakkan Definition of Done.

Dokumen ini kini menjadi versi SSOT awal — beri tahu saya jika mau saya format ulang ke `docs/BLUEPRINT.md`, tambahkan tabel skema DB terperinci, atau buat PR otomatis.
