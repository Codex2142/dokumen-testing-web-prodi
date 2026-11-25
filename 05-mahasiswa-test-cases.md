# 05 — Mahasiswa Test Cases (`MAHASISWA`)

**Akun Uji Utama**: `1203220154` (`Akbar Muhammad Sadat`) / Password: `password123`  
**Akun Uji Sekunder**: `1203220012` (`Bima Pratama Putra`) / Password: `password123`

---

## Role: MAHASISWA

### Feature: Agenda & Kalender (`/student/dashboard`) + Booking (`BookingBottomSheet`)

#### TC-MHS-01 — Pengajuan Janji Temu Normal (Happy Path)
| Field | Description |
|---|---|
| **Test Category** | Functional / Integration / State & Lifecycle |
| **Priority** | Critical |
| **Objective** | Memastikan Mahasiswa dapat memilih dosen, tanggal, jam, kategori, dan mengirim pengajuan janji temu baru |
| **Preconditions** | Login sebagai Akbar (`1203220154`), Dr. Hendra tersedia pada Senin pekan depan pukul `10:00 - 10:30` |
| **Test Data** | Dosen: `Dr. Eng. Hendra Kurniawan`, Tanggal: Senin pekan depan, Jam: `10:00 - 10:30`, Kategori: `Bimbingan Skripsi / Tugas Akhir`, Topik: `[UAT-2026] Bimbingan Skripsi Bab 1`, Lokasi: `Ruang IF-204` |
| **Route** | `/student/dashboard` |
| **Expected Result** | Toast hijau sukses muncul, `BookingBottomSheet` tertutup, janji temu tersimpan dengan status `MENUNGGU_PERSETUJUAN`, tampil di mode `Jadwal Saya` dan di `/student/appointments` tab `Menunggu` |
| **Referensi Kode** | `frontend/nextjs/src/components/booking/BookingBottomSheet.tsx`, `backend/internal/service/appointment_service.go:56-194`, `POST /api/v1/appointments` |
| **Status QA** | `[✅] PASS` `[ ] FAIL` `[ ] BLOCKED` — Catatan: Sistem berhasil menyimpan jadwal bimbingan, dan kalender juga menampilkan jadwal yang dibuat oleh mahasiswa. Sistem juga menampilkan alert "Berhasil" |

**Manual Testing Instructions**:
1. Buka `Agenda & Kalender` (`/student/dashboard`).
2. Klik tombol `+` (FAB) atau klik slot kosong pada pukul `10:00` hari Senin pekan depan.
3. Pilih Dosen `Dr. Eng. Hendra Kurniawan`, Kategori `Bimbingan Skripsi / Tugas Akhir`, Tanggal Senin pekan depan, Jam `10:00` – `10:30`.
4. Isi Topik `[UAT-2026] Bimbingan Skripsi Bab 1` dan Lokasi `Ruang IF-204`.
5. Klik kirim pengajuan dan amati toast serta kalender.

**Database Verification**:
```sql
SELECT id, title, category, start_at, end_at, status FROM public.appointments ORDER BY created_at DESC LIMIT 3;
```

---

#### TC-MHS-02 — Validasi Boundary Durasi (< 15 Menit / > 60 Menit) & Batas Tanggal (Masa Lalu / > H+30)
| Field | Description |
|---|---|
| **Test Category** | Boundary / Negative |
| **Priority** | High |
| **Objective** | Memastikan `BookingBottomSheet` & `AppointmentService` menolak durasi di luar `15–60 menit`, waktu masa lalu, atau lebih dari `H+30` |
| **Preconditions** | `system_settings` menggunakan aturan `min=15, max=60, max_advance=30` |
| **Test Data** | Case A: Jam sudah lewat hari ini; Case B: Durasi `10:00 - 11:15` (75 menit); Case C: Tanggal `H+35` |
| **Route** | `/student/dashboard` |
| **Expected Result** | Case A ditolak `"waktu janji temu tidak boleh di masa lalu"`; Case B ditolak `"durasi pertemuan maksimal 60 menit"`; Case C ditolak `"pemesanan maksimal H+30 ke depan"` |
| **Referensi Kode** | `backend/internal/service/appointment_service.go:76-96` |
| **Status QA** | `[✅] PASS` `[ ] FAIL` `[ ] BLOCKED` — Catatan: Case A sistem berhasil menolak dan menampilkan alert bahwa jam sudah terlewat, Case B Sistem mengunci tombol simpan karena durasi pertemuan melebihi batas maksimal, Case C sistem menolak dan menampilkan alert "Batas H+30 hari" |

