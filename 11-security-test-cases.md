# 11 — Security Test Cases & Panduan Pengujian Pemula

Dokumen ini berisi **14 skenario pengujian keamanan (`TC-SEC-01` s/d `TC-SEC-14`)** berbasis _trust boundary_ aktual sistem **Janjian Dosen Informatika** (`https://38-9-47-207.sslip.io`), merujuk metodologi [docs/security/SKILL.md](../security/SKILL.md).

Dokumen ini **dirancang khusus agar QA yang belum memahami alur sistem maupun tools keamanan** dapat langsung menjalankannya langkah demi langkah tanpa perlu menginstal aplikasi tambahan.

---

## Bagian 0 — Panduan Pemula Sebelum Mulai (Wajib Dibaca 3 Menit)

### 0.1. Peralatan yang Dibutuhkan (Sudah Ada di Laptop Anda)

Anda **tidak perlu** menginstal Postman, Burp Suite, atau tools hacking apa pun. Cukup gunakan:

1. **Google Chrome** (buka **2 jendela**: 1 Jendela Normal + 1 Jendela Incognito via `Cmd+Shift+N` di Mac / `Ctrl+Shift+N` di Windows).
2. **Terminal bawaan**:
    - **Mac**: Tekan `Cmd + Spasi`, ketik `Terminal`, tekan `Enter`.
    - **Windows**: Tekan tombol `Windows`, ketik `PowerShell` (atau `cmd`), tekan `Enter`. _(Di Windows PowerShell, gunakan `curl.exe` sebagai pengganti `curl`.)_

### 0.2. Kamus Mini Hasil Uji Security (Jangan Sampai Tertukar!)

Pada pengujian fungsional biasa, `200 OK` berarti bagus. **Pada pengujian keamanan (Security Testing), justru SEBALIKNYA** untuk aksi yang dilarang:

|    Kode HTTP yang Muncul     | Arti Sederhana                                             | Status Umum di Security Test                                        |
| :--------------------------: | :--------------------------------------------------------- | :------------------------------------------------------------------ |
|    **`401 Unauthorized`**    | Ditolak karena tidak punya/salah token login               | **AMAN (`PASS`)** untuk uji tanpa login / token palsu               |
|     **`403 Forbidden`**      | Ditolak karena tidak punya hak akses / bukan pemilik data  | **AMAN (`PASS`)** untuk uji akses data orang lain / menu Admin      |
|    **`400 Bad Request`**     | Ditolak karena input tidak valid / mencurigakan            | **AMAN (`PASS`)** untuk uji input rusak / payload aneh              |
| **`429 Too Many Requests`**  | Ditolak karena terlalu banyak percobaan (Rate Limit aktif) | **AMAN (`PASS`)** untuk uji brute-force / spam                      |
| **`200 OK` / `201 Created`** | Aksi **berhasil dieksekusi**                               | **BAHAYA (`FAIL`)** jika muncul pada aksi yang seharusnya dilarang! |

### 0.3. Cara Mengambil Token Login (Pilih Cara Paling Mudah)

Beberapa skenario di bawah meminta `<TOKEN_MHS_A>`, `<TOKEN_MHS_B>`, `<TOKEN_DOSEN>`, atau `<TOKEN_ADMIN>`.

**Cara 1 — Lewat Browser (Paling Mudah, Tanpa Ketik Perintah):**

1. Login ke `https://38-9-47-207.sslip.io/auth/login` menggunakan akun uji (misal Mahasiswa A).
2. Klik kanan di mana saja pada halaman → pilih **Inspect** (atau tekan `F12`).
3. Klik tab **Application** (di bar atas DevTools; jika tidak terlihat, klik ikon `>>`).
4. Di menu sebelah kiri, buka **Local Storage** → klik `https://38-9-47-207.sslip.io`.
5. Cari baris bernama **`situ_access_token`** → klik nilainya di kolom kanan (`eyJhbGci...`) lalu salin (`Cmd+C` / `Ctrl+C`). Itulah token Anda!

**Cara 2 — Simpan di Variabel Terminal (Agar Tinggal Copy-Paste Semua Perintah):**
Buka Terminal, lalu jalankan perintah ini (ganti `NIM_ANDA` dan `PASSWORD_ANDA` sesuai akun uji QA):

```bash
export BASE_URL="https://38-9-47-207.sslip.io"
export TOKEN_MHS_A="PASTE_TOKEN_MAHASISWA_A_DI_SINI"
export TOKEN_MHS_B="PASTE_TOKEN_MAHASISWA_B_DI_SINI"
export TOKEN_DOSEN="PASTE_TOKEN_DOSEN_DI_SINI"
export TOKEN_ADMIN="PASTE_TOKEN_ADMIN_DI_SINI"
```

_(Jika memakai Cara 2, Anda bisa langsung copy-paste perintah `curl` di bawah tanpa mengedit token satu per satu.)_

### 0.4. Cara Mengambil ID Data (`<APPOINTMENT_ID>` / `<FEEDBACK_ID>`)

- **Untuk Janji Temu**: Login sebagai Mahasiswa A → buka menu **Janji Temu** → klik salah satu janji temu → lihat bilah alamat (URL) browser: `.../student/appointments/`**`xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`** ← bagian belakang itulah `<APPOINTMENT_ID>`.
- **Untuk Masukan (Feedback)**: Buka `Inspect` → tab **Network** → klik **Fetch/XHR** → buka halaman `/student/feedback` tab **Riwayat** → klik request `my` → klik tab **Response** → salin nilai `"id": "..."`.

### 0.5. Aturan Keselamatan Pengujian

