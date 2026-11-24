# 12 — Notifikasi Email (8 Event), Custom Reminder, & Broadcast Notifikasi Sistem Admin

Dokumen ini berisi **14 test case (`TC-NTF-01` s/d `TC-NTF-14`)** untuk menguji 3 fitur baru yang telah dideploy ke VPS (`https://38-9-47-207.sslip.io`):

1. **Notifikasi Email 8 Event Alur Janji Temu & Reschedule + Toggle Profil**
2. **Custom Reminder Menit (In-App Lonceng SSE + Email)**
3. **Broadcast Notifikasi Sistem oleh Admin (`/admin/notifications` — 4 Mode Target, In-App SSE Only)**

---

## Bagian 0 — Panduan Persiapan Singkat (Untuk QA Pemula)

Agar pengujian berjalan mulus tanpa bingung:

1. **Buka 2 Jendela Browser**:
    - **Jendela 1 (Normal)**: Untuk login sebagai **Mahasiswa** atau **Admin**.
    - **Jendela 2 (Incognito — `Cmd+Shift+N` / `Ctrl+Shift+N`)**: Untuk login sebagai **Dosen** secara bersamaan agar Anda bisa melihat notifikasi masuk secara real-time.
2. **Siapkan Email Uji yang Bisa Anda Buka**:
    - Pastikan akun Mahasiswa uji dan/atau Dosen uji memiliki alamat email aktif (misal Gmail pribadi Anda) yang bisa dibuka Inbox/Spam-nya. _(Admin dapat mengganti email akun uji lewat menu **Kelola Pengguna** di `/admin/users`.)_
3. **Catatan Waktu Kirim**:
    - Notifikasi in-app (lonceng/SSE) muncul dalam **≤ 4 detik**.
    - Notifikasi email (Gmail SMTP) masuk ke Inbox/Spam dalam **3–15 detik** (asinkron, tidak memperlambat klik tombol).
    - Pengingat otomatis (_Reminder_) dicek oleh server setiap **30 detik**.

---

## Bagian A — Notifikasi Email 8 Event Janji Temu & Toggle Profil

### Feature: Email Otomatis Alur Janji Temu & Reschedule

#### TC-NTF-01 — Pengajuan Janji Temu Baru (`APPOINTMENT_REQUESTED`) → Email Masuk ke Dosen

| Field               | Description                                                                                                                                                                                                                                                                                                             |
| ------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Category**   | Functional / Integration (Email SMTP)                                                                                                                                                                                                                                                                                   |
| **Priority**        | Critical                                                                                                                                                                                                                                                                                                                |
| **Objective**       | Memastikan saat Mahasiswa mengajukan janji temu baru, Dosen menerima notifikasi email otomatis dengan format rapi (`#9E1B32`) berisi nama/NIM mahasiswa, topik, jadwal, dan lokasi                                                                                                                                      |
| **Preconditions**   | Toggle **Notifikasi Email Janji Temu** di `/lecturer/profile` berstatus **Aktif**; alamat email Dosen uji dapat dibuka                                                                                                                                                                                                  |
| **Test Data**       | Mahasiswa mengajukan janji temu ke Dosen uji dengan topik `[QA-EMAIL] Uji Pengajuan`                                                                                                                                                                                                                                    |
| **Route**           | `/student/dashboard` → Inbox Email Dosen                                                                                                                                                                                                                                                                                |
| **Expected Result** | 1) Submit booking berjalan cepat tanpa delay SMTP. 2) Dalam 3–15 detik, email berjudul **`Pengajuan Janji Temu Baru - Janjian Dosen Informatika`** masuk ke email Dosen dengan badge kuning **`MENUNGGU PERSETUJUAN`**, nama & NIM mahasiswa, topik, jadwal, lokasi, dan catatan cara mematikan email di halaman Profil |
| **Referensi Kode**  | `backend/internal/notification/email_sender.go` (`FormatOutboxEmail`), `backend/internal/notification/whatsapp_stub.go` (`CompositeDispatcher`)                                                                                                                                                                         |
| **Status QA**       | `[ ] PASS` `[ ] FAIL` `[ ] BLOCKED` — Catatan:                                                                                                                                                                                                                                                                          |