**Manual Testing Instructions**:
1. Buka `BookingBottomSheet`.
2. Uji Case A: pilih hari ini dan jam yang sudah terlewat → submit → amati penolakan.
3. Uji Case B: pilih Senin pekan depan jam `10:00` sampai `11:15` (75 menit) → submit → amati penolakan durasi.
4. Uji Case C: pilih tanggal 35 hari dari hari ini → submit → amati penolakan batas H+30.

---

#### TC-MHS-03 — Penolakan Booking pada Hari Libur, Jadwal Kuliah, & Jadwal Terblokir Dosen
| Field | Description |
|---|---|
| **Test Category** | Negative / Integration |
| **Priority** | Critical |
| **Objective** | Memastikan engine ketersediaan menolak booking pada hari libur (`TC-ADM-08`), jam kuliah (`TC-DSN-01`), dan jam blokir (`TC-DSN-03`) dengan pesan edukatif |
| **Preconditions** | Data uji `TC-ADM-08` (libur Jumat), `TC-DSN-01` (kuliah Senin 13:00–15:30), `TC-DSN-03` (blokir Kamis 09:00–12:00) sudah dibuat |
| **Test Data** | Case A: Jumat libur `10:00-10:30`; Case B: Senin `13:30-14:00` (Dr. Hendra); Case C: Kamis `10:00-10:30` (Dr. Hendra) |
| **Route** | `/student/dashboard` |
| **Expected Result** | Case A ditolak `"Tanggal yang dipilih merupakan hari libur kalender akademik"`; Case B ditolak `"Dosen memiliki jadwal mengajar '[UAT-2026] Arsitektur Komputasi Awan'..."`; Case C ditolak `"Dosen memiliki agenda terblokir: '[UAT-2026] Rapat Senat Kurikulum'..."` |
| **Referensi Kode** | `backend/internal/service/availability_service.go:248-324` |
| **Status QA** | `[✅] PASS` `[ ] FAIL` `[ ] BLOCKED` — Catatan: Sistem berhasil menampilkan alert bahwa jadwal bertabrakan dengan jadwal mata kuliah dan jadwal blokir |

**Manual Testing Instructions**:
1. Di `/student/dashboard`, beralih ke mode `Jadwal Dosen`, pilih `Dr. Eng. Hendra Kurniawan`, amati balok kuliah (Senin 13:00–15:30) dan balok blokir (Kamis 09:00–12:00).
2. Buka `BookingBottomSheet` dan coba submit masing-masing Case A, Case B, dan Case C.
3. Catat pesan penolakan yang tampil di layar.

---

#### TC-MHS-04 — Validasi Buffer 10 Menit Antar Janji Temu
| Field | Description |
|---|---|
| **Test Category** | Boundary / Negative |
| **Priority** | High |
| **Objective** | Memastikan pengajuan baru yang menempel langsung tanpa jeda `buffer_minutes = 10` ditolak oleh `CheckSlotAvailability` |
| **Preconditions** | Janji temu Senin `10:00 - 10:30` (Dr. Hendra) sudah ada |
| **Test Data** | Pengajuan baru Senin `10:30 - 11:00` (jeda 0 menit) vs `10:45 - 11:15` (jeda 15 menit >= 10 menit) |
| **Route** | `/student/dashboard` |
| **Expected Result** | `10:30 - 11:00` ditolak bentrok (masuk rentang buffer `09:50–10:40`); `10:45 - 11:15` berhasil diajukan |
| **Referensi Kode** | `backend/internal/service/availability_service.go:329-346` |
| **Status QA** | `[ ] PASS` `[❌] FAIL` `[ ] BLOCKED` — Catatan: Sistem tetap meloloskan sehingga pada kalender jadwal janjian mengalami duplikasi |

