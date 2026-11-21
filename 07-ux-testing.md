# 07 — UX Testing Documentation (12 Dimensi Pengalaman Pengguna)

Dokumen ini menguji kualitas pengalaman pengguna (*UX Testing*) pada **12 dimensi wajib** yang seluruhnya dipetakan langsung ke komponen aktual dalam source code.

---

## Ringkasan Perbaikan UX Sebelum Testing & Hal yang Diverifikasi (`VERIFIED`)

Isu UX yang sebelumnya terdeteksi pada analisis source code telah **diperbaiki sebelum deploy ulang** sehingga QA memverifikasi kepatuhannya:
1. **Avatar Initials Dinamis di Sidebar** (`LecturerSidebar.tsx`, `StudentSidebar.tsx`): Inisial dihitung dinamis dari `currentUser?.name` (tidak lagi hardcoded `HK`/`AS`).
2. **Toast Konsisten di `MediaDropzone.tsx`**: Notifikasi batas 4 file & format tidak didukung kini menggunakan `useToast()` (tidak lagi `window.alert()`).
3. **Aksesibilitas Tombol Refresh di `/admin/feedbacks`**: Sudah dilengkapi atribut `aria-label="Segarkan daftar masukan"` dan `title`.
4. **Halaman Profil Read-Only vs Backend `PUT /users/me`**: Endpoint pembaruan profil tersedia di backend, namun halaman `/student/profile` dan `/lecturer/profile` bersifat read-only (pastikan tidak ada tombol edit palsu yang membingungkan user).

---

## Skenario Pengujian UX Terstruktur

#### TC-UX-01 — Clarity & Visual Consistency (Label Tombol, Badge Status, & Dropdown Alignment)
| Field | Description |
|---|---|
| **Test Category** | UX — Clarity |
| **Priority** | High |
| **Objective** | Memastikan label tombol, warna badge status, dan gaya dropdown `<select>` seragam serta mudah dipahami tanpa kerancuan |
| **Preconditions** | Akses lintas role (`ADMIN`, `DOSEN`, `MAHASISWA`) |
| **Test Data** | Halaman `/student/feedback`, `/admin/feedbacks`, `/admin/users`, `/student/appointments` |
| **Route** | Multi-route |
| **Expected Result** | Dropdown heading `LMSRichEditor` dan filter status admin memiliki border, radius, & tipografi selaras; warna badge konsisten (Amber = Menunggu, Emerald = Disetujui/Selesai Dibenahi, Rose = Ditolak, Slate = Diarsipkan/Blokir) |
| **Referensi Kode** | `LMSRichEditor.tsx`, `admin/feedbacks/page.tsx`, `AdminFeedbackDetailModal.tsx` |
| **Status QA** | `[ ] PASS` `[ ] FAIL` `[ ] BLOCKED` — Catatan: |

**UX Observation**: Amati apakah pengguna langsung memahami perbedaan status `Menunggu Review`, `Sedang Ditindaklanjuti`, `Selesai Dibenahi`, dan `Diarsipkan`. Catat jika ada teks terpotong atau kontras rendah.

---

#### TC-UX-02 — Feedback, Error Communication, & Toast Consistency
| Field | Description |
|---|---|
| **Test Category** | UX — Feedback & Error Communication |
| **Priority** | High |
| **Objective** | Memastikan setiap aksi sukses/gagal memunculkan feedback yang jelas, berbahasa Indonesia natural, dan konsisten |
| **Preconditions** | Lakukan aksi sukses (simpan pengaturan, kirim feedback) dan aksi gagal (login salah password, upload `.txt` di `MediaDropzone`) |
| **Test Data** | Password salah pada `/auth/login`; file `.txt` pada `/student/feedback` |
| **Route** | `/auth/login`, `/student/feedback`, `/admin/system-settings` |
| **Expected Result** | Error login tampil dalam banner merah + toast yang jelas; catat penggunaan native `alert()` pada `MediaDropzone` sebagai observasi perbaikan UX |
| **Referensi Kode** | `auth/login/page.tsx:187-194`, `MediaDropzone.tsx:106-108` |
| **Status QA** | `[ ] PASS` `[ ] FAIL` `[ ] BLOCKED` — Catatan: |

