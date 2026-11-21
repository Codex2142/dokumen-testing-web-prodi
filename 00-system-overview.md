# 00 — System Overview & Database Preparation Report

**Aplikasi**: Janjian Dosen Informatika (SITU-Reservasi Dosen & Bimbingan Akademik)  
**Lingkungan Uji Aktif**: `https://38-9-47-207.sslip.io` (VPS `38.9.47.207` + Neon PostgreSQL `db_cadangan`)  
**Metode Penyusunan**: *Codebase-Driven & Evidence-Based QA* (`docs/skills/test_app.md`)  
**Tanggal Analisis**: 21 November 2026

---

## 1. Laporan Hasil Phase 1 — Database & Storage Preparation

### 1.1 Identifikasi Database

| Atribut | Nilai Aktual (`VERIFIED`) | Referensi |
| :--- | :--- | :--- |
| **Driver** | PostgreSQL (`github.com/jackc/pgx/v5/pgxpool`) | `backend/internal/infrastructure/database/postgres.go` |
| **Versi Server** | PostgreSQL 18.7 (aarch64-unknown-linux-gnu) | Verifikasi langsung via `psql` |
| **Host** | `ep-red-term-b3rlb6sd-pooler.c-4.ap-southeast-1.aws.neon.tech` | `.env` |
| **Database Name** | `db_cadangan` | `.env` |
| **Pool Config** | `DB_MAX_CONNS=25`, `DB_MIN_CONNS=5`, `search_path=public` | `.env`, `postgres.go` |
| **APP_ENV** | `development` (digunakan bersama lokal & VPS staging `38.9.47.207`) | `.env` |
| **Mekanisme Migrasi** | Eksekusi berurutan `*.up.sql` idempoten (tanpa tabel `schema_migrations`) | `backend/internal/infrastructure/database/migrate.go` |

### 1.2 Backup & Pembersihan Database (Disetujui User)

- **Backup Pra-Cleanup**: Berhasil dibuat & diverifikasi via `postgres:18-alpine` pada VPS:
  `/opt/reservasi-dosen/backups/postgres/neondb_reservasi_20261121_075144.sql.gz` (`40 KB`).
- **Storage VPS**: Direktori `/opt/reservasi-dosen/data/storage/uploads/` telah dikosongkan (`total 0`, izin `drwxrwxrwx`).
- **Kondisi 14 Tabel Pasca-Cleanup**:

| No | Nama Tabel | Kategori | Sebelum Cleanup | Sesudah Cleanup | Status |
| :---: | :--- | :--- | :---: | :---: | :--- |
| 1 | `users` | Master Pengguna | 9 | **9** | **Dipertahankan** |
| 2 | `system_settings` | Konfigurasi Sistem | 1 | **1** | **Dipertahankan** |
| 3 | `academic_calendar` | Jadwal Operasional | 5 | **0** | Dibersihkan |
| 4 | `teaching_schedules` | Jadwal Operasional | 3 | **0** | Dibersihkan |
| 5 | `lecturer_availabilities` | Jadwal Operasional | 4 | **0** | Dibersihkan |
| 6 | `lecturer_blocked_schedules` | Jadwal Operasional | 0 | **0** | Dibersihkan |
| 7 | `appointments` | Transaksi Inti | 21 | **0** | Dibersihkan |
| 8 | `appointment_reschedule_requests` | Transaksi Relasi | 1 | **0** | Dibersihkan |
| 9 | `feedbacks` | Transaksi Masukan | 1 | **0** | Dibersihkan |
| 10 | `feedback_attachments` | Lampiran Masukan | 2 | **0** | Dibersihkan |
| 11 | `notifications` | Log Notifikasi | 45 | **0** | Dibersihkan |
| 12 | `notification_outbox` | Antrean Outbox | 38 | **0** | Dibersihkan |
| 13 | `audit_logs` | Log Audit | 216 | **0** | Dibersihkan |
| 14 | `user_sessions` | Sesi Refresh Token | 156 | **0** | Dibersihkan |

### 1.3 Verifikasi Migrasi (16/16 Terverifikasi)