**Manual Testing Instructions**:
1. Buka `BookingBottomSheet` untuk Dr. Hendra pada Senin pekan depan.
2. Coba submit jam `10:30 - 11:00` → pastikan ditolak karena bertabrakan dengan buffer janji `10:00 - 10:30`.
3. Ubah jam ke `10:45 - 11:15` → pastikan berhasil.

---

### Feature: Janjian Saya (`/student/appointments` & `/student/appointments/[id]`)

#### TC-MHS-05 — Tab Filter Status, Pencarian, & Date Separator di `/student/appointments`
| Field | Description |
|---|---|
| **Test Category** | Functional / UX |
| **Priority** | High |
| **Objective** | Memastikan seluruh tab status (`ALL`, `MENUNGGU_PERSETUJUAN`, `DISETUJUI`, `SELESAI`, `BENTROK`, `DITOLAK`, `DIBATALKAN`), kotak pencarian, dan Date Separator bekerja akurat |
| **Preconditions** | Login sebagai Akbar dengan minimal 2 janji temu |
| **Test Data** | Pencarian berdasarkan judul topik, nama dosen, atau ruangan |
| **Route** | `/student/appointments` |
| **Expected Result** | Badge angka di tiap tab cocok dengan jumlah kartu; kartu dikelompokkan dengan pemisah tanggal; pencarian menyaring secara real-time; jika hasil kosong muncul Empty State |
| **Referensi Kode** | `frontend/nextjs/src/app/student/appointments/page.tsx:83-120` |
| **Status QA** | `[✅] PASS` `[ ] FAIL` `[ ] BLOCKED` — Catatan: Sistem berhasil menampilkan dengan benar (cocok), kemudian ketika kata kunci tidak ada sistem menampilkan empty state |

**Manual Testing Instructions**:
1. Buka menu `Janjian Saya` (`/student/appointments`).
2. Klik tiap tab status dan cocokan angka counter dengan jumlah kartu yang tampil.
3. Ketik kata kunci yang tidak ada (`zzzzz`) → pastikan Empty State informatif tampil.

---

#### TC-MHS-06 — Pembatalan Janji Temu Mandiri oleh Mahasiswa (Alasan Wajib & Notifikasi Ganda)
| Field | Description |
|---|---|
| **Test Category** | State & Lifecycle / Negative |
| **Priority** | Critical |
| **Objective** | Memastikan Mahasiswa dapat membatalkan janji temu masa depan dengan alasan wajib, dan menerima notifikasi konfirmasi pembatalan |
| **Preconditions** | Akbar memiliki 1 janji temu masa depan berstatus `MENUNGGU_PERSETUJUAN` atau `DISETUJUI` |
| **Test Data** | Alasan: `[UAT-2026] Ada jadwal responsi praktikum mendadak` |
| **Route** | `/student/appointments/[id]` |
| **Expected Result** | Status berubah menjadi `DIBATALKAN`, kartu tampil di tab `Dibatalkan` dengan alasan tersebut, dan muncul notifikasi `"Pembatalan Janji Temu Berhasil"` di `/student/notifications` |
| **Referensi Kode** | `frontend/nextjs/src/app/student/appointments/[id]/page.tsx`, `backend/internal/repository/appointment_repo.go:594-724`, `PATCH /api/v1/appointments/:id/cancel` |
| **Status QA** | `[✅] PASS` `[ ] FAIL` `[ ] BLOCKED` — Catatan: Janji temu berhasil dibatalkan, sistem juga memberi konfimasi batal dengan form alasan sebelum pembatalan dilakukan, |