**Langkah Pengujian (Klik UI):**

1. Di Jendela 2 (Dosen), buka `/lecturer/profile` → pastikan switch **Notifikasi Email Janji Temu** dalam posisi **Aktif** (merah menyala).
2. Di Jendela 1 (Mahasiswa), buka `/student/dashboard`, pilih Dosen uji, lalu buat janji temu baru topik `[QA-EMAIL] Uji Pengajuan`.
3. Buka Inbox (atau Spam) email Dosen uji.

- **PASS jika**: Email masuk dengan detail lengkap & format rapi.
- **FAIL jika**: Email tidak masuk setelah > 1 menit atau format berantakan.

---

#### TC-NTF-02 — Persetujuan (`APPROVED`), Penolakan (`REJECTED`), & Bentrok (`CONFLICTED`) → Email ke Mahasiswa

| Field               | Description                                                                                                                                                                                                                                                                        |
| ------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Category**   | Functional / Integration (Email SMTP)                                                                                                                                                                                                                                              |
| **Priority**        | Critical                                                                                                                                                                                                                                                                           |
| **Objective**       | Memastikan Mahasiswa menerima email untuk status **Disetujui**, **Ditolak** (beserta alasan), dan **Bentrok Otomatis**                                                                                                                                                             |
| **Preconditions**   | Toggle **Notifikasi Email Janji Temu** di `/student/profile` **Aktif**; tersedia pengajuan janji temu `MENUNGGU_PERSETUJUAN`                                                                                                                                                       |
| **Test Data**       | 1) Dosen klik **Setujui** pada janji temu A; 2) Dosen klik **Tolak** pada janji temu B dengan alasan `Dosen sedang rapat`; 3) _(Opsional)_ Dua pengajuan pada slot sama lalu salah satunya disetujui agar yang lain otomatis `BENTROK`                                             |
| **Route**           | `/lecturer/dashboard` → Inbox Email Mahasiswa                                                                                                                                                                                                                                      |
| **Expected Result** | 1) Approve → Email **`Janji Temu Disetujui`** (badge hijau `DISETUJUI`, nama dosen, topik, jadwal). 2) Reject → Email **`Janji Temu Ditolak`** (badge merah `DITOLAK` + alasan `Dosen sedang rapat`). 3) Conflict → Email **`Jadwal Janji Temu Bentrok`** (badge oranye `BENTROK`) |
| **Referensi Kode**  | `backend/internal/repository/appointment_repo.go:354-510`, `backend/internal/notification/email_sender.go`                                                                                                                                                                         |
| **Status QA**       | `[ ] PASS` `[ ] FAIL` `[ ] BLOCKED` — Catatan:                                                                                                                                                                                                                                     |

**Langkah Pengujian (Klik UI):**

1. Di Jendela 2 (Dosen), klik **Setujui** pada 1 janji temu → cek Inbox email Mahasiswa.
2. Klik **Tolak** pada 1 janji temu lain dengan alasan `Dosen sedang rapat` → cek Inbox email Mahasiswa apakah alasan tertulis jelas.

- **PASS jika**: Email diterima sesuai status & alasan penolakan tampil akurat.

---

#### TC-NTF-03 — Pembatalan (`CANCELLED`) & Seluruh Alur Reschedule (`REQUESTED`, `ACCEPTED`, `REJECTED`)