- Gunakan **hanya akun uji QA** (bukan akun pribadi mahasiswa/dosen asli).
- Jangan menjalankan uji spam/beban berat berlebihan di luar jumlah percobaan kecil yang tertulis di bawah.
- Jika uji upload (`TC-SEC-06`) berhasil menaruh file uji, laporkan ke developer agar file uji dibersihkan dan celah segera ditutup.

---

## Bagian 1 — Broken Access Control, IDOR & Privilege Escalation

_(Referensi domain: `docs/security/DATA-ISOLATION-AND-LIFECYCLE.md`, `WEB-PROTOCOL-AND-AUTH.md`)_

#### TC-SEC-01 — Horizontal IDOR (Mahasiswa B Mencoba Melihat/Membatalkan Data Milik Mahasiswa A)

| Field               | Description                                                                                                                                                                                   |
| ------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Category**   | Security / Authorization (IDOR)                                                                                                                                                               |
| **Priority**        | Critical                                                                                                                                                                                      |
| **Objective**       | Memastikan Mahasiswa B tidak bisa membaca detail janji temu, membatalkan janji temu, membaca detail feedback, atau menandai notifikasi milik Mahasiswa A hanya dengan menebak/menyalin ID-nya |
| **Preconditions**   | Mahasiswa A sudah punya minimal 1 janji temu (`<APPOINTMENT_ID>`) dan 1 feedback (`<FEEDBACK_ID>`); Anda sudah punya `$TOKEN_MHS_B`                                                           |
| **Test Data**       | Token Mahasiswa B + ID milik Mahasiswa A                                                                                                                                                      |
| **Route**           | `GET /api/v1/appointments/:id`, `PATCH /api/v1/appointments/:id/cancel`, `GET /api/v1/feedbacks/:id`                                                                                          |
| **Expected Result** | Seluruh percobaan akses lintas pengguna ditolak dengan HTTP **`403 Forbidden`** (atau `404 Not Found`); data Mahasiswa A **tidak bocor** dan status janji temu **tidak berubah**              |
| **Referensi Kode**  | `backend/internal/handler/appointment_handler.go`, `backend/internal/handler/feedback_handler.go:96-99`                                                                                       |
| **Status QA**       | `[✅] PASS` `[ ] FAIL` `[ ] BLOCKED` — Catatan: semua skenario berhasil dicoba dan mengembalikan ststus `403` yang artinya sistem berhasil menutupi celah keamanan token                      |

**Tata Cara Testing (Copy-Paste di Terminal):**

```bash
# 1. Uji baca detail janji temu Mahasiswa A memakai token Mahasiswa B:
curl -sS -i "$BASE_URL/api/v1/appointments/<APPOINTMENT_ID>" \
  -H "Authorization: Bearer $TOKEN_MHS_B" | head -n 15

# 2. Uji batalkan janji temu Mahasiswa A memakai token Mahasiswa B:
curl -sS -i -X PATCH "$BASE_URL/api/v1/appointments/<APPOINTMENT_ID>/cancel" \
  -H "Authorization: Bearer $TOKEN_MHS_B" \
  -H "Content-Type: application/json" \
  -d '{"reason":"uji security idor"}' | head -n 15

# 3. Uji baca detail masukan (feedback) Mahasiswa A memakai token Mahasiswa B:
curl -sS -i "$BASE_URL/api/v1/feedbacks/<FEEDBACK_ID>" \
  -H "Authorization: Bearer $TOKEN_MHS_B" | head -n 15
```

- **PASS jika**: Ketiga perintah menampilkan `HTTP/2 403` (atau `404`) dan pesan `"Akses ditolak"`.
- **FAIL jika**: Salah satu perintah menampilkan `HTTP/2 200` atau menampilkan isi data milik Mahasiswa A.

---

#### TC-SEC-02 — Vertical Privilege Escalation (Mahasiswa/Dosen Akses Endpoint Admin & Dosen)

| Field               | Description                                                                                                                                                                                                                                                           |
| ------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Category**   | Security / Authorization (RBAC)                                                                                                                                                                                                                                       |
| **Priority**        | Critical                                                                                                                                                                                                                                                              |
| **Objective**       | Memastikan Mahasiswa tidak bisa mengakses menu/API Admin maupun aksi khusus Dosen (`approve`, `reject`, `teaching-schedules`), dan Dosen tidak bisa mengakses API Admin                                                                                               |
| **Preconditions**   | Siapkan `$TOKEN_MHS_A` dan `$TOKEN_DOSEN`                                                                                                                                                                                                                             |
| **Test Data**       | Token non-Admin menembak `/api/v1/admin/users`, `/api/v1/admin/audit-logs`, dan Mahasiswa menembak `PATCH /api/v1/appointments/<ID>/approve`                                                                                                                          |
| **Route**           | `/api/v1/admin/*`, `/api/v1/appointments/:id/approve`, `/api/v1/teaching-schedules`                                                                                                                                                                                   |
| **Expected Result** | Semua request ditolak oleh middleware `RequireRole` dengan HTTP **`401 Unauthorized`** (`"Akses ditolak: peran Anda tidak memiliki izin..."`); di UI browser, mengakses `https://38-9-47-207.sslip.io/admin/dashboard` saat login Mahasiswa otomatis dialihkan keluar |
| **Referensi Kode**  | `backend/internal/middleware/auth.go:64-84`, `backend/cmd/api/main.go:262-281`                                                                                                                                                                                        |
| **Status QA**       | `[✅] PASS` `[ ] FAIL` `[ ] BLOCKED` — Catatan: sistem berhasil menolak semua request yang tidak memiliki akses dengan kode error `401 Unathorized`                                                                                                                   |

**Tata Cara Testing:**

1. **Lewat Browser (UI)**: Saat sedang login sebagai **Mahasiswa**, ketik langsung `https://38-9-47-207.sslip.io/admin/users` di bilah alamat lalu tekan `Enter` → pastikan halaman Admin **tidak tampil** (otomatis redirect).
2. **Lewat Terminal (API)**:

```bash
# Mahasiswa coba lihat daftar semua user (khusus Admin):
curl -sS -i "$BASE_URL/api/v1/admin/users" \
  -H "Authorization: Bearer $TOKEN_MHS_A" | head -n 12

# Dosen coba lihat audit logs (khusus Admin):
curl -sS -i "$BASE_URL/api/v1/admin/audit-logs" \
  -H "Authorization: Bearer $TOKEN_DOSEN" | head -n 12

# Mahasiswa coba menyetujui janji temu sendiri (khusus Dosen):
curl -sS -i -X PATCH "$BASE_URL/api/v1/appointments/<APPOINTMENT_ID>/approve" \
  -H "Authorization: Bearer $TOKEN_MHS_A" | head -n 12
```

- **PASS jika**: Ketiga respon menunjukkan `HTTP/2 403`.
- **FAIL jika**: Ada yang mengembalikan `HTTP/2 200`.

---

## Bagian 2 — Authentication, JWT, & Session Security

_(Referensi domain: `docs/security/WEB-PROTOCOL-AND-AUTH.md`)_

#### TC-SEC-03 — Manipulasi Token JWT (Token Palsu, Tanpa Signature, Format Salah)

| Field               | Description                                                                                                                                        |
| ------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Category**   | Security / Authentication                                                                                                                          |
| **Priority**        | Critical                                                                                                                                           |
| **Objective**       | Memastikan backend menolak token yang diubah payload/signature-nya, token tanpa algoritma (`alg: none`), atau request tanpa header `Authorization` |
| **Preconditions**   | Tidak perlu login                                                                                                                                  |
| **Test Data**       | Token rusak, token `alg:none`, dan request kosong                                                                                                  |
| **Route**           | `GET /api/v1/users/me`                                                                                                                             |
| **Expected Result** | Seluruh percobaan ditolak dengan HTTP **`401 Unauthorized`** (`"Token tidak valid atau telah kedaluwarsa"`)                                        |
| **Referensi Kode**  | `backend/internal/middleware/auth.go:21-62`, `backend/pkg/token/`                                                                                  |
| **Status QA**       | `[✅] PASS` `[ ] FAIL` `[ ] BLOCKED` — Catatan: semua case berhasil ditangani dan berhasil mengembalikan status `401 Unathorized`                  |

**Tata Cara Testing (Copy-Paste di Terminal):**

```bash
# 1. Tanpa token sama sekali:
curl -sS -i "$BASE_URL/api/v1/users/me" | head -n 10

# 2. Token JWT alg:none palsu:
curl -sS -i "$BASE_URL/api/v1/users/me" \
  -H "Authorization: Bearer eyJhbGciOiJub25lIiwidHlwIjoiSldUIn0.eyJyb2xlIjoiQURNSU4ifQ." | head -n 10

# 3. Token asli yang 3 huruf terakhirnya diganti "XXX":
curl -sS -i "$BASE_URL/api/v1/users/me" \
  -H "Authorization: Bearer ${TOKEN_MHS_A%???}XXX" | head -n 10
```

- **PASS jika**: Ketiga respon menunjukkan `HTTP/2 401`.
- **FAIL jika**: Ada yang mengembalikan `HTTP/2 200` atau `500 Internal Server Error`.

---

#### TC-SEC-04 — Deteksi Pencurian Refresh Token (_Token Reuse Detection_) & Logout

| Field               | Description                                                                                                                                                                                                                                                                                                                                                                             |
| ------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Category**   | Security / Session Lifecycle                                                                                                                                                                                                                                                                                                                                                            |
| **Priority**        | Critical                                                                                                                                                                                                                                                                                                                                                                                |
| **Objective**       | Memastikan jika refresh token lama yang sudah pernah diputar/dipakai dicoba digunakan kembali (_replay_), sistem mendeteksi penyalahgunaan dan mencabut seluruh sesi user (`RevokeAllUserSessions`)                                                                                                                                                                                     |
| **Preconditions**   | Siapkan 1 akun uji QA (NIM & password)                                                                                                                                                                                                                                                                                                                                                  |
| **Test Data**       | `refresh_token` hasil login dipakai 2 kali berturut-turut                                                                                                                                                                                                                                                                                                                               |
| **Route**           | `POST /api/v1/auth/login` → `POST /api/v1/auth/refresh`                                                                                                                                                                                                                                                                                                                                 |
| **Expected Result** | 1) Pemanggilan `/auth/refresh` pertama berhasil (`200 OK`) dan memberikan `refresh_token` baru. 2) Pemanggilan `/auth/refresh` kedua memakai **`refresh_token` lama yang sama** ditolak **`401 TOKEN_REUSE_DETECTED`** (`"Sesi telah dibatalkan karena indikasi penyalahgunaan token"`). 3) Bahkan `refresh_token` baru pun ikut hangus sehingga penyerang & korban dipaksa login ulang |
| **Referensi Kode**  | `backend/internal/service/auth_service.go:116-123`, `backend/internal/handler/auth_handler.go:117-121`                                                                                                                                                                                                                                                                                  |
| **Status QA**       | `[✅] PASS` `[ ] FAIL` `[ ] BLOCKED` — Catatan: refresh token yang didapatkan berahasul memanggil refresh token pertama dan hanya bisa dijalankan 1x saja. tidak bisa refresh token berkali kali                                                                                                                                                                                        |

**Tata Cara Testing (Copy-Paste di Terminal — ganti NIM & password akun uji):**

