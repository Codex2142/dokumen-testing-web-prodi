# 06 — Cross-Role & Critical Business Flow Test Cases

Dokumen ini menguji skenario **end-to-end lintas peran** (`ADMIN` ↔ `DOSEN` ↔ `MAHASISWA`) yang merepresentasikan **Critical Business Flow** utama aplikasi. Disarankan menggunakan 2 jendela browser (Normal + Incognito) untuk mengamati sinkronisasi status dan notifikasi SSE secara real-time.

---

## Skenario Lintas Role

### Feature: Transparansi Slot & Auto-Conflict Resolution

#### TC-XROLE-01 — Transparansi Slot Mahasiswa A pada POV Mahasiswa B & Pencegahan Double-Booking
| Field | Description |
|---|---|
| **Test Category** | Integration / Cross-Role |
| **Priority** | Critical |
| **Objective** | Memastikan slot yang diajukan Akbar (`MENUNGGU_PERSETUJUAN`) langsung terlihat di kalender `Jadwal Dosen` Bima dan mencegah Bima mengajukan slot yang sama |
| **Preconditions** | Akbar (`1203220154`) sudah mengajukan Senin pekan depan `10:00 - 10:30` ke Dr. Hendra |
| **Test Data** | Jendela 2 (Incognito): Login Bima (`1203220012`), cek Dr. Hendra Senin `10:00 - 10:30` |
| **Route** | `/student/dashboard` (Mode `Jadwal Dosen`) |
| **Expected Result** | Kalender Bima menampilkan balok kuning atas nama `Akbar Muhammad Sadat (1203220154)`; modal booking Bima menampilkan peringatan jadwal lain & submit pada jam `10:00 - 10:30` ditolak backend (`"Slot waktu telah diajukan terlebih dahulu oleh mahasiswa lain..."`) |
| **Referensi Kode** | `backend/internal/service/schedule_service.go:207-242`, `backend/internal/service/availability_service.go:326-346` |
| **Status QA** | `[ ] PASS` `[❌] FAIL` `[ ] BLOCKED` — Catatan: Jadwal yang diajukan akbar terlihat pada kalender dashboard Bima. Sistem meloloskan Bima saat mengajukan pertemuan dengan jam, tanggal, dan waktu yang sama dengan Akbar, sistem tidak menolak dan tidak memberikan pesan bentrok |

**Manual Testing Instructions**:
1. Login Akbar di browser normal, ajukan janji ke Dr. Hendra Senin `10:00 - 10:30`.
2. Login Bima di Incognito, buka `/student/dashboard`, pilih mode `Jadwal Dosen` → `Dr. Eng. Hendra Kurniawan`.
3. Periksa hari Senin pukul `10:00 - 10:30`, pastikan balok Akbar tampil.
4. Coba buka `BookingBottomSheet` dan ajukan jam `10:00 - 10:30` pada hari yang sama → pastikan ditolak dengan pesan bentrok informatif.

---

### Feature: Real-Time SSE Notification & Approval Lifecycle

#### TC-XROLE-02 — Mahasiswa Ajukan Janjian → Dosen Terima Notifikasi SSE Real-Time → Dosen Approve → Mahasiswa Terima Notifikasi
| Field | Description |
|---|---|
| **Test Category** | Integration / Critical Business Flow / UX |
| **Priority** | Critical |
| **Objective** | Memverifikasi alur penuh Outbox Worker (`2s` poll) → SSE `/notifications/stream` → auto-reload dashboard Dosen → persetujuan → notifikasi balik ke Mahasiswa |
| **Preconditions** | Browser 1: Login Dr. Hendra buka `/lecturer/dashboard`; Browser 2 (Incognito): Login Akbar buka `/student/dashboard` |
| **Test Data** | Booking Rabu pekan depan `09:00 - 09:30`, Topik: `[UAT-2026] Uji SSE Real-Time` |
| **Route** | `/student/dashboard` ↔ `/lecturer/dashboard` |
| **Expected Result** | Dalam ≤ 4 detik setelah Akbar submit, browser Dr. Hendra memunculkan toast `"Pemberitahuan: Ada pembaruan permohonan janji temu masuk."` dan kartu baru muncul tanpa refresh; setelah Dr. Hendra klik `Setujui`, status di sisi Akbar berubah menjadi `DISETUJUI` dan notifikasi `"Janji Temu Disetujui!"` masuk di `/student/notifications` |
| **Referensi Kode** | `backend/internal/scheduler/outbox_worker.go`, `backend/internal/notification/sse_handler.go`, `frontend/nextjs/src/app/lecturer/dashboard/page.tsx:130-152` |
| **Status QA** | `[✅] PASS` `[ ] FAIL` `[ ] BLOCKED` — Catatan: Sistem secara realtime langsung mengupdate bahwa ada janji temu baru yang dibuat oleh mahasiswa |

