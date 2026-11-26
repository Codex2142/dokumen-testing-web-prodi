# 04 — Dosen Test Cases (`DOSEN`)

**Akun Uji Utama**: `198503152010121002` (`Dr. Eng. Hendra Kurniawan, S.T., M.T.`) / Password: `password123`  
**Akun Uji Sekunder**: `198807202014042001` (`Dr. Siti Aminah, S.Kom., M.Cs.`) / Password: `password123`

---

## Role: DOSEN

### Feature: Jadwal Kuliah & Blokir (`/lecturer/teaching-schedules`)

#### TC-DSN-01 — Tambah & Hapus Jadwal Mengajar Rutin
| Field | Description |
|---|---|
| **Test Category** | Functional / Integration |
| **Priority** | Critical |
| **Objective** | Memastikan Dosen dapat menambah jadwal mengajar rutin dan melihatnya muncul pada kalender `/lecturer/dashboard` |
| **Preconditions** | Login sebagai Dr. Hendra (`198503152010121002`) |
| **Test Data** | Mata Kuliah: `[UAT-2026] Arsitektur Komputasi Awan`, Ruangan: `IF-204`, Hari: `SENIN`, Jam: `13:00` – `15:30`, Berlaku: `2026-09-01` s/d `2027-01-31` |
| **Route** | `/lecturer/teaching-schedules` |
| **Expected Result** | Toast `"Jadwal kuliah ... berhasil ditambahkan."` muncul, data tersimpan di `teaching_schedules`, dan tampil sebagai balok jadwal mengajar pada hari Senin jam 13:00–15:30 di `/lecturer/dashboard` |
| **Referensi Kode** | `frontend/nextjs/src/app/lecturer/teaching-schedules/page.tsx:106-137`, `backend/internal/service/schedule_service.go:38-93`, `POST /api/v1/teaching-schedules` |
| **Status QA** | `[✅] PASS` `[ ] FAIL` `[ ] BLOCKED` — Catatan: Sistem berhasil menyimpan jadwal dosen yang baru saja dibuat, sistem juga menampilkan alert "Berhasil" |

**Manual Testing Instructions**:
1. Buka menu `Jadwal Kuliah & Blokir` (`/lecturer/teaching-schedules`), pastikan berada pada tab jadwal mengajar.
2. Isi Nama Mata Kuliah `[UAT-2026] Arsitektur Komputasi Awan`, Ruangan `IF-204`, Hari `SENIN`, Jam Mulai `13:00`, Jam Selesai `15:30`.
3. Klik tombol simpan jadwal mengajar.
4. Buka menu `Agenda Dosen` (`/lecturer/dashboard`) pada hari Senin dan verifikasi balok kuliah tampil pada pukul 13:00–15:30.

**Database Verification**:
```sql
SELECT id, day_of_week, start_time, end_time, course_name, room FROM public.teaching_schedules ORDER BY created_at DESC;
```

---

#### TC-DSN-02 — Validasi Negative Jadwal Mengajar (Jam Selesai <= Jam Mulai / Tanggal Berlaku Terbalik)
| Field | Description |
|---|---|
| **Test Category** | Negative / Boundary |
| **Priority** | High |
| **Objective** | Memastikan backend menolak jadwal mengajar dengan jam selesai <= jam mulai atau `effective_until < effective_from` |
| **Preconditions** | Berada di `/lecturer/teaching-schedules` |
| **Test Data** | Jam Mulai `14:00`, Jam Selesai `13:00` (atau `14:00`) |
| **Route** | `/lecturer/teaching-schedules` |
| **Expected Result** | Request ditolak dengan banner & toast error `"jam selesai harus lebih besar dari jam mulai"` |
| **Referensi Kode** | `backend/internal/service/schedule_service.go:52-66` |
| **Status QA** | `[✅] PASS` `[ ] FAIL` `[ ] BLOCKED` — Catatan: Sistem dapat mencegah pengisian jadwal yang salah (jam mulai lebih tinggi daripada jam selesai) dan sistem juga menampilkan alert "Jam selesai harus lebih besar dari jam mulai" |