**Manual Testing Instructions**:
1. Buka detail janji temu di `/student/appointments/[id]`.
2. Klik tombol `Batalkan Janji Temu`.
3. Isi alasan `[UAT-2026] Ada jadwal responsi praktikum mendadak` dan klik konfirmasi.
4. Periksa perubahan status menjadi `Dibatalkan` dan buka `Pusat Notifikasi` (`/student/notifications`).

---

#### TC-MHS-07 — Pengajuan Reschedule dari Halaman Detail (`/student/appointments/[id]`)
| Field | Description |
|---|---|
| **Test Category** | Functional / Negative / Boundary |
| **Priority** | High |
| **Objective** | Memastikan Mahasiswa dapat mengajukan reschedule pada janji temu `MENUNGGU_PERSETUJUAN`/`DISETUJUI` dengan slot dari `GET /availabilities/slots`, dan dicegah membuat 2 request `PENDING` sekaligus |
| **Preconditions** | Akbar memiliki janji temu berstatus `DISETUJUI` di masa depan |
| **Test Data** | Tanggal baru: Selasa pekan depan, pilih 1 slot tersedia, Alasan: `[UAT-2026] Penyesuaian jadwal praktikum` |
| **Route** | `/student/appointments/[id]` |
| **Expected Result** | Request tersimpan dengan status `PENDING` di `appointment_reschedule_requests`; percobaan mengajukan reschedule kedua sebelum yang pertama direspon ditolak (`ErrPendingRescheduleExists`) |
| **Referensi Kode** | `frontend/nextjs/src/app/student/appointments/[id]/page.tsx:131-160`, `backend/internal/service/reschedule_service.go:51-108`, `POST /api/v1/appointments/:id/reschedule` |
| **Status QA** | `[ ] PASS` `[❌] FAIL` `[ ] BLOCKED` — Catatan: Re-Schedule tidak dapat dilaksanakan di tanggal berapapun, sistem selalu menampilkan alert **"Tidak ada slot ketersediaan pada tanggal ini. Silakan pilih tanggal lain."** padahal dosen tidak terdapat kegiatan apapun,  |

**Manual Testing Instructions**:
1. Buka detail janji temu berstatus `Disetujui`.
2. Klik tombol Reschedule, pilih tanggal baru Selasa pekan depan, tunggu daftar slot tersedia termuat, pilih 1 slot, masukkan alasan, lalu kirim.
3. Verifikasi riwayat permohonan reschedule tampil berstatus `PENDING`.
4. Coba ajukan reschedule sekali lagi pada janji yang sama → pastikan ditolak.

---

#### TC-MHS-08 — Proteksi Anti-Pembatalan Janji Temu `SELESAI` / Lewat Jadwal
| Field | Description |
|---|---|
| **Test Category** | State & Lifecycle / Negative / Database Trigger |
| **Priority** | Critical |
| **Objective** | Memastikan janji temu yang waktu selesainya sudah lewat (`end_at < NOW()`) otomatis berstatus `SELESAI` dan tidak dapat dibatalkan baik dari UI maupun API/DB |
| **Preconditions** | Terdapat janji temu `DISETUJUI` yang telah melewati `end_at` (atau disimulasikan lewat SQL QA setelah verifikasi) |
| **Test Data** | Appointment berstatus `SELESAI` |
| **Route** | `/student/appointments` → Tab `Selesai` → `/student/appointments/[id]` |
| **Expected Result** | Tombol batalkan/reschedule tidak aktif di UI; jika dipaksa via API `PATCH /appointments/:id/cancel` ditolak backend (`ErrCannotCancelAppointment`) dan dilindungi trigger DB `trg_prevent_cancellation` |
| **Referensi Kode** | `backend/migrations/000014_enforce_appointment_completed_and_anti_cancellation.up.sql`, `backend/internal/repository/appointment_repo.go:245-256,632` |
| **Status QA** | `[✅] PASS` `[ ] FAIL` `[ ] BLOCKED` — Catatan: Pada janji temu yang sudah selesai tidak didapati button "Batalkan Janji Temu" |