| Field               | Description                                                                                                                                                                                                                                                                                                                                                            |
| ------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Category**   | Functional / Integration (Email SMTP)                                                                                                                                                                                                                                                                                                                                  |
| **Priority**        | High                                                                                                                                                                                                                                                                                                                                                                   |
| **Objective**       | Memastikan aksi Pembatalan janji temu dan seluruh alur Reschedule (Diajukan, Disetujui, Ditolak) mengirimkan email kepada pihak lawan/pengaju                                                                                                                                                                                                                          |
| **Preconditions**   | Kedua akun uji (Mahasiswa & Dosen) memiliki toggle Notifikasi Email **Aktif** dan memiliki janji temu berstatus `DISETUJUI`                                                                                                                                                                                                                                            |
| **Test Data**       | 1) Ajukan reschedule dengan alasan `Bentrok jadwal praktikum`; 2) Respon reschedule (`ACCEPT` atau `REJECT`); 3) Batalkan 1 janji temu dengan alasan `Berhalangan hadir`                                                                                                                                                                                               |
| **Route**           | `/student/appointments/[id]`, `/lecturer/dashboard` → Inbox Email                                                                                                                                                                                                                                                                                                      |
| **Expected Result** | - Reschedule Diajukan → Email **`Permohonan Ubah Jadwal (Reschedule)`** (badge biru `RESCHEDULE DIAJUKAN`, usulan waktu & alasan). <br> - Reschedule Disetujui/Ditolak → Email **`Permohonan Reschedule Disetujui`** / **`Ditolak`** ke pengaju. <br> - Dibatalkan → Email **`Janji Temu Dibatalkan`** (badge abu-abu `DIBATALKAN` + alasan pembatalan) ke pihak lawan |
| **Referensi Kode**  | `backend/internal/repository/appointment_repo.go:626-645`, `backend/internal/repository/reschedule_repo.go:100-390`                                                                                                                                                                                                                                                    |
| **Status QA**       | `[ ] PASS` `[ ] FAIL` `[ ] BLOCKED` — Catatan:                                                                                                                                                                                                                                                                                                                         |

---

#### TC-NTF-04 — Toggle Enable / Disable Notifikasi Email di `/student/profile` & `/lecturer/profile`

| Field               | Description                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| ------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Category**   | Functional / User Preference                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| **Priority**        | Critical                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| **Objective**       | Memastikan Mahasiswa & Dosen dapat mematikan dan menyalakan notifikasi email lewat halaman Profil, status tersimpan di database, dan saat **Nonaktif** email benar-benar tidak dikirim (sementara notifikasi in-app tetap masuk)                                                                                                                                                                                                                                |
| **Preconditions**   | Login Mahasiswa di `/student/profile` dan Dosen di `/lecturer/profile`                                                                                                                                                                                                                                                                                                                                                                                          |
| **Test Data**       | Klik switch **Notifikasi Email Janji Temu** dari `Aktif` → `Nonaktif`                                                                                                                                                                                                                                                                                                                                                                                           |
| **Route**           | `/student/profile`, `/lecturer/profile` (`PUT /api/v1/users/me`)                                                                                                                                                                                                                                                                                                                                                                                                |
| **Expected Result** | 1) Default awal adalah **Aktif**. 2) Saat diklik menjadi **Nonaktif**, muncul toast `"Notifikasi email janji temu dinonaktifkan."` dan badge berubah menjadi `Nonaktif`. 3) Refresh browser (`F5`) → status tetap **Nonaktif**. 4) Picu 1 event janji temu ke akun tersebut → notifikasi **tetap masuk ke lonceng aplikasi**, tetapi **tidak ada email** yang dikirim ke Inbox. 5) Nyalakan kembali ke **Aktif** → email kembali terkirim pada event berikutnya |
| **Referensi Kode**  | `frontend/nextjs/src/components/common/EmailNotificationToggleCard.tsx`, `backend/internal/notification/whatsapp_stub.go:128-140`                                                                                                                                                                                                                                                                                                                               |
| **Status QA**       | `[ ] PASS` `[ ] FAIL` `[ ] BLOCKED` — Catatan:                                                                                                                                                                                                                                                                                                                                                                                                                  |

---

#### TC-NTF-05 — Independensi Email Keamanan (OTP Reset Password Tetap Terkirim Saat Toggle Nonaktif)