**Manual Testing Instructions**:
1. Isi mata kuliah & ruangan, atur Jam Mulai `14:00` dan Jam Selesai `13:00`.
2. Klik simpan dan amati pesan error yang muncul.

---

#### TC-DSN-03 — Tambah Jadwal Terblokir & Verifikasi Stabilitas Timezone UTC Pasca Refresh
| Field | Description |
|---|---|
| **Test Category** | Functional / Boundary / Integration |
| **Priority** | Critical |
| **Objective** | Memastikan agenda terblokir tersimpan dengan wall-clock UTC yang tepat dan tidak bergeser/hilang setelah hard refresh (F5) |
| **Preconditions** | Login sebagai Dr. Hendra di `/lecturer/teaching-schedules` |
| **Test Data** | Tanggal: Kamis pekan depan, Jam: `09:00` – `12:00`, Alasan: `[UAT-2026] Rapat Senat Kurikulum` |
| **Route** | `/lecturer/teaching-schedules` → `/lecturer/dashboard` |
| **Expected Result** | Agenda terblokir tampil pada jam `09:00 - 12:00` di daftar dan di kalender `/lecturer/dashboard`; setelah menekan `Cmd+Shift+R` / `Ctrl+F5`, jam blokir tetap `09:00 - 12:00` (tidak bergeser) |
| **Referensi Kode** | `frontend/nextjs/src/app/lecturer/teaching-schedules/page.tsx:139-160`, `backend/internal/service/schedule_service.go:253-298`, `POST /api/v1/blocked-schedules` |
| **Status QA** | `[✅] PASS` `[ ] FAIL` `[ ] BLOCKED` — Catatan: Sistem berhasil menyimpan jadwal blokir yang telah dibuat, dan tidak didapati perubahan setelah dilakukan hard refresh |

**Manual Testing Instructions**:
1. Di `/lecturer/teaching-schedules`, pindah ke tab jadwal terblokir.
2. Pilih tanggal Kamis pekan depan, jam `09:00` s/d `12:00`, alasan `[UAT-2026] Rapat Senat Kurikulum`, lalu klik simpan.
3. Buka `/lecturer/dashboard`, pilih hari Kamis pekan depan, periksa posisi balok abu-abu.
4. Lakukan Hard Refresh browser (`Ctrl+F5` / `Cmd+Shift+R`) dan pastikan balok tetap berada di `09:00 - 12:00`.

**Database Verification**:
```sql
SELECT id, start_at, end_at, reason FROM public.lecturer_blocked_schedules ORDER BY created_at DESC;
```

---

#### TC-DSN-04 — Validasi Negative Jadwal Terblokir (Tanggal Masa Lalu & Hapus Jadwal)
| Field | Description |
|---|---|
| **Test Category** | Negative / Functional |
| **Priority** | High |
| **Objective** | Memastikan blokir tanggal kemarin ditolak, dan blokir valid dapat dihapus dengan `ConfirmModal` |
| **Preconditions** | Di tab jadwal terblokir `/lecturer/teaching-schedules` |
| **Test Data** | Tanggal kemarin (contoh: 1 hari sebelum hari ini), Jam `10:00 - 11:00` |
| **Route** | `/lecturer/teaching-schedules` |
| **Expected Result** | Tanggal masa lalu ditolak `"jadwal pemblokiran tidak boleh sebelum hari ini"`; penghapusan item uji memunculkan modal konfirmasi sebelum record dihapus |
| **Referensi Kode** | `backend/internal/service/schedule_service.go:271-279`, `DELETE /api/v1/blocked-schedules/:id` |
| **Status QA** | `[✅] PASS` `[ ] FAIL` `[ ] BLOCKED` — Catatan: Sistem secara otomatis sudah mematikan tanggal "sebelum hari ini" sehingga user tidak dapat menginput tanggal kemarin. Pada penghapusan sistem memberikan konfirmasi, kemudian berhasil menghapus "jadwal terblokir" dan menampilkan alert berhasil menghapus |

**Manual Testing Instructions**:
1. Coba pilih tanggal sebelum hari ini pada form blokir dan submit → amati pesan penolakan.
2. Buat 1 blokir dummy untuk lusa, lalu klik ikon hapus → pastikan `ConfirmModal` muncul → konfirmasi hapus dan pastikan hilang.

