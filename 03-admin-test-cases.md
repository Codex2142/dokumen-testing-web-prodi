# 03 — Admin Test Cases (`ADMIN`)

**Akun Uji**: `198001012005011001` (`Admin Prodi Informatika`) / Password: `password123`  
**Klasifikasi Bukti**: Seluruh test case di bawah berstatus `VERIFIED` dari source code aktual kecuali ditandai lain.

---

## Role: ADMIN

### Feature: Autentikasi & Dashboard Admin

#### TC-ADM-01 — Login Berhasil & Redirect ke Dashboard Admin
| Field | Description |
|---|---|
| **Test Category** | Functional / Integration |
| **Priority** | Critical |
| **Objective** | Memastikan akun `ADMIN` dapat login dan diarahkan ke `/admin/dashboard` dengan data overview termuat |
| **Preconditions** | Browser dalam kondisi belum login (cookies bersih), server VPS & DB `db_cadangan` aktif |
| **Test Data** | NIM/NIP: `198001012005011001`, Password: `password123` |
| **Route** | `/auth/login` → `/admin/dashboard` |
| **Expected Result** | Toast sukses muncul, redirect ke `/admin/dashboard`, 6 menu sidebar tampil (`Dashboard & Kalender`, `Kelola Pengguna`, `Kalender Akademik`, `Umpan Balik`, `Pengaturan Sistem`, `Audit Logs`) |
| **Referensi Kode** | `frontend/nextjs/src/app/auth/login/page.tsx:101-125`, `backend/internal/service/auth_service.go:45-100`, `POST /api/v1/auth/login`, `GET /api/v1/admin/overview` |
| **Status QA** | `[✅] PASS` `[ ] FAIL` `[ ] BLOCKED` — Catatan: Sistem berhasil memasukan user ke sesi Admin Dashboard |

**Manual Testing Instructions**:
1. Buka `https://38-9-47-207.sslip.io/auth/login`.
2. Pastikan banner `"Server Sedang Dalam Perbaikan"` tidak muncul.
3. Masukkan `198001012005011001` pada kolom `NIM / NIP` dan `password123` pada kolom `Kata Sandi SSO`.
4. Klik tombol masuk.
5. Amati redirect ke `/admin/dashboard`, kartu overview, dan timeline kalender.

**Database Verification**:
```sql
SELECT user_id, ip_address, is_revoked, expires_at FROM public.user_sessions ORDER BY created_at DESC LIMIT 1;
```

---

#### TC-ADM-02 — Export Laporan Janji Temu CSV dari Dashboard Admin
| Field | Description |
|---|---|
| **Test Category** | Functional / Integration |
| **Priority** | Medium |
| **Objective** | Memastikan Admin dapat mengunduh file CSV seluruh janji temu |
| **Preconditions** | Login sebagai `ADMIN` di `/admin/dashboard` |
| **Test Data** | Minimal 0 atau 1 janji temu di DB |
| **Route** | `/admin/dashboard` |
| **Expected Result** | Browser mengunduh file `.csv` dengan 15 kolom header (`ID Janji Temu` s/d `Tanggal Diajukan`) dan toast `"File CSV laporan janji temu berhasil diunduh."` muncul |
| **Referensi Kode** | `frontend/nextjs/src/app/admin/dashboard/page.tsx:147-160`, `backend/internal/service/admin_service.go:183-246`, `GET /api/v1/admin/appointments/export` |
| **Status QA** | `[✅] PASS` `[ ] FAIL` `[ ] BLOCKED` — Catatan: File CSV Berhasil didownload, dan berhasil menampilkan data didalam file CSV |

**Manual Testing Instructions**:
1. Buka `/admin/dashboard`.
2. Klik tombol export CSV laporan janji temu.
3. Buka file `.csv` yang terunduh dan periksa header kolom serta datanya.

---

### Feature: Kelola Pengguna (`/admin/users`)

