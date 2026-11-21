# 09 — Test Coverage Summary & QA Execution Sign-Off

Dokumen ini merangkum seluruh cakupan pengujian berbasis source code aktual (`docs/qa/00`–`08`), klasifikasi bukti (`VERIFIED` / `INFERRED` / `UNVERIFIED`), temuan risiko, serta lembar rekap hasil eksekusi **PASS / FAIL / BLOCKED** untuk keputusan rilis **Go / No-Go**.

---

## 1. Ringkasan Metrik Sistem & Cakupan Pengujian

| Metrik | Jumlah Aktual (`VERIFIED`) | Keterangan |
| :--- | :---: | :--- |
| **Role Pengguna** | **3** | `ADMIN`, `DOSEN`, `MAHASISWA` |
| **Halaman / Route Frontend** | **22** | 3 Public/Auth, 6 Admin, 7 Dosen (1 redirect), 6 Mahasiswa |
| **Endpoint Backend API** | **44** | `/healthz`, `/metrics`, dan 42 endpoint di `/api/v1/*` (+ 2 Nginx storage routes) |
| **Tabel Database PostgreSQL** | **14** | 16 migrasi `.up.sql`, 6 ENUM, 1 GIST Exclusion Constraint, 1 Trigger |
| **Total Test Case Terdokumentasi** | **49** | Tersebar di 5 dokumen test case (`03`–`07`) |

### Distribusi Test Case per Role / Dokumen

| Dokumen | Fokus Pengujian | Jumlah TC | Rentang ID |
| :--- | :--- | :---: | :--- |
| `03-admin-test-cases.md` | Role `ADMIN` | **12** | `TC-ADM-01` s/d `TC-ADM-11`, `TC-ADM-AUTH-01` |
| `04-dosen-test-cases.md` | Role `DOSEN` | **10** | `TC-DSN-01` s/d `TC-DSN-10` |
| `05-mahasiswa-test-cases.md` | Role `MAHASISWA` | **14** | `TC-MHS-01` s/d `TC-MHS-11`, `TC-MHS-AUTH-01` s/d `03` |
| `06-cross-role-test-cases.md` | Cross-Role & Critical Flows | **5** | `TC-XROLE-01` s/d `TC-XROLE-05` |
| `07-ux-testing.md` | 12 Dimensi UX Testing | **8** | `TC-UX-01` s/d `TC-UX-08` |
| **TOTAL** | | **49** | |

### Distribusi Test Case per Kategori

| Kategori Testing | Jumlah Cakupan | Keterangan |
| :--- | :---: | :--- |
| **Functional** | 18 | Happy path seluruh fitur aktual Admin, Dosen, Mahasiswa |
| **Negative & Boundary** | 12 | Validasi NIM/NIP, domain email, durasi 15–60m, buffer 10m, H+30, jam lampau, 4 file max, >25MB, format file, panjang judul/konten |
| **Authorization & Security** | 6 | `RoleGuard` lintas role, backend `RequireRole`, isolasi appointment ID, revokasi sesi, uji IDOR `GET /feedbacks/:id` |
| **Integration & Cross-Role** | 10 | Transparansi slot, SSE Outbox Worker, reschedule, feedback → admin → pengirim, aturan sistem → booking |
| **State & Lifecycle** | 7 | Seluruh 6 status `appointment_status`, 3 status `reschedule_status`, 4 status `feedback_status`, trigger anti-cancel |
| **UX Testing (12 Dimensi)** | 8 | Clarity, Feedback, Loading, Error, Confirmation, Navigation, Form, Consistency, Empty State, Responsive, Double-Submit, Accessibility |

*(Catatan: Beberapa test case mencakup lebih dari 1 kategori secara bersamaan.)*

---

## 2. Daftar Temuan & Area `NEEDS VERIFICATION` (Quality Control — Phase 7)

### A. Status Perbaikan Temuan Pra-Testing (`VERIFIED`)
1. **✅ Security / IDOR (`TC-MHS-AUTH-03`)**: **DIPERBAIKI** — `GET /api/v1/feedbacks/:id` (`feedback_handler.go`) kini memeriksa `role != domain.RoleAdmin && item.UserID != userID` → `403 FORBIDDEN`.
2. **✅ UX Consistency (`TC-UX-05`)**: **DIPERBAIKI** — Inisial avatar sidebar di `LecturerSidebar.tsx` dan `StudentSidebar.tsx` kini dinamis mengikuti `currentUser?.name`.
3. **✅ UX Feedback (`TC-UX-02`)**: **DIPERBAIKI** — `MediaDropzone.tsx` kini memakai `useToast()` untuk pesan error format/batas file.
4. **✅ UX Accessibility (`TC-UX-08`)**: **DIPERBAIKI** — Tombol refresh di `admin/feedbacks/page.tsx` kini memiliki `aria-label` & `title`.
5. **⚠️ Infrastructure Security (Observasi Terbuka)**: Nginx `PUT /storage/upload/(.+)` menerima upload tanpa validasi token JWT / `upload_token` di layer Nginx (`deploy/nginx/conf.d/app.conf`).

### B. Endpoint Backend Tanpa UI Frontend (Membutuhkan Uji API Manual / Postman jika Diperlukan)
- `PUT /api/v1/users/me` — Halaman profil `/student/profile` dan `/lecturer/profile` saat ini read-only.
- `PUT /api/v1/academic-calendar/:id` — Halaman `/admin/academic-calendar` hanya menyediakan Create & Delete.
- `POST/GET/DELETE /api/v1/availabilities` — Halaman `/lecturer/availabilities` redirect ke `/lecturer/appointments`.
- `PUT /api/v1/teaching-schedules/:id` — Halaman `/lecturer/teaching-schedules` menyediakan Create & Delete.
- **NEEDS VERIFICATION**: Ketersediaan tombol UI respon reschedule (`ACCEPT`/`REJECT`) di halaman Dosen (`/lecturer/dashboard` & `/lecturer/appointments`) pada `TC-XROLE-03`.

---

## 3. Rekap Lembar Kerja Eksekusi QA (Diisi Saat Pengujian Manual)

| Dokumen | Total TC | PASS | FAIL | BLOCKED | Tingkat Kelulusan (%) |
| :--- | :---: | :---: | :---: | :---: | :---: |
| `03-admin-test-cases.md` | 12 | | | | |
| `04-dosen-test-cases.md` | 10 | | | | |
| `05-mahasiswa-test-cases.md` | 14 | | | | |
| `06-cross-role-test-cases.md` | 5 | | | | |
| `07-ux-testing.md` | 8 | | | | |
| **TOTAL KESELURUHAN** | **49** | | | | |

---

## 4. Keputusan Rilis (*Go / No-Go Sign-Off*)

- **Kriteria Go**: Seluruh test case `Priority: Critical` berstatus **PASS**, tidak ada error 500 yang tidak tertangani, dan temuan security kritis telah dievaluasi/ditangani.
- **Status Keputusan**: `[ ] GO (Siap Production)` / `[ ] NO-GO (Perlu Perbaikan Defect)`
- **Nama QA Tester**: ________________________ | **Tanggal**: ____ / ____ / 2026 | **Tanda Tangan**: ____________