```bash
# 1. Login untuk dapat refresh_token_1 (salin nilai "refresh_token" dari output):
curl -sS -X POST "$BASE_URL/api/v1/auth/login" \
  -H "Content-Type: application/json" \
  -d '{"identifier_or_email":"NIM_AKUN_UJI","password":"PASSWORD_AKUN_UJI"}'

# 2. Panggil refresh pertama dengan REFRESH_TOKEN_1 (HARUS 200 OK):
curl -sS -i -X POST "$BASE_URL/api/v1/auth/refresh" \
  -H "Content-Type: application/json" \
  -d '{"refresh_token":"PASTE_REFRESH_TOKEN_1"}' | head -n 15

# 3. Ulangi perintah yang PERSIS SAMA dengan REFRESH_TOKEN_1 (HARUS 401 TOKEN_REUSE_DETECTED):
curl -sS -i -X POST "$BASE_URL/api/v1/auth/refresh" \
  -H "Content-Type: application/json" \
  -d '{"refresh_token":"PASTE_REFRESH_TOKEN_1"}' | head -n 15
```

- **PASS jika**: Langkah 3 mengembalikan `HTTP/2 401` dengan kode `TOKEN_REUSE_DETECTED`.
- **FAIL jika**: Langkah 3 masih mengembalikan `HTTP/2 200`.

---

#### TC-SEC-05 — Pembatasan Query Token (`?token=`) Hanya untuk SSE Stream

| Field               | Description                                                                                                                                                                                                            |
| ------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Category**   | Security / Token Exposure                                                                                                                                                                                              |
| **Priority**        | High                                                                                                                                                                                                                   |
| **Objective**       | Memastikan parameter `?token=<JWT>` di URL hanya diizinkan pada endpoint `/api/v1/notifications/stream` (kebutuhan EventSource browser) dan **ditolak** pada semua endpoint API lain agar token tidak bocor di log URL |
| **Preconditions**   | Siapkan `$TOKEN_MHS_A`                                                                                                                                                                                                 |
| **Test Data**       | `GET /api/v1/users/me?token=$TOKEN_MHS_A` tanpa header `Authorization`                                                                                                                                                 |
| **Route**           | `GET /api/v1/users/me`, `GET /api/v1/appointments`                                                                                                                                                                     |
| **Expected Result** | Ditolak dengan HTTP **`401 Unauthorized`**                                                                                                                                                                             |
| **Referensi Kode**  | `backend/internal/middleware/auth.go:33-37`                                                                                                                                                                            |
| **Status QA**       | `[✅] PASS` `[ ] FAIL` `[ ] BLOCKED` — Catatan: testig berhasil mendapatkan respon `401 Unauthorized` yang mana mengindikasikan keamanan ketika token digunakan pada endpoint lainnya                                  |

**Tata Cara Testing (Copy-Paste di Terminal):**

```bash
curl -sS -i "$BASE_URL/api/v1/users/me?token=$TOKEN_MHS_A" | head -n 12
```

- **PASS jika**: Muncul `HTTP/2 401`.
- **FAIL jika**: Muncul `HTTP/2 200` dan menampilkan data profil.

---

## Bagian 3 — File Upload & Nginx Storage Boundary

_(Referensi domain: `docs/security/CLOUD-AND-DEPLOYMENT.md`)_

#### TC-SEC-06 — Unauthenticated Direct Upload pada Nginx `PUT /storage/upload/(.+)` _(KNOWN GAP)_

| Field               | Description                                                                                                                                                                                                                                                                                                                                                       |
| ------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Category**   | Security / Infrastructure Boundary                                                                                                                                                                                                                                                                                                                                |
| **Priority**        | Critical                                                                                                                                                                                                                                                                                                                                                          |
| **Objective**       | Memverifikasi apakah endpoint direct upload Nginx (`PUT /storage/upload/...`) menolak atau masih menerima unggahan file tanpa token login (observasi terbuka dari audit sebelumnya di `09-test-coverage-summary.md`)                                                                                                                                              |
| **Preconditions**   | Tidak perlu login                                                                                                                                                                                                                                                                                                                                                 |
| **Test Data**       | File teks kecil tidak berbahaya `qa_sec_test.txt` berisi `"uji-security-qa"`                                                                                                                                                                                                                                                                                      |
| **Route**           | `PUT /storage/upload/qa_sec_test.txt` → `GET /storage/uploads/qa_sec_test.txt`                                                                                                                                                                                                                                                                                    |
| **Expected Result** | **Target Aman (`PASS`)**: `PUT` tanpa autentikasi ditolak (`401` / `403`). <br> **Catatan Observasi Saat Ini**: Di `deploy/nginx/conf.d/app.conf:120-126`, `dav_methods PUT` belum memakai `auth_request` / validasi `upload_token`, sehingga kemungkinan besar saat ini mengembalikan `201 Created` (**tandai `FAIL — KNOWN GAP`** dan eskalasi untuk hardening) |
| **Referensi Kode**  | `deploy/nginx/conf.d/app.conf:120-126`                                                                                                                                                                                                                                                                                                                            |
| **Status QA**       | `[] PASS` `[❌] FAIL` `[ ] BLOCKED` — Catatan: sistem meloloskan pengguna tanpa login untuk mengakses storage                                                                                                                                                                                                                                                     |

**Tata Cara Testing (Copy-Paste di Terminal — sangat aman, hanya 15 byte teks):**

```bash
curl -sS -i -X PUT "$BASE_URL/storage/upload/qa_sec_test.txt" \
  --data-binary "uji-security-qa" | head -n 12
```

- **PASS jika**: Muncul `HTTP/2 401` atau `HTTP/2 403`.
- **FAIL (Celah Terkonfirmasi) jika**: Muncul `HTTP/2 201 Created` atau `HTTP/2 204 No Content`. _(Jika FAIL, catat di laporan QA agar developer menambahkan validasi token di Nginx dan menghapus `/opt/reservasi-dosen/data/storage/uploads/qa_sec_test.txt`.)_