#### TC-ADM-03 — Tambah Pengguna Baru (Dosen & Mahasiswa) dengan Validasi Sukses
| Field | Description |
|---|---|
| **Test Category** | Functional / Integration |
| **Priority** | Critical |
| **Objective** | Memastikan Admin dapat membuat akun `DOSEN` dan `MAHASISWA` baru yang tersimpan di DB |
| **Preconditions** | Login sebagai `ADMIN`, buka `/admin/users` |
| **Test Data** | NIM: `1203229901`, Nama: `QA Student Baru`, Email: `qa.student@student.telkomuniversity.ac.id`, Role: `MAHASISWA` |
| **Route** | `/admin/users` |
| **Expected Result** | Toast `"Pengguna QA Student Baru berhasil ditambahkan."` muncul, modal tertutup, user baru tampil di tabel, dan dapat login dengan default password `password123` |
| **Referensi Kode** | `frontend/nextjs/src/app/admin/users/page.tsx:119-180`, `backend/internal/service/admin_service.go:271-342`, `POST /api/v1/admin/users` |
| **Status QA** | `[✅] PASS` `[ ] FAIL` `[ ] BLOCKED` — Catatan: Pengguna baru berhasil ditambahkan, dan sistem berhasil memberikan alert |

**Manual Testing Instructions**:
1. Buka menu `Kelola Pengguna` (`/admin/users`).
2. Klik tombol tambah pengguna baru.
3. Isi NIM/NIP `1203229901`, Nama `QA Student Baru`, Email `qa.student@student.telkomuniversity.ac.id`, pilih Role `MAHASISWA`.
4. Klik simpan dan amati daftar tabel pengguna.

**Database Verification**:
```sql
SELECT identifier, name, email, role, is_active FROM public.users WHERE identifier = '1203229901';
```

---

#### TC-ADM-04 — Validasi Negative & Boundary Form Tambah Pengguna (NIM < 10 Digit, Non-Angka, Domain Email Salah, Duplikat)
| Field | Description |
|---|---|
| **Test Category** | Negative / Boundary |
| **Priority** | High |
| **Objective** | Memverifikasi seluruh aturan validasi pembuatan akun pada frontend dan backend |
| **Preconditions** | Modal tambah pengguna terbuka di `/admin/users` |
| **Test Data** | Case A: NIM `123456789` (9 digit); Case B: NIM `12032201AB`; Case C: Role `DOSEN` + Email `dosen@gmail.com`; Case D: NIM duplikat `1203220154` |
| **Route** | `/admin/users` |
| **Expected Result** | Case A & B ditolak dengan toast `"NIM/NIP wajib berupa angka dan minimal 10 digit."`; Case C ditolak `"Email Dosen wajib menggunakan domain resmi @telkomuniversity.ac.id"`; Case D ditolak backend `"NIM/NIP '1203220154' sudah digunakan"` |
| **Referensi Kode** | `frontend/nextjs/src/app/admin/users/page.tsx:127-156`, `backend/internal/service/admin_service.go:276-304` |
| **Status QA** | `[✅] PASS` `[ ] FAIL` `[ ] BLOCKED` — Catatan: Sistem berhasil mencegah user memberi NIM di bawah 10 angka, sistem juga berhasil mencegah user untuk mengisi dengan format yang salah (contoh: NIM menggunakan huruf), kemudian sistem juga mencegah user mengisi NIM yang sudah ada / mencegah duplikasi |

**Manual Testing Instructions**:
1. Di modal tambah pengguna, masukkan Case A (`123456789`) lalu klik simpan → amati error toast.
2. Ubah NIM ke Case B (`12032201AB`) lalu simpan → amati error toast.
3. Pilih Role `DOSEN`, masukkan NIM 10+ digit dan email `dosen@gmail.com` → amati error toast.
4. Masukkan NIM yang sudah terdaftar `1203220154` dengan email valid baru → amati penolakan dari server.

---