**Manual Testing Instructions**:
1. Buka tab `Selesai` di `/student/appointments` dan klik detail janji temu yang sudah selesai.
2. Pastikan tidak ada tombol `Batalkan Janji Temu` yang aktif.
3. (Verifikasi Backend) Kirim `PATCH /api/v1/appointments/<id>/cancel` → pastikan gagal.

---

### Feature: Masukan Pengguna (`/student/feedback`), LMS Editor, & Direct Storage

#### TC-MHS-09 — Pengiriman Masukan dengan `LMSRichEditor` + `MediaDropzone` (WebP & Video Direct Upload)
| Field | Description |
|---|---|
| **Test Category** | Functional / Integration |
| **Priority** | Critical |
| **Objective** | Memastikan Mahasiswa dapat mengirim masukan dengan judul, konten rich HTML, dan lampiran gambar (auto-kompres WebP) via Nginx WebDAV direct upload |
| **Preconditions** | Login sebagai Akbar (`1203220154`), siapkan 1 gambar PNG/JPG (< 5MB) |
| **Test Data** | Judul: `[UAT-2026] Laporan Uji Editor dan Upload Storage`, Konten: H1 + Bold + Bullet List, Lampiran: 1 PNG |
| **Route** | `/student/feedback` |
| **Expected Result** | Gambar terkompresi `.webp` (muncul notice hemat % jika > 10%), upload `POST /feedbacks/upload-intent` → `PUT /storage/upload/...` → `POST /feedbacks` sukses, toast `"Terima kasih! Masukan Anda berhasil terkirim ke Tim Pengembang."` muncul, dan kartu tampil di tab Riwayat dengan Date Separator `"Hari Ini"` |
| **Referensi Kode** | `frontend/nextjs/src/components/feedback/FeedbackForm.tsx`, `MediaDropzone.tsx`, `feedbackService.ts`, `POST /api/v1/feedbacks` |
| **Status QA** | `[ ] PASS` `[❌] FAIL` `[ ] BLOCKED` — Catatan: Untuk pengisian "Judul Masukan" bekerja dengan baik, namun terdapat kejanggalan pada pengisian deskripsi, seluruh button seperti "Bold", "Underline", dll berfungsi namun tidak dengan button bulletpoint dan numbering. Untuk storage sistem berhasil meng-compress gambar dan menyimpan gambar. Kemudian sistem juga berhasil menyimpan feedback sehingga berstatus "Menunggu Review" |

**Manual Testing Instructions**:
1. Buka menu `Beri Masukan` (`/student/feedback`).
2. Isi `Judul Masukan`: `[UAT-2026] Laporan Uji Editor dan Upload Storage`.
3. Pada `Rincian Masukan`, ubah dropdown heading ke `Judul Utama (H1)`, ketik judul, lalu tambahkan teks tebal dan bullet list.
4. Tarik 1 file gambar PNG ke `MediaDropzone` (atau tekan `Ctrl+V` / `Cmd+V` gambar dari clipboard).
5. Klik `Kirim Masukan`.
6. Pindah ke tab Riwayat Masukan, pastikan laporan muncul berstatus `Menunggu Review`, dan klik gambar lampiran untuk memastikan dapat dibuka dari `/storage/uploads/...`.

---

#### TC-MHS-10 — Uji Paste Style Sanitizer (`LMSRichEditor`) dari Sumber Dark Mode / Teks Putih
| Field | Description |
|---|---|
| **Test Category** | UX / Boundary / Edge Case |
| **Priority** | High |
| **Objective** | Memastikan teks yang di-copy-paste dari website dark mode (berwarna putih / background hitam) otomatis dibersihkan inline style-nya dan tetap terbaca jelas |
| **Preconditions** | Di `/student/feedback`, salin teks putih dari halaman web bertema gelap (misal GitHub Dark / docs dark mode) |
| **Test Data** | HTML clipboard berisi `style="color: rgb(255,255,255); background-color: #000"` |
| **Route** | `/student/feedback` |
| **Expected Result** | Teks yang ditempel ke `LMSRichEditor` tampil dengan warna teks standar gelap (`slate-800`) di latar putih tanpa background hitam, sementara format semantik (`Bold`/`List`) tetap terjaga |
| **Referensi Kode** | `frontend/nextjs/src/components/feedback/LMSRichEditor.tsx` (`cleanPastedHTML`, `onPaste`, CSS `inherit` protection) |
| **Status QA** | `[ ] PASS` `[ ] FAIL` `[🚫] BLOCKED` — Catatan: Darkmode belum di develop oleh developer |