**Manual Testing Instructions**:
1. Biarkan Dr. Hendra standby di `/lecturer/dashboard` (jangan refresh manual).
2. Di Incognito (Akbar), buat janji temu Rabu `09:00 - 09:30` topik `[UAT-2026] Uji SSE Real-Time`.
3. Amati layar Dr. Hendra dalam 5 detik (toast + kemunculan kartu baru + badge lonceng notifikasi).
4. Klik `Setujui` di layar Dr. Hendra, lalu cek status & notifikasi di layar Akbar.

**Database Verification**:
```sql
SELECT event_type, status, retry_count, processed_at FROM public.notification_outbox ORDER BY created_at DESC LIMIT 5;
```

---

### Feature: Reschedule Interaksi Mahasiswa ↔ Dosen

#### TC-XROLE-03 — Mahasiswa Ajukan Reschedule → Dosen Terima/Tolak Reschedule → Verifikasi Update Jadwal
| Field | Description |
|---|---|
| **Test Category** | Integration / State & Lifecycle |
| **Priority** | High |
| **Objective** | Memastikan permintaan reschedule dari Mahasiswa dapat direspon oleh Dosen (`ACCEPT` atau `REJECT`), memperbarui `start_at`/`end_at` jika diterima, dan mencegah pengaju merespon permintaannya sendiri |
| **Preconditions** | Janji temu `[UAT-2026] Uji SSE Real-Time` berstatus `DISETUJUI` |
| **Test Data** | Usulan waktu baru: Rabu `10:30 - 11:00`, Alasan: `[UAT-2026] Geser jam bimbingan` |
| **Route** | `/student/appointments/[id]` ↔ `PATCH /api/v1/reschedule-requests/:id/respond` |
| **Expected Result** | Permohonan `PENDING` tersimpan; jika direspon `ACCEPT` oleh Dr. Hendra, status request menjadi `ACCEPTED` dan jam janji temu bergeser ke `10:30 - 11:00`; jika Akbar mencoba merespon request sendiri via API → ditolak `ErrSameUserResponse` |
| **Referensi Kode** | `backend/internal/repository/reschedule_repo.go:164-250`, `backend/internal/service/reschedule_service.go:128-140` |
| **Status QA** | `[ ] PASS` `[❌] FAIL` `[ ] BLOCKED` — Catatan: saat dosen menyetujui, pada bagian akbar dan dosen saat di cek pada tanggal schedule tidak terdapat update sama sekali |

**Manual Testing Instructions**:
1. Di sesi Akbar (`/student/appointments/[id]`), ajukan reschedule ke Rabu `10:30 - 11:00`.
2. Periksa notifikasi `"Permohonan Reschedule Baru"` pada sesi Dr. Hendra (`/lecturer/notifications`).
3. Respon permohonan reschedule tersebut (`ACCEPT`) dan verifikasi jam pertemuan pada kalender kedua pihak berubah ke `10:30 - 11:00`.
   > **NEEDS VERIFICATION (UI Dosen)**: Periksa apakah modal/tombol respon reschedule (`ACCEPT`/`REJECT`) tersedia langsung di UI `/lecturer/dashboard` atau `/lecturer/appointments`, atau hanya tersedia di backend API `PATCH /api/v1/reschedule-requests/:id/respond`. Catat hasil pengamatan di kolom Catatan QA!

---