#### TC-ADM-05 — Nonaktifkan Akun User & Verifikasi Revokasi Sesi Otomatis
| Field | Description |
|---|---|
| **Test Category** | Integration / Security |
| **Priority** | High |
| **Objective** | Memastikan menonaktifkan akun (`is_active = false`) mencabut seluruh sesi aktif user dan mencegah login ulang |
| **Preconditions** | Akun uji `1203229901` sudah pernah login di browser Incognito (punya baris `user_sessions` aktif) |
| **Test Data** | Edit akun `1203229901` → ubah status aktif menjadi `Nonaktif` |
| **Route** | `/admin/users` |
| **Expected Result** | Status user berubah `Nonaktif`, seluruh baris `user_sessions` milik user di-set `is_revoked = true`, dan percobaan login ulang user tersebut gagal dengan pesan akun tidak aktif |
| **Referensi Kode** | `backend/internal/service/admin_service.go:392-394`, `backend/internal/service/auth_service.go:54-56`, `PUT /api/v1/admin/users/:id` |
| **Status QA** | `[✅] PASS` `[ ] FAIL` `[ ] BLOCKED` — Catatan: Sistem berhasil menyimpan status yang baru (non-aktif), kemudian di halaman mahasiswa yang dinon-aktifkan sistem berhasil menampilkan alert bahwa akun sedang di non-aktifkan mohon hubungi administrasi |

**Manual Testing Instructions**:
1. Di `/admin/users`, cari `1203229901` dan klik ikon Edit.
2. Ubah status akun menjadi tidak aktif lalu simpan.
3. Di jendela Incognito, coba login menggunakan `1203229901` / `password123`.
4. Amati error pada halaman login dan verifikasi DB.

**Database Verification**:
```sql
SELECT is_active FROM public.users WHERE identifier = '1203229901';
SELECT count(*) AS active_sessions FROM public.user_sessions s JOIN public.users u ON u.id = s.user_id WHERE u.identifier = '1203229901' AND s.is_revoked = FALSE;
```

---

#### TC-ADM-06 — Hapus Pengguna & Proteksi Akun Admin
| Field | Description |
|---|---|
| **Test Category** | Functional / Authorization |
| **Priority** | High |
| **Objective** | Memastikan akun uji dapat dihapus dengan konfirmasi modal, dan akun `ADMIN` tidak muncul di daftar serta tidak bisa dihapus via API |
| **Preconditions** | Login sebagai `ADMIN` di `/admin/users`, akun uji `1203229901` tersedia |
| **Test Data** | Target hapus: `1203229901` |
| **Route** | `/admin/users` |
| **Expected Result** | Akun `ADMIN` tidak tampil di tabel; klik hapus pada `1203229901` memunculkan `ConfirmModal`; setelah dikonfirmasi, user terhapus dari tabel dan DB |
| **Referensi Kode** | `frontend/nextjs/src/app/admin/users/page.tsx:98`, `backend/internal/service/admin_service.go:399-409`, `DELETE /api/v1/admin/users/:id` |
| **Status QA** | `[✅] PASS` `[ ] FAIL` `[ ] BLOCKED` — Catatan: Sistem memberi konfirmasi untuk role yang akan dihapus, kemudian sistem berhasil menghapus role mahasiswa juga sistem berhasil memberikan alert bahwa sistem berhasil menghapus data |

**Manual Testing Instructions**:
1. Pastikan `Admin Prodi Informatika` tidak muncul pada tabel `/admin/users`.
2. Klik ikon Hapus (tempat sampah) pada baris `QA Student Baru` (`1203229901`).
3. Konfirmasi penghapusan pada modal konfirmasi.
4. Pastikan baris hilang dari tabel.

---