---

#### TC-SEC-07 — Path Traversal (`../`), MIME Spoofing, & Ukuran File >25MB

| Field               | Description                                                                                                                                                                                                             |
| ------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Category**   | Security / Input & Storage Validation                                                                                                                                                                                   |
| **Priority**        | High                                                                                                                                                                                                                    |
| **Objective**       | Memastikan permintaan `upload-intent` menolak nama file berbahaya (`../`, ekstensi `.exe`/`.html`/`.svg`), ukuran `> 25MB`, dan Nginx menolak `../` path traversal                                                      |
| **Preconditions**   | Siapkan `$TOKEN_MHS_A`                                                                                                                                                                                                  |
| **Test Data**       | 1) File `../../etc/passwd` atau `shell.html` pada `POST /api/v1/feedbacks/upload-intent`; 2) `file_size: 30000000` (30MB)                                                                                               |
| **Route**           | `POST /api/v1/feedbacks/upload-intent`, `GET /storage/uploads/../../etc/passwd`                                                                                                                                         |
| **Expected Result** | 1) Ekstensi/MIME terlarang (`text/html`, `image/svg+xml`) & ukuran `>25MB` ditolak backend **`400 Bad Request`**. 2) Percobaan `../` di Nginx mengembalikan `400`/`403`/`404`, tidak pernah menampilkan isi file sistem |
| **Referensi Kode**  | `backend/internal/service/feedback_service.go`, `deploy/nginx/conf.d/app.conf:101-126`                                                                                                                                  |
| **Status QA**       | `[ ] PASS` `[❌] FAIL` `[ ] BLOCKED` — Catatan: uji pertama menghasilkan kode status `200`, uji kedua menghasilkan kode status `200`, uji ketiga menghasilkan kode status `404 Not Found`,                              |

**Tata Cara Testing (Copy-Paste di Terminal):**

```bash
# 1. Uji ekstensi/MIME HTML (berisiko XSS jika di-serve):
curl -sS -i -X POST "$BASE_URL/api/v1/feedbacks/upload-intent" \
  -H "Authorization: Bearer $TOKEN_MHS_A" \
  -H "Content-Type: application/json" \
  -d '{"file_name":"hack.html","file_type":"text/html","file_size":1024}' | head -n 12

# 2. Uji ukuran file 30MB (> batas 25MB):
curl -sS -i -X POST "$BASE_URL/api/v1/feedbacks/upload-intent" \
  -H "Authorization: Bearer $TOKEN_MHS_A" \
  -H "Content-Type: application/json" \
  -d '{"file_name":"besar.png","file_type":"image/png","file_size":30000000}' | head -n 12

# 3. Uji path traversal di Nginx:
curl -sS -i --path-as-is "$BASE_URL/storage/uploads/../../etc/passwd" | head -n 12
```

- **PASS jika**: Perintah 1 & 2 mengembalikan `400`, dan perintah 3 mengembalikan `400`/`403`/`404` (tanpa isi file).
- **FAIL jika**: Salah satu mengembalikan `200` atau menampilkan teks `root:x:0:0`.

---

## Bagian 4 — Client-Side Security (Stored XSS & CSV Formula Injection)

_(Referensi domain: `docs/security/CLIENT-SIDE.md`)_

#### TC-SEC-08 — Stored XSS pada Rich HTML Editor Masukan (`Mahasiswa → Admin`)

| Field               | Description                                                                                                                                                       |
| ------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Category**   | Security / Client-Side (XSS)                                                                                                                                      |
| **Priority**        | Critical                                                                                                                                                          |
| **Objective**       | Memastikan payload JavaScript yang dikirim Mahasiswa melalui API/editor `feedbacks` **tidak tereksekusi** saat Admin membuka detail masukan di `/admin/feedbacks` |
| **Preconditions**   | Siapkan `$TOKEN_MHS_A` dan login Admin di Jendela Incognito                                                                                                       |
| **Test Data**       | Payload XSS aman (hanya memicu kotak `alert` jika rentan): `<img src=x onerror="alert('XSS-TEST')">`                                                              |
| **Route**           | `POST /api/v1/feedbacks` → `/admin/feedbacks`                                                                                                                     |
| **Expected Result** | Saat Admin membuka modal detail masukan tersebut, **TIDAK muncul popup alert browser** dan tag berbahaya (`<script>`, `onerror=`) tersanitasi/tidak aktif         |
| **Referensi Kode**  | `frontend/nextjs/src/components/common/LMSRichRenderer.tsx`, `frontend/nextjs/src/components/common/LMSRichEditor.tsx`                                            |
| **Status QA**       | `[ ] PASS` `[❌] FAIL` `[ ] BLOCKED` — Catatan: pop up muncul dan menampilkan angka 1                                                                             |

**Tata Cara Testing:**

1. Jalankan perintah berikut di Terminal (mengirim feedback uji dengan payload XSS aman):

```bash
curl -sS -X POST "$BASE_URL/api/v1/feedbacks" \
  -H "Authorization: Bearer $TOKEN_MHS_A" \
  -H "Content-Type: application/json" \
  -d '{"title":"[QA-SEC] Uji XSS Aman","content":"<p>Tes</p><img src=x onerror=\"alert(1)\"><script>alert(2)</script>"}'
```

2. Login sebagai **Admin** di browser → buka menu **Masukan Pengguna** (`/admin/feedbacks`).
3. Klik laporan berjudul **`[QA-SEC] Uji XSS Aman`** untuk membuka modal detail.

- **PASS jika**: Modal terbuka normal dan **tidak ada popup `alert(1)` atau `alert(2)`** yang muncul.
- **FAIL (Critical) jika**: Muncul popup browser berisi angka `1` atau `2`!