- **14 Tabel** dari `000002`–`000016` tersedia lengkap.
- **6 PostgreSQL ENUM** terverifikasi:
  - `user_role`: `ADMIN`, `DOSEN`, `MAHASISWA`
  - `appointment_status`: `MENUNGGU_PERSETUJUAN`, `DISETUJUI`, `DITOLAK`, `BENTROK`, `DIBATALKAN`, `SELESAI`
  - `reschedule_status`: `PENDING`, `ACCEPTED`, `REJECTED`
  - `day_name`: `SENIN`, `SELASA`, `RABU`, `KAMIS`, `JUMAT`, `SABTU`, `MINGGU`
  - `outbox_status`: `PENDING`, `PUBLISHED`, `FAILED`
  - `feedback_status`: `MENUNGGU_REVIEW`, `SEDANG_DITINJAU`, `SELESAI`, `DIARSIPKAN`
- **Exclusion Constraint**: `prevent_lecturer_double_booking` (`GIST` pada `appointments` untuk `status = 'DISETUJUI'`) aktif.
- **Trigger**: `trg_prevent_cancellation` (`BEFORE UPDATE OF status ON appointments`) aktif.

### 1.4 Akun Master Tersedia untuk QA (`users` = 9 record)

| Role | Identifier (NIM/NIP) | Nama | Email | Status |
| :--- | :--- | :--- | :--- | :---: |
| `ADMIN` | `198001012005011001` | Admin Prodi Informatika | `admin@telkomuniversity.ac.id` | Aktif |
| `DOSEN` | `197911122003121003` | Prof. Ir. Budi Raharjo, Ph.D. | `budi.raharjo@telkomuniversity.ac.id` | Aktif |
| `DOSEN` | `198503152010121002` | Dr. Eng. Hendra Kurniawan, S.T., M.T. | `hendra.kurniawan@telkomuniversity.ac.id` | Aktif |
| `DOSEN` | `198503152010121005` | Prof. Uji Coba 1 edited | `uji.coba@telkomuniversity.ac.id` | Aktif |
| `DOSEN` | `198807202014042001` | Dr. Siti Aminah, S.Kom., M.Cs. | `siti.aminah@telkomuniversity.ac.id` | Aktif |
| `MAHASISWA` | `1203220012` | Bima Pratama Putra | `bima.putra@student.telkomuniversity.ac.id` | Aktif |
| `MAHASISWA` | `1203220088` | Citra Dewi Lestari | `citra.lestari@student.telkomuniversity.ac.id` | Aktif |
| `MAHASISWA` | `1203220114` | Marcel Hans Ganteng | `marcelhans@student.telkomuniversity.ac.id` | Aktif |
| `MAHASISWA` | `1203220154` | Akbar Muhammad Sadat | `akbar.sadat@student.telkomuniversity.ac.id` | Aktif |

> **Konfigurasi `system_settings` (`appointment_rules`) saat ini**:  
> `min_minutes: 15`, `default_minutes: 30`, `max_minutes: 60`, `slot_interval_minutes: 15`, `buffer_minutes: 10`, `min_lead_days: 1`, `max_advance_days: 30`.

---

## 2. Arsitektur & Stack Teknologi (`VERIFIED`)

| Lapisan | Teknologi & Versi | Referensi File |
| :--- | :--- | :--- |
| **Backend API** | Go `1.27.1`, Gin `v1.10.1`, pgx `v5.10.0`, JWT `v5.3.1`, Prometheus `v1.25.0` | `backend/go.mod`, `backend/cmd/api/main.go` |
| **Frontend Web** | Next.js `14.2.18` (App Router), React `18.3.1`, TypeScript `5.6.3`, Tailwind CSS `3.4.15`, Lucide React `0.460.0` | `frontend/nextjs/package.json` |
| **Reverse Proxy & Storage** | Nginx `1.27-alpine` (WebDAV direct upload `PUT /storage/upload/(.+)` max `25M` + static `GET /storage/uploads/`) | `docker-compose.yml`, `deploy/nginx/conf.d/app.conf` |
| **Real-Time & Background** | SSE Broker (`/api/v1/notifications/stream`), Outbox Worker (poll `2s`, batch `20`, retry `5`), Auto-Complete Worker (interval `30s`) | `backend/cmd/api/main.go` |
| **Security Middleware** | CORS, RequestID, Recover, AccessLogger, PrometheusMetrics, SecurityHeaders, IP RateLimiter (`10 req/s burst 20`; Login `1 req/s burst 5`), CSRF Double-Submit Cookie (`csrf_token` / `X-CSRF-Token`), JWT Auth (`Bearer` / Cookie `access_token` / Query `token` khusus SSE) | `backend/internal/middleware/*` |