#### TC-ADM-07 — Batch Import CSV & Export Users CSV
| Field | Description |
|---|---|
| **Test Category** | Functional / Negative |
| **Priority** | Medium |
| **Objective** | Memastikan import CSV memproses baris valid, melaporkan baris error, dan export CSV menghasilkan file tanpa akun `ADMIN` |
| **Preconditions** | Login sebagai `ADMIN` di `/admin/users` |
| **Test Data** | File `.csv` berisi 1 baris valid & 1 baris invalid (email/NIM salah) |
| **Route** | `/admin/users` |
| **Expected Result** | Modal import menampilkan `total_processed`, `success_count`, `error_count`, dan rincian `errors`; tombol Export CSV mengunduh daftar user non-admin |
| **Referensi Kode** | `backend/internal/service/admin_service.go:54-181`, `POST /api/v1/admin/users/batch-import`, `GET /api/v1/admin/users/export` |
| **Status QA** | `[ ] PASS` `[ ] FAIL` `[ ] BLOCKED` — Catatan: |

**Manual Testing Instructions**:
1. Klik tombol Import CSV di `/admin/users`, unggah file CSV uji, dan klik proses.
2. Amati ringkasan jumlah sukses dan daftar baris error.
3. Klik tombol Export CSV dan buka file hasil unduhan; pastikan baris `ADMIN` tidak ikut terekspor.

---

### Feature: Kalender Akademik (`/admin/academic-calendar`)

#### TC-ADM-08 — Tambah & Hapus Hari Libur Kalender Akademik
| Field | Description |
|---|---|
| **Test Category** | Functional / Negative |
| **Priority** | Critical |
| **Objective** | Memastikan Admin dapat menambah agenda libur akademik (dan validasi `end_date < start_date` ditolak) serta menghapusnya |
| **Preconditions** | Login sebagai `ADMIN` di `/admin/academic-calendar` |
| **Test Data** | Judul: `[UAT-2026] Libur Simulasi QA`, Tipe: `CUTI_BERSAMA`, Tanggal Mulai & Selesai: Jumat pekan depan |
| **Route** | `/admin/academic-calendar` |
| **Expected Result** | Jika `end_date < start_date` → ditolak backend `"end_date tidak boleh lebih awal dari start_date"`. Jika valid → tersimpan dengan `is_holiday = true`, tampil di daftar, dan dapat dihapus lewat `ConfirmModal` |
| **Referensi Kode** | `frontend/nextjs/src/app/admin/academic-calendar/page.tsx:68-119`, `backend/internal/service/calendar_service.go:38-102`, `POST/DELETE /api/v1/academic-calendar` |
| **Status QA** | `[✅] PASS` `[ ] FAIL` `[ ] BLOCKED` — Catatan: Sistem berhasil menampilkan pesan error saat tanggal selesai diinput lebih awal dari tanggal mulai, Sistem juga berhasil menyimpan pengisian data yang benar, dan menampilkan alert berhasil |

**Manual Testing Instructions**:
1. Buka menu `Kalender Akademik` (`/admin/academic-calendar`).
2. Uji negatif: isi tanggal selesai lebih awal dari tanggal mulai lalu submit → amati pesan error.
3. Isi Judul `[UAT-2026] Libur Simulasi QA`, pilih tanggal mulai & selesai Jumat pekan depan, pilih tipe `CUTI_BERSAMA`, lalu simpan.
4. Pastikan agenda muncul di tabel dan simpan untuk dipakai pada pengujian Cross-Role Mahasiswa (hapus setelah Cross-Role selesai).

**Database Verification**:
```sql
SELECT id, title, start_date, end_date, type, is_holiday FROM public.academic_calendar ORDER BY created_at DESC;
```

---

### Feature: Pengaturan Sistem (`/admin/system-settings`)