---

#### TC-SEC-09 — CSV Formula Injection pada Export CSV Admin

| Field               | Description                                                                                                                                                                                                          |
| ------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Category**   | Security / Client-Side (CSV Injection)                                                                                                                                                                               |
| **Priority**        | Medium                                                                                                                                                                                                               |
| **Objective**       | Memastikan karakter awal formula Excel (`=`, `+`, `-`, `@`) pada data teks (misal judul janji temu / nama) dinetralisasi saat diekspor ke CSV agar tidak mengeksekusi rumus saat Admin membukanya di Microsoft Excel |
| **Preconditions**   | Login Admin; pastikan ada data uji atau cek output `GET /api/v1/admin/appointments/export` & `GET /api/v1/admin/users/export`                                                                                        |
| **Test Data**       | Judul janji temu/topik diawali `=1+1` (jika diuji)                                                                                                                                                                   |
| **Route**           | `/admin/users` (Export CSV), `/admin/dashboard` (`GET /api/v1/admin/appointments/export`)                                                                                                                            |
| **Expected Result** | Nilai sel yang diawali `=`, `+`, `-`, atau `@` di-prefix petik tunggal (`'`) atau dibersihkan oleh `csvutil` sehingga Excel menampilkannya sebagai teks biasa, bukan hasil rumus `2`                                 |
| **Referensi Kode**  | `backend/pkg/csvutil/`, `backend/internal/handler/admin_handler.go`                                                                                                                                                  |
| **Status QA**       | `[✅] PASS` `[ ] FAIL` `[ ] BLOCKED` — Catatan: formulasi berhasil di netrasilisasi dengan penambahan karakter (`)                                                                                                   |

**Tata Cara Testing:**

1. Login sebagai **Admin** → klik tombol **Export CSV** pada halaman Manajemen Pengguna (`/admin/users`).
2. Buka file `.csv` yang terunduh menggunakan **Notepad / TextEdit** (bukan langsung Excel dulu).
3. Periksa apakah ada kolom yang diawali mentah `=`, `+`, atau `@` tanpa tanda petik pelindung.

- **PASS jika**: Formula dinetralisasi (diawali `'` atau di-escape) atau input formula ditolak.
- **FAIL jika**: Teks `=1+1` tertulis mentah dan saat dibuka di Excel otomatis berubah menjadi angka `2`.

---

## Bagian 5 — CSRF, CORS, Security Headers & Cache Isolation

_(Referensi domain: `docs/security/WEB-PROTOCOL-AND-AUTH.md`, `CLOUD-AND-DEPLOYMENT.md`)_

#### TC-SEC-10 — Uji Perlindungan CSRF & CORS Origin Tidak Dikenal

| Field               | Description                                                                                                                                                                                                                                                                                                 |
| ------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Category**   | Security / Web Protocol                                                                                                                                                                                                                                                                                     |
| **Priority**        | High                                                                                                                                                                                                                                                                                                        |
| **Objective**       | Memastikan permintaan lintas-domain (_cross-origin_) dari situs asing ditolak oleh `CORS()`, dan permintaan mutasi berbasis cookie tanpa `X-CSRF-Token` ditolak oleh `CSRFProtection()`                                                                                                                     |
| **Preconditions**   | Tidak perlu login                                                                                                                                                                                                                                                                                           |
| **Test Data**       | Header `Origin: https://situs-jahat.example`                                                                                                                                                                                                                                                                |
| **Route**           | `OPTIONS /api/v1/users/me`, `PUT /api/v1/users/me`                                                                                                                                                                                                                                                          |
| **Expected Result** | 1) Preflight `OPTIONS` dengan `Origin: https://situs-jahat.example` ditolak **`403 Forbidden`** dan **tidak** mengembalikan header `Access-Control-Allow-Origin: https://situs-jahat.example`. 2) Request `PUT` tanpa `Authorization: Bearer` dan tanpa `X-CSRF-Token` ditolak **`403 CSRF_TOKEN_MISSING`** |
| **Referensi Kode**  | `backend/internal/middleware/cors.go:39-61`, `backend/internal/middleware/csrf.go:19-69`                                                                                                                                                                                                                    |
| **Status QA**       | `[✅] PASS` `[ ] FAIL` `[ ] BLOCKED` — Catatan: uji CORS dan CSRF berhasil mendapatkan respon `403`                                                                                                                                                                                                         |

**Tata Cara Testing (Copy-Paste di Terminal):**

```bash
# 1. Uji CORS Preflight dari domain asing (HARUS 403 & tanpa Access-Control-Allow-Origin):
curl -sS -i -X OPTIONS "$BASE_URL/api/v1/users/me" \
  -H "Origin: https://situs-jahat.example" \
  -H "Access-Control-Request-Method: GET" | head -n 15

# 2. Uji CSRF tanpa header Authorization & tanpa X-CSRF-Token (HARUS 403 CSRF_TOKEN_MISSING):
curl -sS -i -X PUT "$BASE_URL/api/v1/users/me" \
  -H "Content-Type: application/json" \
  -d '{"name":"Uji CSRF"}' | head -n 15
```

- **PASS jika**: Keduanya mengembalikan `HTTP/2 403`.
- **FAIL jika**: Ada header `Access-Control-Allow-Origin: https://situs-jahat.example` atau perintah 2 lolos melewati CSRF.

---

#### TC-SEC-11 — Security Headers (`HSTS`, `X-Frame-Options`) & Nginx `fe_microcache` Isolation