### Feature: Siklus Lengkap Masukan Pengguna (Mahasiswa/Dosen → Admin → Mahasiswa/Dosen)

#### TC-XROLE-04 — Pengiriman Masukan + Gambar → Evaluasi Admin → Pembaruan Status & Catatan di Sisi Pengirim
| Field | Description |
|---|---|
| **Test Category** | Integration / Critical Business Flow |
| **Priority** | Critical |
| **Objective** | Memastikan feedback dan lampiran media yang dikirim Mahasiswa tampil utuh di panel Admin, dan perubahan status + `admin_notes` langsung tampil di tab Riwayat Mahasiswa |
| **Preconditions** | Database & storage VPS bersih (Phase 1) |
| **Test Data** | Feedback Akbar (`TC-MHS-09`), Evaluasi Admin: status `SELESAI`, catatan: `Terima kasih Akbar, sudah kami perbaiki.` |
| **Route** | `/student/feedback` ↔ `/admin/feedbacks` |
| **Expected Result** | Admin melihat judul, konten rich HTML (`LMSRichRenderer`), dan gambar dari `/storage/uploads/...`; setelah Admin menyimpan evaluasi, Akbar membuka `/student/feedback` → tab Riwayat dan melihat badge hijau `Selesai Dibenahi` beserta catatan admin |
| **Referensi Kode** | `frontend/nextjs/src/app/student/feedback/page.tsx`, `frontend/nextjs/src/app/admin/feedbacks/page.tsx`, `AdminFeedbackDetailModal.tsx` |
| **Status QA** | `[✅] PASS` `[ ] FAIL` `[ ] BLOCKED` — Catatan: Feedback dari akbar terkirim, kemudian admin merubah status menjadi "selesai dibenahi", kemudian di user akbar, riwayat masukan sudah ter-update sesuai dengan yang di update oleh admin |

**Manual Testing Instructions**:
1. Login Akbar → kirim feedback berlampiran 1 screenshot PNG di `/student/feedback`.
2. Login Admin → buka `/admin/feedbacks` → klik laporan Akbar → pastikan screenshot tampil jelas di modal & Lightbox.
3. Ubah status menjadi `Selesai Dibenahi`, isi catatan admin, dan klik simpan.
4. Kembali ke sesi Akbar → buka tab Riwayat di `/student/feedback` → pastikan status & catatan admin tampil akurat.

---

### Feature: Dampak Konfigurasi Admin ke Operasional Mahasiswa & Dosen

#### TC-XROLE-05 — Admin Ubah Aturan Sistem & Kalender Libur → Verifikasi Dampak Langsung di Booking Mahasiswa
| Field | Description |
|---|---|
| **Test Category** | Integration / Critical Business Flow |
| **Priority** | High |
| **Objective** | Memastikan perubahan `appointment_rules` (contoh `max_minutes` diubah dari 60 menjadi 45) dan penambahan hari libur langsung berlaku di booking engine Mahasiswa tanpa restart server |
| **Preconditions** | Login Admin & Mahasiswa di 2 jendela |
| **Test Data** | Admin ubah `max_minutes = 45`; Mahasiswa coba booking 60 menit |
| **Route** | `/admin/system-settings` ↔ `/student/dashboard` |
| **Expected Result** | Pengajuan 60 menit langsung ditolak `"durasi pertemuan maksimal 45 menit"`; setelah selesai uji, kembalikan `max_minutes` ke `60` |
| **Referensi Kode** | `backend/internal/service/settings_service.go`, `backend/internal/service/appointment_service.go:70-82` |
| **Status QA** | `[✅] PASS` `[ ] FAIL` `[ ] BLOCKED` — Catatan: Sistem berhasil menolak saat oleh admin diubah durasi maksimalnya |

**Manual Testing Instructions**:
1. Di sesi Admin (`/admin/system-settings`), ubah `Durasi Maksimal` ke `45` lalu simpan.
2. Di sesi Akbar (`/student/dashboard`), coba ajukan janji temu berdurasi 60 menit (`09:00 - 10:00`) → pastikan ditolak maksimal 45 menit.
3. Kembalikan `Durasi Maksimal` ke `60` pada sesi Admin.