---

#### TC-UX-03 — Loading Indicators & Interaction State (Double-Submit Prevention)
| Field | Description |
|---|---|
| **Test Category** | UX — Loading & Interaction State |
| **Priority** | High |
| **Objective** | Memastikan setiap form submission menonaktifkan tombol (`disabled={isSubmitting}`) + menampilkan `<Loader2 className="animate-spin" />` untuk mencegah pengiriman ganda |
| **Preconditions** | Gunakan DevTools Network → Throttling `Slow 3G` saat submit form |
| **Test Data** | Submit login, submit `BookingBottomSheet`, submit `FeedbackForm`, simpan `AdminFeedbackDetailModal` |
| **Route** | `/auth/login`, `/student/dashboard`, `/student/feedback`, `/admin/feedbacks` |
| **Expected Result** | Klik cepat 2x pada tombol `Kirim Masukan` atau submit booking hanya menghasilkan 1 request HTTP (tombol langsung disabled & berlabel `"Mengirimkan..."`) |
| **Referensi Kode** | `FeedbackForm.tsx:112-128`, `auth/login/page.tsx:208,230`, `AdminFeedbackDetailModal.tsx:105-123` |
| **Status QA** | `[ ] PASS` `[ ] FAIL` `[ ] BLOCKED` — Catatan: |

---

#### TC-UX-04 — Confirmation Dialogs pada Aksi Destruktif & Kritis
| Field | Description |
|---|---|
| **Test Category** | UX — Confirmation |
| **Priority** | High |
| **Objective** | Memastikan aksi berisiko (Logout, Hapus User, Hapus Kalender, Hapus Jadwal Kuliah/Blokir, Batalkan Janji Temu) selalu meminta konfirmasi modal sebelum dieksekusi |
| **Preconditions** | Login di masing-masing role |
| **Test Data** | Tombol Logout, Hapus agenda kalender, Hapus jadwal blokir, Batalkan janji temu |
| **Route** | Semua halaman dengan aksi hapus/batal/logout |
| **Expected Result** | `ConfirmModal` muncul dengan backdrop blur, tombol `Batal` menutup modal tanpa perubahan, tombol konfirmasi mengeksekusi aksi |
| **Referensi Kode** | `ConfirmModal.tsx`, `AdminSidebar.tsx:24`, `admin/academic-calendar/page.tsx:44` |
| **Status QA** | `[ ] PASS` `[ ] FAIL` `[ ] BLOCKED` — Catatan: |

---

#### TC-UX-05 — Data Consistency (Avatar Sidebar vs Profil & Sinkronisasi Pasca Aksi)
| Field | Description |
|---|---|
| **Test Category** | UX — Data Consistency |
| **Priority** | Medium |
| **Objective** | Memeriksa konsistensi identitas user (nama, NIM/NIP, inisial avatar) dan konsistensi counter tab setelah aksi dilakukan |
| **Preconditions** | Login sebagai `Bima Pratama Putra` (`1203220012`) lalu `Dr. Siti Aminah` (`198807202014042001`) |
| **Test Data** | Akun non-default (bukan Akbar/Hendra) |
| **Route** | `/student/dashboard`, `/student/profile`, `/lecturer/dashboard`, `/lecturer/profile` |
| **Expected Result** | Nama & NIM/NIP cocok di seluruh komponen; **Observasi Khusus**: verifikasi temuan inisial avatar sidebar (`AS` / `HK`) yang tidak berubah mengikuti user login |
| **Referensi Kode** | `StudentSidebar.tsx:83`, `LecturerSidebar.tsx:88`, `student/profile/page.tsx:61-70` |
| **Status QA** | `[ ] PASS` `[ ] FAIL` `[ ] BLOCKED` — Catatan: |