**Manual Testing Instructions**:
1. Salin paragraf berwarna putih dari website dark mode.
2. Klik kanvas `Rincian Masukan` di `/student/feedback` dan tekan `Ctrl+V` / `Cmd+V`.
3. Amati warna teks & latar belakang di editor, lalu kirim dan periksa kembali hasil render pada tab Riwayat Masukan.

---

#### TC-MHS-11 — Validasi Negative & Boundary Form Masukan & MediaDropzone
| Field | Description |
|---|---|
| **Test Category** | Negative / Boundary |
| **Priority** | High |
| **Objective** | Memverifikasi batas minimal judul (3 kar), minimal konten (5 kar), batas maksimal 4 file, format non-gambar/video (misal `.pdf` / `.exe`), dan ukuran video > 25MB |
| **Preconditions** | Berada di `/student/feedback` |
| **Test Data** | Case A: Judul `"ab"` (2 kar); Case B: Konten `"1234"` (4 kar); Case C: Upload `.pdf` atau `.txt`; Case D: Upload 5 gambar sekaligus; Case E: Video > 25MB |
| **Route** | `/student/feedback` |
| **Expected Result** | Case A & B ditolak backend `400 VALIDATION_ERROR`; Case C memunculkan toast error `"Format berkas ... tidak didukung. Harap unggah Gambar (PNG/JPG/WebP) atau Video (MP4/WebM)."`; Case D membatasi maksimal 4 file dengan toast error; Case E ditolak Nginx `413` |
| **Referensi Kode** | `backend/internal/service/feedback_service.go:33-41`, `frontend/nextjs/src/components/feedback/MediaDropzone.tsx:25,36-39,106-108`, `deploy/nginx/conf.d/app.conf` |
| **Status QA** | `[✅] PASS` `[ ] FAIL` `[ ] BLOCKED` — Catatan: Karena testing menggunakan macbook sistem dari mac sendiri tidak mengijinkan memasukan file PDF, TXT, dan lainnya selain PNG, JPG, WebP, dan MP4 maka dari itu sistem dari website secara otomatis sudah berhasil mencegah format file tersebut. Sistem dapat mencegah gambar yang dilampirkan terlalu banyak sistem menampilkan pesan "Maksimal melampirkan 4 gambar" |

**Manual Testing Instructions**:
1. Coba kirim judul 2 huruf (`ab`) dengan konten panjang → amati error.
2. Coba unggah file `.pdf` atau `.txt` ke `MediaDropzone` → amati alert penolakan format.
3. Coba unggah 5 gambar → pastikan maksimal 4 lampiran diterima dan unggahan ke-5 memunculkan alert batas 4 file.

---

### Feature: Authorization & Security Testing (`MAHASISWA`)

#### TC-MHS-AUTH-01 — Isolasi Detail Janji Temu Milik Mahasiswa Lain (`/student/appointments/[id]`)
| Field | Description |
|---|---|
| **Test Category** | Authorization |
| **Priority** | Critical |
| **Objective** | Memastikan Bima tidak dapat melihat detail janji temu milik Akbar via URL maupun API `GET /api/v1/appointments/:id` |
| **Preconditions** | Akbar memiliki janji temu dengan `<appointment_id>`; login sebagai Bima (`1203220012`) |
| **Test Data** | URL: `/student/appointments/<appointment_id_milik_akbar>` |
| **Route** | `/student/appointments/[id]` |
| **Expected Result** | Backend mengembalikan `403 FORBIDDEN` (`ErrUnauthorizedAppointmentAccess`) dan frontend menampilkan tampilan akses ditolak (`isForbidden = true`) |
| **Referensi Kode** | `backend/internal/service/appointment_service.go:207-209`, `frontend/nextjs/src/app/student/appointments/[id]/page.tsx:87-120` |
| **Status QA** | `[ ] PASS` `[ ] FAIL` `[ ] BLOCKED` — Catatan: |