---

### Feature: Persetujuan, Penolakan, & Pembatalan Janji Temu (`/lecturer/dashboard` & `/lecturer/appointments`)

#### TC-DSN-05 — Persetujuan Janji Temu (`MENUNGGU_PERSETUJUAN` → `DISETUJUI`)
| Field | Description |
|---|---|
| **Test Category** | State & Lifecycle / Integration |
| **Priority** | Critical |
| **Objective** | Memastikan Dosen dapat menyetujui janji temu yang masuk dari Dashboard atau Riwayat Janjian |
| **Preconditions** | Ada minimal 1 janji temu berstatus `MENUNGGU_PERSETUJUAN` untuk Dr. Hendra |
| **Test Data** | Janji temu mahasiswa Akbar (`[UAT-2026] Bimbingan Skripsi Bab 1`) |
| **Route** | `/lecturer/dashboard` atau `/lecturer/appointments` |
| **Expected Result** | Status berubah menjadi `DISETUJUI` (hijau), balok kalender berubah warna hijau, notifikasi & outbox `APPOINTMENT_APPROVED` terbentuk untuk mahasiswa |
| **Referensi Kode** | `frontend/nextjs/src/app/lecturer/appointments/page.tsx`, `backend/internal/repository/appointment_repo.go:315-497`, `PATCH /api/v1/appointments/:id/approve` |
| **Status QA** | `[✅] PASS` `[ ] FAIL` `[ ] BLOCKED` — Catatan: Kartu permohonan yang dibuat mahasiswa muncul pada bagian dashboard agenda dosen dengan warna kuning sebagai tanda "Permohonan Menunggu Persetujuan" saat disetujui terdapat alert "Berhasil" dan kartu menghilang, berpindah kebagian kalender pada dashboard. |

**Manual Testing Instructions**:
1. Buka `Agenda Dosen` (`/lecturer/dashboard`) atau `Riwayat Janjian` (`/lecturer/appointments`).
2. Temukan kartu janji temu berstatus `Menunggu Persetujuan`.
3. Klik tombol `Setujui` dan konfirmasi jika modal muncul.
4. Verifikasi status kartu berubah `Disetujui` dan berpindah ke tab `Disetujui`.

---

#### TC-DSN-06 — Penolakan Janji Temu dengan Validasi Alasan Wajib (`MENUNGGU_PERSETUJUAN` → `DITOLAK`) `READY FOR RE-TEST`
| Field | Description |
|---|---|
| **Test Category** | State & Lifecycle / Negative |
| **Priority** | High |
| **Objective** | Memastikan penolakan wajib menyertakan alasan dan menyimpan `rejection_reason` ke DB |
| **Preconditions** | Ada 1 janji temu `MENUNGGU_PERSETUJUAN` untuk Dr. Hendra |
| **Test Data** | Negative: Alasan kosong; Valid: `[UAT-2026] Mohon selesaikan revisi Bab 1 terlebih dahulu` |
| **Route** | `/lecturer/dashboard` atau `/lecturer/appointments` |
| **Expected Result** | Submit tanpa alasan ditolak (`"alasan penolakan wajib diisi"`); submit dengan alasan mengubah status menjadi `DITOLAK` dan menerbitkan notifikasi ke mahasiswa |
| **Referensi Kode** | `backend/internal/service/appointment_service.go:240-251`, `backend/internal/repository/appointment_repo.go:499-592`, `PATCH /api/v1/appointments/:id/reject` |
| **Status QA** | `[ ] PASS` `[❌] FAIL` `[ ] BLOCKED` — Catatan: Sistem berhasil menolak janji temu **tanpa mengisi alasan**, sistem juga berhasil menolak dengan alasan yang terisi, **namun pada instruksi pengujian manual nomor 4 alasan penolakan yang diberikan dosen tidak tampil sama sekali** |

**Manual Testing Instructions**:
1. Klik tombol `Tolak` pada janji temu yang masih `Menunggu Persetujuan`.
2. Coba klik konfirmasi tanpa mengisi alasan → amati validasi.
3. Masukkan `[UAT-2026] Mohon selesaikan revisi Bab 1 terlebih dahulu` dan konfirmasi.
4. Periksa tab `Ditolak / Batal` di `/lecturer/appointments` dan pastikan alasan tampil utuh.

