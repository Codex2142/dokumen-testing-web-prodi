# 01 — Role & Permission Analysis (RBAC Matrix)

Dokumen ini menganalisis seluruh mekanisme otentikasi dan otorisasi aktual pada **Frontend (`RoleGuard`)** maupun **Backend (`Authenticate`, `RequireRole`, dan ownership check pada Service/Repository)**.

**Prinsip Evidence-Based QA**: *Frontend hiding bukan pengganti backend authorization.* Setiap temuan di bawah ini berstatus `VERIFIED` dari source code aktual.

---

## 1. Inventaris Role Aktual (`user_role` ENUM)

Referensi: `backend/migrations/000001_init_extensions_and_enums.up.sql`, `backend/internal/domain/user.go`, `frontend/nextjs/src/components/common/RoleGuard.tsx`.

| Role | Dashboard Default | Layout Guard Frontend | Backend Middleware Guard |
| :--- | :--- | :--- | :--- |
| `ADMIN` | `/admin/dashboard` | `<RoleGuard allowedRole="ADMIN">` (`admin/layout.tsx`) | `authMiddleware` + `adminGuard` (`RequireRole(RoleAdmin)`) |
| `DOSEN` | `/lecturer/dashboard` | `<RoleGuard allowedRole="DOSEN">` (`lecturer/layout.tsx`) | `authMiddleware` + `dosenGuard` (`RequireRole(RoleDosen)`) |
| `MAHASISWA` | `/student/dashboard` | `<RoleGuard allowedRole="MAHASISWA">` (`student/layout.tsx`) | `authMiddleware` + `mhsGuard` (`RequireRole(RoleMahasiswa)`) |

### Mekanisme Autentikasi (`backend/internal/middleware/auth.go`)
1. **Urutan ekstraksi token**:
   - Header `Authorization: Bearer <token>`
   - Query `?token=<token>` (**hanya** untuk path berakhiran `/notifications/stream`)
   - Cookie `access_token` (HttpOnly)
2. **Proteksi CSRF (`backend/internal/middleware/csrf.go`)**:
   - Berlaku pada semua request non-GET/HEAD/OPTIONS di `/api/v1/*`, **kecuali**:
     - Path `/api/v1/auth/*`
     - Request yang menyertakan `Authorization: Bearer ...`
   - Request berbasis cookie wajib mencocokkan cookie `csrf_token` dengan header `X-CSRF-Token` (jika tidak → `403 CSRF_TOKEN_MISSING` / `CSRF_TOKEN_INVALID`).

---

## 2. Matriks Hak Akses Halaman Frontend

| Route | Public (Unauth) | `ADMIN` | `DOSEN` | `MAHASISWA` | Perilaku Jika Tidak Berwenang |
| :--- | :---: | :---: | :---: | :---: | :--- |
| `/auth/login`, `/login` | Boleh | Boleh | Boleh | Boleh | Form login aktif |
| `/admin/*` (6 halaman) | Redirect `/auth/login` | **Boleh** | Redirect `/lecturer/dashboard` | Redirect `/student/dashboard` | Dicegat `RoleGuard` |
| `/lecturer/*` (7 halaman) | Redirect `/auth/login` | Redirect `/admin/dashboard` | **Boleh** | Redirect `/student/dashboard` | Dicegat `RoleGuard` |
| `/student/*` (6 halaman) | Redirect `/auth/login` | Redirect `/admin/dashboard` | Redirect `/lecturer/dashboard` | **Boleh** | Dicegat `RoleGuard` |

---

## 3. Matriks Otorisasi Endpoint Backend (`/api/v1/*`)