---

#### TC-UX-06 — Empty State & Date Separator Readability
| Field | Description |
|---|---|
| **Test Category** | UX — Empty State & Navigation |
| **Priority** | Medium |
| **Objective** | Memastikan seluruh halaman daftar menampilkan Empty State yang informatif saat database bersih (0 record) dan Date Separator yang rapi saat data terisi |
| **Preconditions** | Kondisi database baru dibersihkan (0 appointments, 0 feedbacks, 0 notifications) |
| **Test Data** | Buka `/student/appointments`, `/student/feedback` (tab Riwayat), `/student/notifications`, `/admin/feedbacks`, `/admin/academic-calendar` |
| **Route** | Multi-route |
| **Expected Result** | Tidak ada tabel kosong mentah tanpa penjelasan atau error `Cannot read properties of null`; muncul ikon + teks panduan ramah |
| **Referensi Kode** | `admin/feedbacks/page.tsx`, `student/appointments/page.tsx`, `lib/dateUtils.ts` |
| **Status QA** | `[ ] PASS` `[ ] FAIL` `[ ] BLOCKED` — Catatan: |

---

#### TC-UX-07 — Responsive Experience (Mobile 375px vs Tablet 768px vs Desktop 1440px)
| Field | Description |
|---|---|
| **Test Category** | UX — Responsive Experience |
| **Priority** | High |
| **Objective** | Memastikan tidak ada horizontal overflow, `BottomNav` tampil di mobile (Mahasiswa & Dosen), drawer hamburger tampil di mobile Admin, dan `BookingBottomSheet` / `LMSRichEditor` nyaman dipakai di layar 375px |
| **Preconditions** | Gunakan Chrome DevTools Device Toolbar (`iPhone SE 375x667`, `iPad 768x1024`, `Desktop 1440x900`) |
| **Test Data** | Seluruh halaman utama 3 role |
| **Route** | `/admin/*`, `/lecturer/*`, `/student/*` |
| **Expected Result** | Tidak ada scrollbar horizontal; bottom navigation tidak menutupi tombol submit (`pb-24` memadai); modal detail dapat di-scroll dengan lancar |
| **Referensi Kode** | `AdminSidebar.tsx:54`, `BottomNav.tsx`, `BookingBottomSheet.tsx` |
| **Status QA** | `[ ] PASS` `[ ] FAIL` `[ ] BLOCKED` — Catatan: |

---

#### TC-UX-08 — Accessibility & Keyboard Navigation (`Tab`, `Enter`, `Escape`, `aria-label`)
| Field | Description |
|---|---|
| **Test Category** | UX — Accessibility |
| **Priority** | Medium |
| **Objective** | Memastikan modal tertutup dengan tombol `Escape`, input form memiliki label jelas, tombol ikon memiliki `aria-label`, dan toggle password dapat dioperasikan via keyboard |
| **Preconditions** | Navigasi menggunakan keyboard (`Tab`, `Shift+Tab`, `Enter`, `Escape`) |
| **Test Data** | `/auth/login` (toggle lihat password), `BookingBottomSheet` (ESC), `AdminFeedbackDetailModal` (ESC untuk Lightbox & modal), `/admin/feedbacks` (tombol refresh) |
| **Route** | `/auth/login`, `/student/dashboard`, `/admin/feedbacks` |
| **Expected Result** | Tombol `Escape` menutup `BookingBottomSheet` dan `AdminFeedbackDetailModal` (termasuk Lightbox secara bertahap); catat kekurangan `aria-label` pada tombol refresh `/admin/feedbacks` |
| **Referensi Kode** | `BookingBottomSheet.tsx:140-149`, `AdminFeedbackDetailModal.tsx:86-95`, `admin/feedbacks/page.tsx:143-152` |
| **Status QA** | `[ ] PASS` `[ ] FAIL` `[ ] BLOCKED` — Catatan: |