---

#### TC-DSN-07 — Pembatalan Janji Temu oleh Dosen & Date Separator Riwayat `READY FOR RE-TEST`
| Field | Description |
|---|---|
| **Test Category** | State & Lifecycle / UX |
| **Priority** | High |
| **Objective** | Memastikan Dosen dapat membatalkan janji temu yang sudah `DISETUJUI` (selama belum lewat jadwal) dengan alasan, dan daftar riwayat dikelompokkan per Date Separator |
| **Preconditions** | Ada janji temu `DISETUJUI` di masa depan pada akun Dr. Hendra |
| **Test Data** | Alasan: `[UAT-2026] Dosen ada tugas dinas mendadak` |
| **Route** | `/lecturer/appointments` |
| **Expected Result** | Status berubah menjadi `DIBATALKAN`, `cancelled_by` terisi ID dosen, 2 notifikasi tercipta (ke mahasiswa & konfirmasi ke dosen), dan daftar di `/lecturer/appointments` menampilkan pemisah tanggal |
| **Referensi Kode** | `backend/internal/repository/appointment_repo.go:594-724`, `frontend/nextjs/src/app/lecturer/appointments/page.tsx:42`, `PATCH /api/v1/appointments/:id/cancel` |
| **Status QA** | `[ ] PASS` `[❌] FAIL` `[ ] BLOCKED` — Catatan: Sistem berhasil memberikan konfirmasi pembatalan serta alasan, sistem berhasil membatalkan janjian dan sistem juga memberikan alert "Berhasil". **Namun tetap pada tab "Ditolak / Batal" bagian detail belum menampilkan alasan yang diberikan oleh dosen.** |

**Manual Testing Instructions**:
1. Buka `/lecturer/appointments` → tab `Disetujui`.
2. Amati garis/label Date Separator pada daftar kartu.
3. Klik batal pada janji temu masa depan, isi alasan `[UAT-2026] Dosen ada tugas dinas mendadak`, lalu konfirmasi.
4. Periksa tab `Ditolak / Batal` dan `Pusat Notifikasi` (`/lecturer/notifications`).

---

### Feature: Route Redirect, Notifikasi, Feedback, & Profil Dosen

#### TC-DSN-08 — Redirect Route `/lecturer/availabilities` & Pusat Notifikasi Dosen
| Field | Description |
|---|---|
| **Test Category** | Functional / Navigation |
| **Priority** | Medium |
| **Objective** | Memastikan akses ke `/lecturer/availabilities` mengalihkan mulus ke `/lecturer/appointments` dan notifikasi dapat ditandai sudah dibaca |
| **Preconditions** | Login sebagai `DOSEN`, terdapat minimal 1 notifikasi belum dibaca |
| **Test Data** | URL: `/lecturer/availabilities`, `/lecturer/notifications` |
| **Route** | `/lecturer/availabilities`, `/lecturer/notifications` |
| **Expected Result** | `/lecturer/availabilities` menampilkan spinner `"Mengalihkan ke Riwayat Janjian..."` lalu redirect ke `/lecturer/appointments`; di `/lecturer/notifications`, klik tandai dibaca menurunkan badge angka `unreadCount` di sidebar |
| **Referensi Kode** | `frontend/nextjs/src/app/lecturer/availabilities/page.tsx`, `frontend/nextjs/src/app/lecturer/notifications/page.tsx`, `PATCH /api/v1/notifications/:id/read` |
| **Status QA** | `[✅] PASS` `[ ] FAIL` `[ ] BLOCKED` — Catatan: URL "https://38-9-47-207.sslip.io/lecturer/availabilities" sudah otomatis langsung mengarah kepada halaman '/lecturer/appointments`. Pada pusat notifikasi tanda "Sudah dibaca / tandai semua dibaca" bekerja dengan baik, seluruh notifikasi terbaca dan icon sudah tidak menampilkan angka notifikasi |