| Field               | Description                                                                                                                                          |
| ------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Category**   | Security / Functional                                                                                                                                |
| **Priority**        | High                                                                                                                                                 |
| **Objective**       | Memastikan mematikan toggle **Notifikasi Email Janji Temu** tidak menghalangi pengiriman email keamanan kritis seperti **Kode OTP Reset Kata Sandi** |
| **Preconditions**   | Akun uji memiliki toggle Notifikasi Email berstatus **Nonaktif** (`email_notifications_enabled = FALSE`)                                             |
| **Test Data**       | Minta OTP di `/auth/forgot-password` untuk akun uji tersebut                                                                                         |
| **Route**           | `/auth/forgot-password` (`POST /api/v1/auth/forgot-password`)                                                                                        |
| **Expected Result** | Kode OTP 6 digit **tetap dikirim dan masuk ke email** akun uji meskipun notifikasi email janji temu sedang dimatikan                                 |
| **Referensi Kode**  | `backend/internal/service/auth_service.go` (`RequestPasswordResetOTP`)                                                                               |
| **Status QA**       | `[ ] PASS` `[ ] FAIL` `[ ] BLOCKED` — Catatan:                                                                                                       |

---

## Bagian B — Custom Reminder Menit (In-App Lonceng SSE + Email)

### Feature: Pengaturan & Eksekusi Pengingat Otomatis Sebelum Janji Temu

#### TC-NTF-06 — Pengaturan Preset Cepat & Input Bebas Menit Reminder di Halaman Profil

| Field               | Description                                                                                                                                                                                                                                                                                                                                                                                 |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Category**   | Functional / UX                                                                                                                                                                                                                                                                                                                                                                             |
| **Priority**        | Critical                                                                                                                                                                                                                                                                                                                                                                                    |
| **Objective**       | Memastikan Dosen & Mahasiswa dapat mengatur waktu pengingat menggunakan tombol preset (`Nonaktif`, `15 menit`, `30 menit`, `1 jam`, `2 jam`) maupun kolom input bebas (`5–1440` menit) dan tersimpan permanen                                                                                                                                                                               |
| **Preconditions**   | Login di `/student/profile` dan ulangi di `/lecturer/profile`                                                                                                                                                                                                                                                                                                                               |
| **Test Data**       | 1) Klik preset `15 menit`; 2) Isi input kustom `45` → klik **Simpan Menit**; 3) Klik preset `Nonaktif`                                                                                                                                                                                                                                                                                      |
| **Route**           | `/student/profile`, `/lecturer/profile` (`PUT /api/v1/users/me`)                                                                                                                                                                                                                                                                                                                            |
| **Expected Result** | 1) Default awal adalah **30 menit**. 2) Klik `15 menit` → toast sukses & teks status berubah menjadi _"diingatkan 15 menit sebelum janji temu dimulai"_. 3) Input `45` + klik **Simpan Menit** → toast sukses, refresh halaman (`F5`) → angka `45` dan status _"45 menit"_ tetap tersimpan. 4) Klik `Nonaktif` → status berubah _"Pengingat dinonaktifkan"_ (`reminder_minutes_before = 0`) |
| **Referensi Kode**  | `frontend/nextjs/src/components/common/EmailNotificationToggleCard.tsx`, `backend/internal/service/user_service.go:35-43`                                                                                                                                                                                                                                                                   |
| **Status QA**       | `[ ] PASS` `[ ] FAIL` `[ ] BLOCKED` — Catatan:                                                                                                                                                                                                                                                                                                                                              |

---

#### TC-NTF-07 — Validasi Batas Input Menit Reminder (`< 5` atau `> 1440`)

