# 08 — Database Verification Guide (SQL Queries Aktual)

Dokumen ini menyediakan kumpulan query SQL **read-only** yang 100% menggunakan nama tabel, kolom, dan ENUM aktual pada PostgreSQL 18.7 (`db_cadangan`, schema `public`) untuk memverifikasi hasil setiap skenario QA.

> **Penting**: Selalu jalankan `SET search_path TO public;` di awal sesi `psql` pada Neon DB agar relasi di schema `public` langsung terbaca.

---

## 1. Cek Kondisi Umum & Jumlah Baris Seluruh 14 Tabel

```sql
SET search_path TO public;

SELECT table_name,
       (xpath('/row/cnt/text()', query_to_xml(format('SELECT count(*) AS cnt FROM %I.%I', table_schema, table_name), false, true, '')))[1]::text::int AS row_count
FROM information_schema.tables
WHERE table_schema = 'public'
ORDER BY table_name;
```

---

## 2. Verifikasi Autentikasi, Sesi, & Pengguna (`users`, `user_sessions`, `audit_logs`)

```sql
-- Cek daftar akun dan status aktif
SELECT id, identifier, name, email, role, prodi, is_active, created_at
FROM public.users
ORDER BY role, identifier;

-- Cek sesi login aktif vs dicabut (verifikasi logout / nonaktifkan akun / token reuse)
SELECT u.identifier, u.role, s.ip_address, s.is_revoked, s.expires_at, s.created_at
FROM public.user_sessions s
JOIN public.users u ON u.id = s.user_id
ORDER BY s.created_at DESC
LIMIT 15;

-- Cek catatan audit trail terbaru
SELECT a.action, a.resource, a.status, u.identifier AS actor, a.ip_address, a.metadata, a.created_at
FROM public.audit_logs a
LEFT JOIN public.users u ON u.id = a.user_id
ORDER BY a.created_at DESC
LIMIT 20;
```

---

## 3. Verifikasi Pengaturan Sistem & Kalender Akademik

```sql
-- Cek aturan appointment_rules aktif
SELECT key, value, updated_at
FROM public.system_settings
WHERE key = 'appointment_rules';

-- Cek daftar hari libur / agenda kalender akademik
SELECT id, title, start_date, end_date, type, is_holiday, created_at
FROM public.academic_calendar
ORDER BY start_date ASC;
```

---

## 4. Verifikasi Jadwal Mengajar & Blokir Dosen (Stabilitas Timezone UTC)

```sql
-- Jadwal mengajar rutin dosen
SELECT t.id, u.name AS lecturer_name, t.day_of_week, t.start_time, t.end_time, t.course_name, t.room, t.effective_from, t.effective_until
FROM public.teaching_schedules t
JOIN public.users u ON u.id = t.lecturer_id
ORDER BY u.name, t.day_of_week, t.start_time;

-- Jadwal terblokir dosen (pastikan jam wall-clock UTC sesuai input UI)
SELECT b.id, u.name AS lecturer_name, b.start_at, b.end_at, b.reason, b.created_at
FROM public.lecturer_blocked_schedules b
JOIN public.users u ON u.id = b.lecturer_id
ORDER BY b.start_at ASC;
```

---

## 5. Verifikasi Transaksi Janji Temu, Reschedule, & Proteksi DB

```sql
-- Ringkasan janji temu lengkap beserta nama dosen & mahasiswa
SELECT a.id, a.title, a.category, a.status, a.start_at, a.end_at,
       l.name AS lecturer_name, s.name AS student_name, s.identifier AS student_nim,
       a.rejection_reason, a.cancellation_reason, a.cancelled_by
FROM public.appointments a
JOIN public.users l ON l.id = a.lecturer_id
JOIN public.users s ON s.id = a.student_id
ORDER BY a.created_at DESC;

-- Cek permintaan reschedule
SELECT r.id, r.appointment_id, u.name AS requester, u.role, r.proposed_start_at, r.proposed_end_at,
       r.reason, r.status, r.response_note, r.responded_at
FROM public.appointment_reschedule_requests r
JOIN public.users u ON u.id = r.requested_by
ORDER BY r.created_at DESC;

-- Verifikasi keberadaan GIST exclusion constraint & trigger anti-cancel
SELECT conname, contype, pg_get_constraintdef(oid)
FROM pg_constraint
WHERE conrelid = 'public.appointments'::regclass;

SELECT trigger_name, action_timing, event_manipulation
FROM information_schema.triggers
WHERE event_object_table = 'appointments';
```

---

## 6. Verifikasi Notifikasi & Outbox Worker (`notifications`, `notification_outbox`)

```sql
-- Notifikasi terbaru per pengguna
SELECT n.id, u.identifier, u.role, n.title, n.type, n.is_read, n.link, n.created_at
FROM public.notifications n
JOIN public.users u ON u.id = n.user_id
ORDER BY n.created_at DESC
LIMIT 20;

-- Status pemrosesan Outbox Worker (pastikan status berubah PENDING -> PUBLISHED)
SELECT event_type, status, retry_count, last_error, created_at, processed_at
FROM public.notification_outbox
ORDER BY created_at DESC
LIMIT 20;
```

---

## 7. Verifikasi Masukan Pengguna & Lampiran Storage (`feedbacks`, `feedback_attachments`)

```sql
-- Daftar feedback beserta jumlah lampiran
SELECT f.id, u.identifier, u.role, f.title, f.status, f.admin_notes,
       COUNT(fa.id) AS attachment_count, f.created_at, f.updated_at
FROM public.feedbacks f
JOIN public.users u ON u.id = f.user_id
LEFT JOIN public.feedback_attachments fa ON fa.feedback_id = f.id
GROUP BY f.id, u.identifier, u.role
ORDER BY f.created_at DESC;

-- Rincian URL lampiran yang tersimpan (cocokkan dengan isi /opt/reservasi-dosen/data/storage/uploads/)
SELECT fa.id, fa.feedback_id, fa.file_name, fa.file_type, fa.file_size, fa.file_url, fa.created_at
FROM public.feedback_attachments fa
ORDER BY fa.created_at DESC;
```