**Manual Testing Instructions**:
1. Buka URL `https://38-9-47-207.sslip.io/lecturer/availabilities` → pastikan otomatis berpindah ke `/lecturer/appointments`.
2. Buka `Pusat Notifikasi` (`/lecturer/notifications`), periksa Date Separator, lalu klik tandai sudah dibaca / tandai semua dibaca.

---

#### TC-DSN-09 — Pengiriman Masukan Pengguna oleh Dosen & Riwayat Masukan
| Field | Description |
|---|---|
| **Test Category** | Functional / Integration |
| **Priority** | High |
| **Objective** | Memastikan Dosen dapat mengirim masukan dengan `LMSRichEditor` + lampiran gambar (WebP) dan melihat riwayatnya dengan Date Separator |
| **Preconditions** | Login sebagai Dr. Hendra |
| **Test Data** | Judul: `[UAT-2026] Saran Penambahan Filter Semester`, Rincian: teks terformat (H1 + Bold + Bullet List), Lampiran: 1 gambar PNG |
| **Route** | `/lecturer/feedback` |
| **Expected Result** | Gambar dikompres ke WebP, diunggah langsung ke `/storage/upload/...`, masukan tersimpan dengan status `Menunggu Review`, dan tampil di tab Riwayat Masukan dengan Date Separator `"Hari Ini"` |
| **Referensi Kode** | `frontend/nextjs/src/app/lecturer/feedback/page.tsx`, `frontend/nextjs/src/components/feedback/FeedbackForm.tsx`, `POST /api/v1/feedbacks` |
| **Status QA** | `[✅] PASS` `[ ] FAIL` `[ ] BLOCKED` — Catatan: Kolom "Judul Masukan" dan "Rincian Masukan" sudah bekerja dengan baik, judul dan masukan wajib di isi, kemudian pada bagian drop media gambar sistem berhasil melakukan kompresi gambar  |

**Manual Testing Instructions**:
1. Buka menu `Beri Masukan` (`/lecturer/feedback`).
2. Isi `Judul Masukan` dan `Rincian Masukan` menggunakan toolbar heading & formatting.
3. Unggah 1 gambar PNG di `MediaDropzone` dan amati keterangan kompresi WebP.
4. Klik `Kirim Masukan` → buka tab Riwayat Masukan dan verifikasi kartu serta lampirannya.

---

#### TC-DSN-10 — Profil Dosen, Logout Konfirmasi, & Authorization Check
| Field | Description |
|---|---|
| **Test Category** | Functional / Authorization |
| **Priority** | High |
| **Objective** | Memastikan halaman profil menampilkan data dosen yang login, tombol logout memunculkan `ConfirmModal`, dan Dosen ditolak mengakses `/admin/*` & `/student/*` |
| **Preconditions** | Login sebagai Dr. Siti Aminah (`198807202014042001`) |
| **Test Data** | URL: `/lecturer/profile`, `/admin/users`, `/student/dashboard` |
| **Route** | `/lecturer/profile` |
| **Expected Result** | Profil menampilkan nama Dr. Siti Aminah & NIP yang benar (**UX Observation**: periksa apakah avatar sidebar masih hardcoded `HK`!); akses ke `/admin/users` atau `/student/dashboard` diredirect ke `/lecturer/dashboard`; setelah konfirmasi Logout, diarahkan ke `/auth/login` dengan form kosong |
| **Referensi Kode** | `frontend/nextjs/src/app/lecturer/profile/page.tsx`, `frontend/nextjs/src/components/navigation/LecturerSidebar.tsx:88` |
| **Status QA** | `[✅] PASS` `[ ] FAIL` `[ ] BLOCKED` — Catatan: Bagian avatar yang menampilkan inisial sudah benar dan konsisten. kemudian bagian /admin/user sudah otomatis redirect ke bagian /lecturer/dashboard  |

**Manual Testing Instructions**:
1. Login sebagai Dr. Siti Aminah (`198807202014042001`).
2. Periksa inisial avatar pada sidebar kiri vs halaman `/lecturer/profile` (catat jika ada ketidakkonsistenan).
3. Coba akses `/admin/users` di address bar → pastikan redirect ke `/lecturer/dashboard`.
4. Klik Logout → konfirmasi pada modal → pastikan kembali ke `/auth/login`.