#### TC-ADM-09 — Pembaruan & Validasi Batas Aturan Janji Temu (`appointment_rules`)
| Field | Description |
|---|---|
| **Test Category** | Functional / Boundary / Negative |
| **Priority** | High |
| **Objective** | Memastikan validasi relasi antar angka aturan berjalan (contoh `min_minutes > max_minutes` ditolak) dan pembaruan valid tersimpan ke `system_settings` |
| **Preconditions** | Login sebagai `ADMIN` di `/admin/system-settings` |
| **Test Data** | Negative: `min_minutes=90, max_minutes=60`; Valid: `min=15, default=30, max=60, buffer=10, slot_interval=15, min_lead=1, max_advance=30` |
| **Route** | `/admin/system-settings` |
| **Expected Result** | Input negatif menampilkan banner error merah `"Durasi minimum tidak boleh lebih besar dari durasi maksimum."`; input valid menampilkan toast hijau sukses dan nilai tetap bertahan setelah reload |
| **Referensi Kode** | `frontend/nextjs/src/app/admin/system-settings/page.tsx:73-109`, `backend/internal/service/settings_service.go:28-60`, `PUT /api/v1/admin/settings/appointment-rules` |
| **Status QA** | `[✅] PASS` `[ ] FAIL` `[ ] BLOCKED` — Catatan: Saat durasi minimal di naikkan menjadi 90 menit (lebih besar dari maksimal) sistem tidak dapat menyimpan dan memberikan pesan error "Durasi minimum tidak boleh lebih besar dari durasi maksimum." Sistem berhasil menyimpan saat durasi minimal dikembalikan menjadi 15 menit, sistem juga menampilkan pesan "berhasil" |

**Manual Testing Instructions**:
1. Buka menu `Pengaturan Sistem` (`/admin/system-settings`).
2. Uji negatif: ubah `Durasi Minimal (Menit)` menjadi `90` (lebih besar dari maksimal `60`), klik simpan → amati banner error.
3. Kembalikan ke `15`, ubah `max_advance_days` menjadi `30`, lalu simpan.
4. Refresh halaman (F5) dan pastikan nilai tetap `30`.

---

### Feature: Manajemen Umpan Balik (`/admin/feedbacks`)

#### TC-ADM-10 — Filter, Pencarian, Inspeksi Lampiran, & Evaluasi Status Masukan Pengguna
| Field | Description |
|---|---|
| **Test Category** | Functional / Integration / UX |
| **Priority** | Critical |
| **Objective** | Memastikan Admin dapat memfilter masukan, membuka `AdminFeedbackDetailModal`, melihat gambar via Lightbox, mengubah status, dan menyimpan `admin_notes` |
| **Preconditions** | Minimal 1 feedback telah dikirim oleh Mahasiswa/Dosen (dari `TC-MHS-11` atau `TC-DSN-09`) |
| **Test Data** | Status baru: `SEDANG_DITINJAU` lalu `SELESAI`; Catatan Admin: `Kendala telah diperbaiki dan diverifikasi.` |
| **Route** | `/admin/feedbacks` |
| **Expected Result** | 4 kartu statistik akurat; Date Separator mengelompokkan laporan per tanggal; klik gambar membuka Lightbox (bisa ditutup via `ESC`); setelah klik simpan, toast `"Evaluasi feedback berhasil disimpan."` muncul dan badge berubah menjadi `Selesai Dibenahi` |
| **Referensi Kode** | `frontend/nextjs/src/app/admin/feedbacks/page.tsx`, `frontend/nextjs/src/components/feedback/AdminFeedbackDetailModal.tsx`, `PATCH /api/v1/admin/feedbacks/:id/status` |
| **Status QA** | `[ ] PASS` `[ ] FAIL` `[ ] BLOCKED` — Catatan: |

**Manual Testing Instructions**:
1. Buka menu `Umpan Balik` (`/admin/feedbacks`).
2. Periksa 4 stat card (`Total Masukan Masuk`, `Menunggu Review`, dst.) dan Date Separator.
3. Ketik judul masukan pada kotak pencarian dan uji dropdown filter status.
4. Klik kartu masukan untuk membuka modal detail.
5. Klik thumbnail gambar lampiran untuk menguji Lightbox, lalu tekan tombol `ESC` untuk menutup Lightbox.
6. Ubah dropdown status ke `Selesai Dibenahi` (`SELESAI`), isi catatan tindak lanjut, lalu klik tombol simpan.