| Grup / Endpoint | Public | `ADMIN` | `DOSEN` | `MAHASISWA` | Ownership / Batasan Bisnis Aktual |
| :--- | :---: | :---: | :---: | :---: | :--- |
| `POST /auth/login`, `/refresh`, `/logout` | ✅ | ✅ | ✅ | ✅ | Login cek `is_active = true`; Refresh token rotation + deteksi token reuse mencabut seluruh sesi user (`auth_service.go:116-123`). |
| `GET /academic-calendar` | ✅ | ✅ | ✅ | ✅ | Publik (tanpa `authMiddleware`). |
| `POST/PUT/DELETE /academic-calendar[/:id]` | ❌ 401 | ✅ | ❌ 403 | ❌ 403 | Khusus `ADMIN` (`adminGuard`). |
| `GET /lecturers` | ✅ | ✅ | ✅ | ✅ | Publik (tanpa `authMiddleware`). |
| `GET /settings/appointment-rules` | ❌ 401 | ✅ | ✅ | ✅ | Semua role terautentikasi. |
| `GET/PUT /users/me` | ❌ 401 | ✅ | ✅ | ✅ | Hanya profil `userID` dari JWT claims. |
| `POST/PUT/DELETE /teaching-schedules[/:id]`, `GET /teaching-schedules/my` | ❌ 401 | ❌ 403 | ✅ | ❌ 403 | Hanya `DOSEN`; Update/Delete cek kepemilikan `lecturer_id = userID`. |
| `GET /teaching-schedules/lecturer/:id`, `/activities` | ❌ 401 | ✅ | ✅ | ✅ | Semua user login dapat melihat jadwal mengajar, blokir, & slot terisi dosen (transparansi). |
| `POST/DELETE /blocked-schedules[/:id]`, `GET /blocked-schedules/my` | ❌ 401 | ❌ 403 | ✅ | ❌ 403 | Hanya `DOSEN`; Delete cek `lecturer_id = userID`. |
| `POST/DELETE /availabilities[/:id]`, `GET /availabilities/my` | ❌ 401 | ❌ 403 | ✅ | ❌ 403 | Hanya `DOSEN` (tidak ada UI form khusus; `/lecturer/availabilities` redirect). |
| `GET /availabilities/slots` | ❌ 401 | ✅ | ✅ | ✅ | Semua user terautentikasi. |
| `POST /appointments` | ❌ 401 | ❌ 403 | ❌ 403 | ✅ | Khusus `MAHASISWA` (`mhsGuard`). |
| `GET /appointments` | ❌ 401 | ✅ (Semua) | ✅ (Milik Dosen) | ✅ (Milik Mhs) | Scoped otomatis di `AppointmentService.ListAppointments` (`appointment_service.go:214-230`). |
| `GET /appointments/:id` | ❌ 401 | ✅ (Semua) | ✅ (Owner) | ✅ (Owner) | Service cek `role != ADMIN && StudentID != userID && LecturerID != userID` → `403` (`appointment_service.go:207`). |
| `PATCH /appointments/:id/approve`, `/reject` | ❌ 401 | ❌ 403 | ✅ (Owner) | ❌ 403 | `dosenGuard` + repo cek `app.LecturerID == lecturerID`. |
| `PATCH /appointments/:id/cancel` | ❌ 401 | ❌ 403* | ✅ (Owner) | ✅ (Owner) | Repo cek `app.StudentID == userID \|\| app.LecturerID == userID` (`appointment_repo.go:627`). Admin tidak termasuk pemilik → ditolak. |
| `POST /appointments/:id/reschedule` | ❌ 401 | ❌ 403* | ✅ (Owner) | ✅ (Owner) | Service cek `app.StudentID == userID \|\| app.LecturerID == userID` (`reschedule_service.go:75`). |
| `GET /appointments/:id/reschedule` | ❌ 401 | ✅ | ✅ (Owner) | ✅ (Owner) | Service cek `role != ADMIN && StudentID != userID && LecturerID != userID`. |
| `PATCH /reschedule-requests/:id/respond` | ❌ 401 | ❌ 403* | ✅ (Pihak Lawan) | ✅ (Pihak Lawan) | Repo cek peserta appointment **dan** `responderID != req.RequestedBy` (`ErrSameUserResponse`). |
| `GET /notifications`, `/stream`, `PATCH /:id/read`, `/read-all` | ❌ 401 | ✅ | ✅ | ✅ | Scoped pada `userID` dari JWT claims. |
| `POST /feedbacks`, `GET /feedbacks/my`, `POST /feedbacks/upload-intent` | ❌ 401 | ✅ | ✅ | ✅ | Semua user login; `ListMyFeedbacks` scoped pada `userID`. |
| `GET /feedbacks/:id` | ❌ 401 | ✅ (Semua) | ✅ (Owner) | ✅ (Owner) | **Terproteksi**: `FeedbackHandler.GetFeedbackDetail` mengecek `role != RoleAdmin && item.UserID != userID` → `403 FORBIDDEN`. |
| `GET/POST/PUT/DELETE /admin/*` (17 endpoint) | ❌ 401 | ✅ | ❌ 403 | ❌ 403 | Wajib `adminGuard`. Akun `ADMIN` tidak bisa diubah/dihapus via `/admin/users/:id` (`admin_service.go:365,404`). |

---

## 4. Temuan Otorisasi & Risiko Keamanan Berbasis Source Code (`VERIFIED`)

1. **✅ Proteksi IDOR pada `GET /api/v1/feedbacks/:id`** (`backend/internal/handler/feedback_handler.go:71-102`):
   - Handler memverifikasi `role != domain.RoleAdmin && item.UserID != userID` dan mengembalikan `403 FORBIDDEN` bila pengguna non-admin mencoba membaca feedback milik orang lain (`TC-MHS-AUTH-03`).
2. **⚠️ Endpoint Nginx Direct Upload `PUT /storage/upload/(.+)`** (`deploy/nginx/conf.d/app.conf`):
   - `CreateUploadIntent` menerbitkan `upload_token`, namun Nginx WebDAV `location ~ ^/storage/upload/(.+)$` menerima `PUT` secara langsung tanpa memvalidasi `upload_token` atau JWT cookie di level Nginx.
   - **Implikasi QA**: Perlu diuji dan dicatat sebagai observasi keamanan infrastruktur (risiko upload tanpa autentikasi jika endpoint diketahui publik).
3. **✅ Proteksi Akun Admin di User Management (`admin_service.go:47,365,404`)**:
   - Akun ber-role `ADMIN` disembunyikan dari `ListUsers` & `ExportUsersCSV`, serta diblokir dari `UpdateUser` dan `DeleteUser` di backend.
4. **✅ Revokasi Sesi Otomatis saat Akun Dinonaktifkan (`admin_service.go:392-394`)**:
   - Ketika Admin mengubah `is_active` user dari `true` ke `false`, seluruh `user_sessions` user tersebut langsung di-revoke (`RevokeAllUserSessions`).