| Field               | Description                                                                                                                                     |
| ------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Category**   | Negative / Boundary                                                                                                                             |
| **Priority**        | High                                                                                                                                            |
| **Objective**       | Memastikan sistem menolak angka pengingat yang terlalu kecil (`1–4` menit, negatif) atau melebihi 24 jam (`> 1440` menit) baik di UI maupun API |
| **Preconditions**   | Login di `/student/profile` atau `/lecturer/profile`                                                                                            |
| **Test Data**       | Input `2`, `-10`, dan `2000` pada kolom **Kustom (menit)**                                                                                      |
| **Route**           | `/student/profile`, `PUT /api/v1/users/me`                                                                                                      |
| **Expected Result** | Ditolak dengan pesan error `"Waktu pengingat harus 0 (nonaktif) atau antara 5 hingga 1440 menit (24 jam)."` dan nilai lama di DB tidak berubah  |
| **Referensi Kode**  | `backend/internal/service/user_service.go:35-43`, `backend/migrations/000019_create_appointment_reminders.up.sql`                               |
| **Status QA**       | `[ ] PASS` `[ ] FAIL` `[ ] BLOCKED` — Catatan:                                                                                                  |

---

#### TC-NTF-08 — Eksekusi Otomatis Reminder (In-App SSE + Email) & Anti-Duplikat

| Field               | Description                                                                                                                                                                                                                                                                                                                                         |
| ------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Category**   | Integration / Background Scheduler                                                                                                                                                                                                                                                                                                                  |
| **Priority**        | Critical                                                                                                                                                                                                                                                                                                                                            |
| **Objective**       | Memastikan background ticker (30 detik) mengirim pengingat tepat ketika janji temu `DISETUJUI` masuk jendela waktu `start_at - reminder_minutes_before`, dan **tidak mengirim ulang** pada siklus 30 detik berikutnya                                                                                                                               |
| **Preconditions**   | Atur reminder Mahasiswa uji ke misal **120 menit** (2 jam) dan toggle email **Aktif**; buat & setujui (`DISETUJUI`) 1 janji temu yang dimulai `< 2 jam` dari waktu sekarang (misal 30–60 menit dari sekarang)                                                                                                                                       |
| **Test Data**       | Janji temu `DISETUJUI` dengan `start_at` dalam 30–60 menit ke depan; `reminder_minutes_before = 120`                                                                                                                                                                                                                                                |
| **Route**           | Background Worker → `/student/notifications` & Inbox Email                                                                                                                                                                                                                                                                                          |
| **Expected Result** | 1) Dalam **≤ 35 detik** setelah disetujui, muncul notifikasi **`Pengingat Janji Temu`** di lonceng aplikasi (SSE real-time) DAN 1 email **`Pengingat Janji Temu - Janjian Dosen Informatika`** (badge biru `PENGINGAT JADWAL`). 2) Tunggu 2 menit (4 siklus ticker) → pastikan **hanya ada tepat 1 notifikasi & 1 email** (tidak terjadi duplikasi) |
| **Referensi Kode**  | `backend/internal/repository/appointment_repo.go` (`DispatchDueAppointmentReminders`), `backend/cmd/api/main.go:171-188`                                                                                                                                                                                                                            |
| **Status QA**       | `[ ] PASS` `[ ] FAIL` `[ ] BLOCKED` — Catatan:                                                                                                                                                                                                                                                                                                      |

---

#### TC-NTF-09 — Independensi Menit Mahasiswa vs Dosen & Re-Trigger Setelah Reschedule