**Database Verification**:
```sql
SELECT id, title, status, admin_notes, updated_at FROM public.feedbacks ORDER BY created_at DESC LIMIT 5;
```

---

### Feature: Audit Trail Keamanan (`/admin/audit-logs`)

#### TC-ADM-11 — Pencarian, Filter Aksi, Paginasi, & Inspeksi Metadata JSON Audit Logs
| Field | Description |
|---|---|
| **Test Category** | Functional / UX |
| **Priority** | Medium |
| **Objective** | Memastikan seluruh aktivitas penting tercatat di `audit_logs`, dapat dicari (debounce 300ms), difilter, dan metadata JSON dapat disalin |
| **Preconditions** | Login sebagai `ADMIN`, sudah ada beberapa aktivitas (login, create user, update settings) |
| **Test Data** | Kata kunci pencarian: `LOGIN` atau IP lokal |
| **Route** | `/admin/audit-logs` |
| **Expected Result** | Tabel menampilkan log terbaru, pencarian merespons halus tanpa spam request, klik lihat JSON membuka modal dan tombol Copy menampilkan toast `"Metadata JSON disalin ke clipboard."` |
| **Referensi Kode** | `frontend/nextjs/src/app/admin/audit-logs/page.tsx`, `GET /api/v1/admin/audit-logs` |
| **Status QA** | `[✅] PASS` `[ ] FAIL` `[ ] BLOCKED` — Catatan: Sistem berhasil mencari melalui search box dan berhasil memfilter dari dropdown. Sistem juga berhasil menampilkan adata JSON pada baris yang dipilih |

**Manual Testing Instructions**:
1. Buka menu `Audit Logs` (`/admin/audit-logs`).
2. Ketik `LOGIN` pada kotak pencarian `Cari aksi, IP address, resource...`.
3. Uji dropdown filter aksi dan ubah jumlah item per halaman.
4. Klik ikon lihat JSON pada salah satu baris, lalu klik tombol Copy.

---

### Feature: Authorization Testing (`ADMIN`)

#### TC-ADM-AUTH-01 — Admin Ditolak Mengakses Halaman & Action Khusus Role Lain
| Field | Description |
|---|---|
| **Test Category** | Authorization |
| **Priority** | High |
| **Objective** | Memastikan Admin diredirect saat membuka `/student/dashboard` atau `/lecturer/dashboard`, dan ditolak backend (`403`) saat memanggil `POST /api/v1/appointments` |
| **Preconditions** | Login sebagai `ADMIN` |
| **Test Data** | URL: `/student/dashboard`, `/lecturer/dashboard`; API: `POST /api/v1/appointments` |
| **Route** | `/student/dashboard`, `/lecturer/dashboard` |
| **Expected Result** | Frontend `RoleGuard` langsung mengembalikan ke `/admin/dashboard`; request langsung ke `POST /api/v1/appointments` mengembalikan HTTP `403 FORBIDDEN` |
| **Referensi Kode** | `frontend/nextjs/src/components/common/RoleGuard.tsx:31-34`, `backend/cmd/api/main.go:232` |
| **Status QA** | `[✅] PASS` `[ ] FAIL` `[ ] BLOCKED` — Catatan: Sistem tetap membawa user kepada halaman login awal. |

**Manual Testing Instructions**:
1. Saat login sebagai `ADMIN`, ketik manual `https://38-9-47-207.sslip.io/student/dashboard` di address bar → amati redirect kembali ke `/admin/dashboard`.
2. Ulangi untuk `/lecturer/dashboard`.
3. (Opsional via DevTools Console/curl dengan cookie sesi Admin) kirim `POST /api/v1/appointments` → pastikan response `403 FORBIDDEN`.