---

## 3. Inventaris Lengkap Halaman Frontend (22 Route Aktual)

| No | Route | Role Guard | Komponen File | Tujuan & Catatan Aktual |
| :---: | :--- | :--- | :--- | :--- |
| 1 | `/` | Public | `src/app/page.tsx` | Server redirect langsung ke `/student/dashboard`. |
| 2 | `/login` | Public | `src/app/login/page.tsx` | Alias/redirect ke halaman login. |
| 3 | `/auth/login` | Public | `src/app/auth/login/page.tsx` | Form login NIM/NIP & Kata Sandi SSO + healthcheck `/healthz` failover modal. |
| 4 | `/admin/dashboard` | `ADMIN` | `src/app/admin/dashboard/page.tsx` | Overview metrik, kalender gabungan seluruh dosen (kuliah, blokir, janji temu), filter dosen, Export CSV janji temu. |
| 5 | `/admin/users` | `ADMIN` | `src/app/admin/users/page.tsx` | CRUD pengguna (`DOSEN` & `MAHASISWA`), pencarian, filter role, paginasi (`5/hal`), Batch Import CSV, Export CSV. |
| 6 | `/admin/academic-calendar` | `ADMIN` | `src/app/admin/academic-calendar/page.tsx` | Tambah agenda libur (`is_holiday: true` hardcoded) & hapus agenda kalender akademik (belum ada form Edit di UI). |
| 7 | `/admin/feedbacks` | `ADMIN` | `src/app/admin/feedbacks/page.tsx` | Statistik 4 metrik, filter status, pencarian, daftar masukan ber-Date Separator, modal evaluasi status & `admin_notes`. |
| 8 | `/admin/system-settings` | `ADMIN` | `src/app/admin/system-settings/page.tsx` | Edit 7 parameter `appointment_rules` (`min/default/max_minutes`, `buffer`, `slot_interval`, `min_lead_days`, `max_advance_days`). |
| 9 | `/admin/audit-logs` | `ADMIN` | `src/app/admin/audit-logs/page.tsx` | Daftar log audit tamper-evident, filter aksi, pencarian debounce `300ms`, paginasi, modal inspeksi JSON + Copy. |
| 10 | `/lecturer/dashboard` | `DOSEN` | `src/app/lecturer/dashboard/page.tsx` | Kalender agenda dosen (kuliah, blokir, janji temu), panel permohonan menunggu, quick Approve/Reject, deep-link `?appointment_id=` & SSE live reload. |
| 11 | `/lecturer/appointments` | `DOSEN` | `src/app/lecturer/appointments/page.tsx` | Metrik ringkas, tab status, Date Separator, pencarian, detail modal, aksi Approve/Reject/Cancel dengan alasan. |
| 12 | `/lecturer/teaching-schedules` | `DOSEN` | `src/app/lecturer/teaching-schedules/page.tsx` | 2 tab: `Jadwal Mengajar` (tambah & hapus) dan `Jadwal Terblokir` (tambah & hapus dengan normalisasi UTC). |
| 13 | `/lecturer/availabilities` | `DOSEN` | `src/app/lecturer/availabilities/page.tsx` | **Redirect otomatis** ke `/lecturer/appointments` (tidak memiliki UI input sendiri). |
| 14 | `/lecturer/notifications` | `DOSEN` | `src/app/lecturer/notifications/page.tsx` | Pusat notifikasi dosen, Date Separator, tandai dibaca per item / semua. |
| 15 | `/lecturer/feedback` | `DOSEN` | `src/app/lecturer/feedback/page.tsx` | 2 tab: `Buat Masukan Baru` (`FeedbackForm` + `LMSRichEditor` + `MediaDropzone`) & `Riwayat Masukan` (ber-Date Separator). |
| 16 | `/lecturer/profile` | `DOSEN` | `src/app/lecturer/profile/page.tsx` | Profil dosen (read-only), statistik pending/selesai, tombol Logout dengan `ConfirmModal`. |
| 17 | `/student/dashboard` | `MAHASISWA` | `src/app/student/dashboard/page.tsx` | Kalender 2 mode (`MY` & `LECTURER`), pilih dosen, klik slot kosong / FAB membuka `BookingBottomSheet`. |
| 18 | `/student/appointments` | `MAHASISWA` | `src/app/student/appointments/page.tsx` | Tab status (`ALL`, `MENUNGGU_PERSETUJUAN`, `DISETUJUI`, `SELESAI`, `BENTROK`, `DITOLAK`, `DIBATALKAN`), Date Separator, pencarian, tombol booking. |
| 19 | `/student/appointments/[id]` | `MAHASISWA` | `src/app/student/appointments/[id]/page.tsx` | Detail janji temu, cek kepemilikan frontend, modal Reschedule (pilih slot tersedia), modal Batalkan Janji Temu. |
| 20 | `/student/notifications` | `MAHASISWA` | `src/app/student/notifications/page.tsx` | Pusat notifikasi mahasiswa, Date Separator, tandai dibaca per item / semua. |
| 21 | `/student/feedback` | `MAHASISWA` | `src/app/student/feedback/page.tsx` | 2 tab: `Buat Masukan Baru` (`FeedbackForm`) & `Riwayat Masukan` (ber-Date Separator). |
| 22 | `/student/profile` | `MAHASISWA` | `src/app/student/profile/page.tsx` | Profil mahasiswa (read-only), statistik 5 status janji temu, tombol Logout dengan `ConfirmModal`. |