| Field               | Description                                                                                                                                                                                                                                                                |
| ------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Category**   | State & Lifecycle / Integration                                                                                                                                                                                                                                            |
| **Priority**        | High                                                                                                                                                                                                                                                                       |
| **Objective**       | Memastikan pengaturan menit Mahasiswa dan Dosen bekerja independen (termasuk jika salah satu `Nonaktif/0`), dan jika janji temu yang sudah pernah diingatkan di-reschedule ke waktu baru, pengingat otomatis terkirim lagi untuk waktu baru tersebut                       |
| **Preconditions**   | Mahasiswa set reminder `60 menit`, Dosen set reminder `Nonaktif (0)` pada janji temu `DISETUJUI` yang sama                                                                                                                                                                 |
| **Test Data**       | 1) Amati saat jendela 60 menit tercapai; 2) Lakukan reschedule janji temu ke waktu baru yang juga masih dalam jendela reminder                                                                                                                                             |
| **Route**           | `/student/profile`, `/lecturer/profile`, `/student/appointments/[id]`                                                                                                                                                                                                      |
| **Expected Result** | 1) Mahasiswa menerima pengingat, sedangkan Dosen (`0`) **tidak** menerima pengingat. 2) Setelah jadwal bergeser lewat reschedule (`start_at` berubah), baris baru tercatat di `appointment_reminders_sent` dengan `scheduled_start_at` baru dan pengingat terkirim kembali |
| **Referensi Kode**  | `backend/migrations/000019_create_appointment_reminders.up.sql` (`UNIQUE (appointment_id, user_id, scheduled_start_at)`)                                                                                                                                                   |
| **Status QA**       | `[ ] PASS` `[ ] FAIL` `[ ] BLOCKED` — Catatan:                                                                                                                                                                                                                             |

---

## Bagian C — Broadcast Notifikasi Sistem oleh Admin (`/admin/notifications`)

### Feature: Pengumuman Sistem Real-Time (4 Mode Target, In-App SSE Only)

#### TC-NTF-10 — Broadcast Mode `ALL` (Template Maintenance `warning`) → Real-Time Tanpa Email

| Field               | Description                                                                                                                                                                                                                                                                                                                                                                                              |
| ------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Category**   | Functional / Integration (SSE Broadcast)                                                                                                                                                                                                                                                                                                                                                                 |
| **Priority**        | Critical                                                                                                                                                                                                                                                                                                                                                                                                 |
| **Objective**       | Memastikan Admin dapat mengirim notifikasi sistem ke seluruh pengguna aktif (Dosen & Mahasiswa) memakai template Maintenance, tampil real-time di lonceng & tab `Sistem`, tercatat di Riwayat Broadcast, dan **tidak mengirim email**                                                                                                                                                                    |
| **Preconditions**   | Jendela 1: Login **Admin** buka menu **Notifikasi Sistem** (`/admin/notifications`); Jendela 2 (Incognito): Login **Mahasiswa** (atau Dosen) standby di dashboard                                                                                                                                                                                                                                        |
| **Test Data**       | Klik **`+ Gunakan Template Maintenance`**, pilih Target **`Semua Pengguna`**, klik **`Kirim Notifikasi Sistem`**                                                                                                                                                                                                                                                                                         |
| **Route**           | `/admin/notifications` → `/student/notifications` & `/lecturer/notifications`                                                                                                                                                                                                                                                                                                                            |
| **Expected Result** | 1) Muncul toast sukses `"Notifikasi sistem berhasil dikirim ke X pengguna."` dan entri baru langsung tampil di panel **Riwayat Broadcast Terkirim** dengan badge kuning `Peringatan` & `Semua Pengguna • X penerima`. 2) Di Jendela 2, badge lonceng bertambah real-time (≤ 4 detik) dan pesan maintenance tampil di tab **`Sistem`**. 3) **Tidak ada email** masuk ke Inbox pengguna (zero kuota Gmail) |
| **Referensi Kode**  | `frontend/nextjs/src/app/admin/notifications/page.tsx`, `backend/internal/repository/notification_repo.go` (`CreateSystemBroadcast`), `backend/internal/notification/whatsapp_stub.go:121`                                                                                                                                                                                                               |
| **Status QA**       | `[ ] PASS` `[ ] FAIL` `[ ] BLOCKED` — Catatan:                                                                                                                                                                                                                                                                                                                                                           |

---

#### TC-NTF-11 — Broadcast Mode `ROLE` (`DOSEN` / `MAHASISWA`) & Mode `PRODI`