| Field               | Description                                                                                                                                                                                                                                                                                                                                                                              |
| ------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Category**   | Security / Headers & Caching                                                                                                                                                                                                                                                                                                                                                             |
| **Priority**        | Medium                                                                                                                                                                                                                                                                                                                                                                                   |
| **Objective**       | Memeriksa kelengkapan header keamanan HTTP pada respon Frontend & API, serta memastikan respon API terautentikasi (`GET /api/v1/users/me`) **tidak pernah di-cache** oleh Nginx                                                                                                                                                                                                          |
| **Preconditions**   | Tidak perlu login (untuk cek halaman login); siapkan `$TOKEN_MHS_A` untuk cek API                                                                                                                                                                                                                                                                                                        |
| **Test Data**       | `HEAD /auth/login` dan `GET /api/v1/users/me`                                                                                                                                                                                                                                                                                                                                            |
| **Route**           | `/auth/login`, `/api/v1/users/me`                                                                                                                                                                                                                                                                                                                                                        |
| **Expected Result** | 1) Respon API `/api/v1/users/me` membawa header keamanan dan **tidak** memiliki `X-Cache-Status: HIT`. 2) Periksa pada `HEAD /auth/login` apakah header `Strict-Transport-Security`, `X-Frame-Options`, dan `X-Content-Type-Options` tampil (catatan: di `app.conf:167`, `add_header X-Cache-Status` di dalam `location /` dapat menimpa `add_header` level server — catat bila hilang!) |
| **Referensi Kode**  | `deploy/nginx/conf.d/app.conf:50-56,149-168`, `backend/internal/middleware/security.go`                                                                                                                                                                                                                                                                                                  |
| **Status QA**       | `[ ] PASS` `[❌] FAIL` `[ ] BLOCKED` — Catatan: Strict-Transport-Security header tidak konsisten pada endpoint /auth/login. Jangan menyatakan seluruh security header sudah lengkap sebelum diverifikasi ulang.                                                                                                                                                                          |

**Tata Cara Testing (Copy-Paste di Terminal):**

```bash
# 1. Cek header keamanan pada halaman login:
curl -sS -I "$BASE_URL/auth/login" | grep -iE "strict-transport|x-frame|x-content-type|x-cache-status"

# 2. Cek bahwa API privat tidak pernah di-cache Nginx:
curl -sS -I "$BASE_URL/api/v1/users/me" -H "Authorization: Bearer $TOKEN_MHS_A" | grep -iE "cache|x-frame|x-content-type"
```

- **PASS jika**: API tidak memiliki `X-Cache-Status: HIT` dan seluruh security header tampil lengkap.
- **FAIL / Observasi jika**: `Strict-Transport-Security` hilang pada `/auth/login` (catat untuk perbaikan `add_header` Nginx).

---

## Bagian 6 — Injection & Input Validation

_(Referensi domain: `docs/security/ATTACK-CLASSES.md`)_

#### TC-SEC-12 — SQL Injection pada Parameter Filter & SMTP CRLF Header Injection

| Field               | Description                                                                                                                                                                                                                                                                                                                               |
| ------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Category**   | Security / Injection                                                                                                                                                                                                                                                                                                                      |
| **Priority**        | High                                                                                                                                                                                                                                                                                                                                      |
| **Objective**       | Memastikan input karakter SQL (`' OR '1'='1`, `; DROP TABLE`) tidak memicu error database (`500`) atau kebocoran data, serta karakter baris baru (`\r\n`) pada input email OTP dinetralisasi                                                                                                                                              |
| **Preconditions**   | Siapkan `$TOKEN_ADMIN`                                                                                                                                                                                                                                                                                                                    |
| **Test Data**       | `search=' OR '1'='1--` pada `/api/v1/admin/feedbacks`, serta payload `\r\nBcc:attacker@example.com` pada `/api/v1/auth/forgot-password`                                                                                                                                                                                                   |
| **Route**           | `GET /api/v1/admin/feedbacks?search=...`, `POST /api/v1/auth/forgot-password`                                                                                                                                                                                                                                                             |
| **Expected Result** | 1) Query SQL memakai parameterized placeholder (`$1`, `$2`) sehingga payload diperlakukan sebagai teks biasa → mengembalikan `200 OK` dengan daftar kosong (`[]`), **bukan `500`** dan bukan seluruh isi tabel. 2) Payload CRLF pada `forgot-password` tidak mengeksekusi header SMTP tambahan (`sanitizeHeader` menghapus `\r` dan `\n`) |
| **Referensi Kode**  | `backend/internal/repository/feedback_repo.go`, `backend/internal/notification/email_sender.go:136-140`                                                                                                                                                                                                                                   |
| **Status QA**       | `[✅] PASS` `[ ] FAIL` `[ ] BLOCKED` — Catatan: sistem berhasil mengirimkan respon pesan kode `200`                                                                                                                                                                                                                                       |

**Tata Cara Testing (Copy-Paste di Terminal):**

```bash
# 1. Uji SQLi pada filter pencarian feedback Admin:
curl -sS -i --get "$BASE_URL/api/v1/admin/feedbacks" \
  --data-urlencode "search=' OR '1'='1--" \
  -H "Authorization: Bearer $TOKEN_ADMIN" | head -n 15

# 2. Uji SMTP CRLF Header Injection pada forgot-password:
curl -sS -i -X POST "$BASE_URL/api/v1/auth/forgot-password" \
  -H "Content-Type: application/json" \
  -d '{"identifier_or_email":"test@example.com\r\nBcc:evil@example.com"}' | head -n 15
```

- **PASS jika**: Perintah 1 mengembalikan `HTTP/2 200` dengan `total: 0` (tanpa error SQL/500), dan perintah 2 mengembalikan `200 OK` generik tanpa error server.
- **FAIL jika**: Muncul `HTTP/2 500`, pesan error `syntax error at or near`, atau seluruh data terbongkar.