---

## 4. Ringkasan Endpoint Backend (`/api/v1/*`)

Total **44 endpoint** terverifikasi di `backend/cmd/api/main.go`:
- **System (2)**: `GET /healthz`, `GET /metrics`
- **Auth (3)**: `POST /auth/login`, `POST /auth/refresh`, `POST /auth/logout`
- **Academic Calendar (4)**: `GET /academic-calendar`, `POST/PUT/DELETE /academic-calendar[/:id]`
- **Teaching & Blocked Schedules (9)**: `POST /teaching-schedules`, `GET /teaching-schedules/my`, `GET /teaching-schedules/lecturer/:lecturer_id`, `GET /teaching-schedules/lecturer/:lecturer_id/activities`, `PUT/DELETE /teaching-schedules/:id`, `POST /blocked-schedules`, `GET /blocked-schedules/my`, `DELETE /blocked-schedules/:id`
- **Availabilities (4)**: `POST /availabilities`, `GET /availabilities/my`, `DELETE /availabilities/:id`, `GET /availabilities/slots`
- **Appointments & Reschedule (9)**: `POST/GET /appointments`, `GET /appointments/:id`, `PATCH /appointments/:id/{approve,reject,cancel}`, `POST/GET /appointments/:id/reschedule`, `PATCH /reschedule-requests/:request_id/respond`
- **Users & Notifications (7)**: `GET/PUT /users/me`, `GET /lecturers`, `GET /notifications`, `GET /notifications/stream`, `PATCH /notifications/:id/read`, `PATCH /notifications/read-all`
- **Feedbacks & Direct Storage (4 + 2 Nginx)**: `POST /feedbacks`, `GET /feedbacks/my`, `GET /feedbacks/:id`, `POST /feedbacks/upload-intent` (+ Nginx `PUT /storage/upload/(.+)`, `GET /storage/uploads/`)
- **Settings (1)**: `GET /settings/appointment-rules`
- **Admin Group (17)**: `/admin/users` (GET/POST/PUT/DELETE/batch-import/export), `/admin/audit-logs`, `/admin/appointments/export`, `/admin/blocked-schedules`, `/admin/teaching-schedules`, `/admin/overview`, `/admin/settings`, `/admin/settings/appointment-rules` (GET/PUT), `/admin/feedbacks` (GET, GET `/:id`, PATCH `/:id/status`).