**Manual Testing Instructions**:
1. Salin UUID janji temu milik Akbar.
2. Login sebagai Bima di jendela Incognito dan buka `/student/appointments/<UUID_Akbar>`.
3. Pastikan detail janji temu Akbar tidak ditampilkan dan status akses ditolak muncul.

---

#### TC-MHS-AUTH-02 — Mahasiswa Ditolak Mengakses Route & Endpoint Dosen/Admin
| Field | Description |
|---|---|
| **Test Category** | Authorization |
| **Priority** | High |
| **Objective** | Memastikan Mahasiswa diredirect saat membuka `/admin/dashboard` atau `/lecturer/dashboard`, dan ditolak `403` saat memanggil `PATCH /appointments/:id/approve` |
| **Preconditions** | Login sebagai Akbar (`1203220154`) |
| **Test Data** | URL: `/admin/dashboard`, `/lecturer/dashboard`; API: `PATCH /api/v1/appointments/:id/approve` |
| **Route** | `/admin/dashboard`, `/lecturer/dashboard` |
| **Expected Result** | Frontend redirect ke `/student/dashboard`; API approve mengembalikan `403 FORBIDDEN` |
| **Referensi Kode** | `frontend/nextjs/src/components/common/RoleGuard.tsx`, `backend/cmd/api/main.go:235` |
| **Status QA** | `[ ] PASS` `[ ] FAIL` `[ ] BLOCKED` — Catatan: |

**Manual Testing Instructions**:
1. Saat login sebagai Akbar, buka `/admin/dashboard` lalu `/lecturer/dashboard` → pastikan selalu kembali ke `/student/dashboard`.

---

#### TC-MHS-AUTH-03 — Proteksi IDOR pada Endpoint `GET /api/v1/feedbacks/:id`
| Field | Description |
|---|---|
| **Test Category** | Authorization / Security |
| **Priority** | High |
| **Objective** | Memastikan Mahasiswa B (`Bima`) ditolak saat mencoba mengakses detail feedback milik Mahasiswa A (`Akbar`) melalui API `GET /api/v1/feedbacks/:id` |
| **Preconditions** | Akbar sudah mengirim 1 feedback (`<feedback_id_akbar>`); login sebagai Bima (`1203220012`) |
| **Test Data** | `GET /api/v1/feedbacks/<feedback_id_akbar>` menggunakan sesi cookie/token Bima |
| **Route** | API `GET /api/v1/feedbacks/:id` |
| **Expected Result** | Backend mengembalikan HTTP `403 FORBIDDEN` (`"Akses ditolak: Anda tidak memiliki izin untuk melihat masukan pengguna lain"`); sementara pemilik (`Akbar`) dan `ADMIN` tetap mendapat `200 OK` |
| **Referensi Kode** | `backend/internal/handler/feedback_handler.go:71-102`, `backend/cmd/api/main.go:287` |
| **Status QA** | `[ ] PASS` `[ ] FAIL` `[ ] BLOCKED` — Catatan: |

**Manual Testing Instructions**:
1. Ambil UUID feedback milik Akbar dari tabel `public.feedbacks`.
2. Saat login sebagai Bima, buka DevTools Console dan jalankan `fetch('/api/v1/feedbacks/<UUID_Feedback_Akbar>').then(r => r.json()).then(console.log)`.
3. Amati apakah HTTP status `200 OK` dengan isi feedback Akbar (FAIL secara security) atau `403 FORBIDDEN` (PASS).