---

## Bagian 7 — Availability, Rate Limiting & Exposed Monitoring Surfaces

_(Referensi domain: `docs/security/RESOURCE-EXHAUSTION-AND-AVAILABILITY.md`, `CLOUD-AND-DEPLOYMENT.md`)_

#### TC-SEC-13 — Uji Rate Limiter Login/OTP (`1 req/s, burst 5`) & Anti-Spoofing `X-Forwarded-For`

| Field               | Description                                                                                                                                                                                                                                |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Test Category**   | Security / Rate Limiting                                                                                                                                                                                                                   |
| **Priority**        | High                                                                                                                                                                                                                                       |
| **Objective**       | Memastikan serangan brute-force cepat pada `/api/v1/auth/login` diblokir oleh `loginRateLimiter` (`429 Too Many Requests`) setelah burst 5, dan mengganti header `X-Forwarded-For` palsu tidak bisa menembus pembatasan IP asli dari Nginx |
| **Preconditions**   | Gunakan akun fiktif `9999999999` agar tidak mengunci akun nyata                                                                                                                                                                            |
| **Test Data**       | 8 request login cepat berturut-turut dalam 1 detik                                                                                                                                                                                         |
| **Route**           | `POST /api/v1/auth/login`                                                                                                                                                                                                                  |
| **Expected Result** | Request ke-1 s/d ke-5 mengembalikan `401 INVALID_CREDENTIALS`, lalu request ke-6 s/d ke-8 langsung diblokir **`429 Too Many Requests`** (termasuk saat disisipi header `X-Forwarded-For: 1.2.3.4` palsu)                                   |
| **Referensi Kode**  | `backend/cmd/api/main.go:64-66,192`, `backend/internal/middleware/ratelimit.go`                                                                                                                                                            |
| **Status QA**       | `[ ] PASS` `[❌] FAIL` `[ ] BLOCKED` — Catatan: setelah dilakukan spam login sebanyak 20x, sistem tetap memberikan respon kode `401`                                                                                                       |

**Tata Cara Testing (Copy-Paste di Terminal — sangat ringan, hanya 8 request):**

```bash
for i in 1 2 3 4 5 6 7 8; do
  code=$(curl -sS -o /dev/null -w "%{http_code}" -X POST "$BASE_URL/api/v1/auth/login" \
    -H "Content-Type: application/json" \
    -H "X-Forwarded-For: 10.20.30.$i" \
    -d '{"identifier_or_email":"9999999999","password":"salah"}')
  echo "Percobaan #$i -> HTTP $code"
done
```

- **PASS jika**: Setelah beberapa percobaan `401`, mulai muncul **`HTTP 429`** (rate limiter bekerja dan kebal spoofing `X-Forwarded-For`).
- **FAIL jika**: Seluruh 8 percobaan mengembalikan `401` tanpa satu pun `429`.

---

#### TC-SEC-14 — Eksposur Monitoring (`/metrics`, `/grafana/`) & Paksa Redirect HTTP → HTTPS

| Field               | Description                                                                                                                                                                                                                                                                                                                                                            |
| ------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Category**   | Security / Deployment Surface                                                                                                                                                                                                                                                                                                                                          |
| **Priority**        | Medium                                                                                                                                                                                                                                                                                                                                                                 |
| **Objective**       | Memastikan akses HTTP (port 80) otomatis dialihkan ke HTTPS (`301`), halaman `/metrics` Prometheus tidak dapat diakses publik via HTTPS Nginx, dan `/grafana/` mewajibkan login                                                                                                                                                                                        |
| **Preconditions**   | Tidak perlu login                                                                                                                                                                                                                                                                                                                                                      |
| **Test Data**       | `http://38-9-47-207.sslip.io/`, `https://38-9-47-207.sslip.io/metrics`, `https://38-9-47-207.sslip.io/grafana/`                                                                                                                                                                                                                                                        |
| **Route**           | HTTP `:80`, `/metrics`, `/grafana/`                                                                                                                                                                                                                                                                                                                                    |
| **Expected Result** | 1) Akses `http://` langsung `301 Moved Permanently` ke `https://38-9-47-207.sslip.io/`. 2) Akses publik ke `https://38-9-47-207.sslip.io/metrics` **tidak** menampilkan metrik internal Go/Prometheus (diteruskan ke Next.js `404`). 3) Akses `/grafana/` mengarahkan ke halaman login Grafana (`302` ke `/grafana/login`), tidak menampilkan dashboard tanpa password |
| **Referensi Kode**  | `deploy/nginx/conf.d/app.conf:17-33,138-147`                                                                                                                                                                                                                                                                                                                           |
| **Status QA**       | `[✅] PASS` `[ ] FAIL` `[ ] BLOCKED` — Catatan: berhasil redirect dan metric tertutup oleh publik                                                                                                                                                                                                                                                                      |

**Tata Cara Testing (Copy-Paste di Terminal):**

```bash
# 1. Cek redirect HTTP -> HTTPS (HARUS 301 ke https://...):
curl -sS -I "http://38-9-47-207.sslip.io/" | head -n 8

# 2. Cek apakah /metrics tertutup dari publik (HARUS 404, BUKAN 200 dengan go_goroutines):
curl -sS -i "$BASE_URL/metrics" | head -n 10

# 3. Cek apakah /grafana/ meminta login (HARUS 302 ke /grafana/login):
curl -sS -I "$BASE_URL/grafana/" | head -n 10
```

- **PASS jika**: HTTP→HTTPS `301`, `/metrics` `404` (tidak membocorkan metrik), dan `/grafana/` redirect ke `/grafana/login`.
- **FAIL jika**: `/metrics` mengembalikan `200` berisi metrik internal, atau `/grafana/` terbuka tanpa login.