| Field               | Description                                                                                                                                                                                                                                                                                              |
| ------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Category**   | Functional / Filtering                                                                                                                                                                                                                                                                                   |
| **Priority**        | High                                                                                                                                                                                                                                                                                                     |
| **Objective**       | Memastikan filter target berdasarkan Peran (`ROLE`) dan Program Studi (`PRODI`) hanya mengirim notifikasi ke pengguna yang cocok dengan filter tersebut                                                                                                                                                  |
| **Preconditions**   | Login Admin di `/admin/notifications`                                                                                                                                                                                                                                                                    |
| **Test Data**       | 1) Mode **`Filter Peran`** → pilih **`Semua Dosen`**, judul `[QA-BC] Khusus Dosen`; 2) Mode **`Filter Prodi`** → pilih **`Informatika`**, judul `[QA-BC] Info Prodi IF`                                                                                                                                  |
| **Route**           | `/admin/notifications`                                                                                                                                                                                                                                                                                   |
| **Expected Result** | 1) Pada kirim ke `Semua Dosen`, perkiraan penerima hanya menghitung jumlah Dosen aktif; akun Dosen menerima `[QA-BC] Khusus Dosen` di tab `Sistem`, sedangkan akun Mahasiswa **tidak** menerimanya. 2) Pada kirim ke `Prodi: Informatika`, hanya pengguna aktif ber-prodi `Informatika` yang menerimanya |
| **Referensi Kode**  | `backend/internal/repository/notification_repo.go:111-165` (`ResolveBroadcastRecipients`)                                                                                                                                                                                                                |
| **Status QA**       | `[ ] PASS` `[ ] FAIL` `[ ] BLOCKED` — Catatan:                                                                                                                                                                                                                                                           |

---

#### TC-NTF-12 — Broadcast Mode `USERS` (Pencarian & Multi-Select Pengguna Spesifik)

| Field               | Description                                                                                                                                                                                              |
| ------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Category**   | Functional / Filtering                                                                                                                                                                                   |
| **Priority**        | High                                                                                                                                                                                                     |
| **Objective**       | Memastikan Admin dapat mencari pengguna berdasarkan nama/NIM/NIP, memilih beberapa akun tertentu (chip terpilih tampil & bisa dihapus dengan ikon `X`), dan hanya akun terpilih yang menerima notifikasi |
| **Preconditions**   | Login Admin di `/admin/notifications`                                                                                                                                                                    |
| **Test Data**       | Pilih target **`Pengguna Spesifik`**, cari nama 1 Mahasiswa uji (centang), pastikan perkiraan penerima tertulis `1 pengguna aktif`, kirim judul `[QA-BC] Pesan Personal`                                 |
| **Route**           | `/admin/notifications`                                                                                                                                                                                   |
| **Expected Result** | Hanya Mahasiswa terpilih yang menerima `[QA-BC] Pesan Personal` di tab `Sistem`; Mahasiswa lain & Dosen **tidak** menerimanya; Riwayat Broadcast mencatat `Pengguna Spesifik • 1 penerima`               |
| **Referensi Kode**  | `frontend/nextjs/src/app/admin/notifications/page.tsx:387-468`, `backend/internal/repository/notification_repo.go`                                                                                       |
| **Status QA**       | `[ ] PASS` `[ ] FAIL` `[ ] BLOCKED` — Catatan:                                                                                                                                                           |

---

#### TC-NTF-13 — UX Tab `Sistem` di `/student/notifications` & `/lecturer/notifications`

| Field               | Description                                                                                                                                                                                                                                                                                                                                          |
| ------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Category**   | UX / Navigation                                                                                                                                                                                                                                                                                                                                      |
| **Priority**        | Medium                                                                                                                                                                                                                                                                                                                                               |
| **Objective**       | Memastikan notifikasi broadcast bertipe `info` maupun `warning` tampil di tab **`Sistem`** (tidak salah masuk ke tab **`Janji Temu`**), dan saat diklik hanya menandai terbaca (_mark as read_) tanpa redirect paksa ke halaman janji temu                                                                                                           |
| **Preconditions**   | Akun Mahasiswa & Dosen sudah menerima minimal 1 broadcast `info` dan 1 broadcast `warning` (maintenance)                                                                                                                                                                                                                                             |
| **Test Data**       | Buka `/student/notifications` (dan `/lecturer/notifications`) → klik tab **`Sistem`** → klik kartu notifikasi maintenance yang belum dibaca                                                                                                                                                                                                          |
| **Route**           | `/student/notifications`, `/lecturer/notifications`                                                                                                                                                                                                                                                                                                  |
| **Expected Result** | 1) Kedua notifikasi broadcast (`info` & `warning`) tampil di tab **`Sistem`** dan **tidak** tampil di tab **`Janji Temu`**. 2) Saat kartu diklik, status berubah menjadi sudah dibaca (`is_read = true`, badge unread berkurang 1) dan halaman **tetap di Pusat Notifikasi** (tidak terpental ke `/student/appointments` atau `/lecturer/dashboard`) |
| **Referensi Kode**  | `frontend/nextjs/src/app/student/notifications/page.tsx:69-130`, `frontend/nextjs/src/app/lecturer/notifications/page.tsx:70-145`                                                                                                                                                                                                                    |
| **Status QA**       | `[ ] PASS` `[ ] FAIL` `[ ] BLOCKED` — Catatan:                                                                                                                                                                                                                                                                                                       |

---

#### TC-NTF-14 — Validasi Input, Proteksi RBAC Non-Admin (`403`), & Audit Log Broadcast

| Field               | Description                                                                                                                                                                                                                                                                                                              |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Test Category**   | Security / Negative / Audit                                                                                                                                                                                                                                                                                              |
| **Priority**        | High                                                                                                                                                                                                                                                                                                                     |
| **Objective**       | Memastikan form broadcast menolak input kosong/0 penerima, endpoint `/api/v1/admin/notifications/broadcast` menolak token Mahasiswa/Dosen (`403 Forbidden`), dan setiap broadcast sukses tercatat di `audit_logs`                                                                                                        |
| **Preconditions**   | Siapkan token Mahasiswa (`$TOKEN_MHS_A`) dan login Admin untuk memeriksa `/admin/audit-logs`                                                                                                                                                                                                                             |
| **Test Data**       | 1) Mode `USERS` tanpa memilih 1 pun pengguna → tombol kirim non-aktif / ditolak; 2) Tembak `POST /api/v1/admin/notifications/broadcast` memakai `$TOKEN_MHS_A`                                                                                                                                                           |
| **Route**           | `POST /api/v1/admin/notifications/broadcast`, `/admin/audit-logs`                                                                                                                                                                                                                                                        |
| **Expected Result** | 1) Tombol **`Kirim Notifikasi Sistem`** otomatis _disabled_ saat perkiraan penerima = `0`. 2) Request dengan token Mahasiswa/Dosen ditolak **`403 Forbidden`**. 3) Di `/admin/audit-logs`, tercatat aksi **`SYSTEM_NOTIFICATION_BROADCAST`** dengan metadata `title`, `notif_type`, `target_mode`, dan `recipient_count` |
| **Referensi Kode**  | `backend/internal/handler/notification_handler.go:100-140`, `backend/cmd/api/main.go:303-305`                                                                                                                                                                                                                            |
| **Status QA**       | `[ ] PASS` `[ ] FAIL` `[ ] BLOCKED` — Catatan:                                                                                                                                                                                                                                                                           |

**Verifikasi API (`curl` Proteksi Non-Admin):**

```bash
curl -sS -i -X POST "https://38-9-47-207.sslip.io/api/v1/admin/notifications/broadcast" \
  -H "Authorization: Bearer $TOKEN_MHS_A" \
  -H "Content-Type: application/json" \
  -d '{"title":"Uji","message":"Tes","type":"info","target_mode":"ALL"}' | head -n 12
```

- **PASS jika**: Mengembalikan `HTTP/2 403`.
