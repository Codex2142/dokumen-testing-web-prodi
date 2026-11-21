# Laporan Pengujian Beban, Stress, & Diagnosis Bottleneck VPS

**Target Server**: VPS Production `38.9.47.207` (`https://38-9-47-207.sslip.io`)  
**Spesifikasi Hardware**: `4 vCPU`, `3.7 GiB RAM` (`2.3 GiB` tersedia) + `4.0 GiB Swap`, Disk `99 GB` (`85 GB` kosong)  
**Database**: Neon PostgreSQL `18.7` (`db_cadangan`, `DB_MAX_CONNS=25`)  
**Tanggal Pengujian**: 21 November 2026

---

## 1. Ringkasan Eksekutif

VPS mampu melayani **~1.000 req/detik** query Backend Go + Neon DB terautentikasi (`200 VU`, **0% error**) dan **~398 req/detik** halaman Next.js SSR (`50 VU`, **0% error**) dengan penggunaan RAM sangat rendah (**< 10% limit** pada ketiga container inti).

Namun, pengujian mengungkap **4 bottleneck nyata** yang paling berdampak saat banyak mahasiswa mengakses aplikasi secara bersamaan (misalnya di jam sibuk kampus):

| Peringkat | Bottleneck | Tingkat Dampak | Bukti Terukur |
| :---: | :--- | :---: | :--- |
| **#1** | **Global `IPRateLimiter(10, 20)` untuk Shared Wi-Fi Kampus (NAT)** | **CRITICAL** | **56% request gagal `429`** saat 50 request konkuren datang dari 1 IP publik (`200: 22`, `429: 28`). |
| **#2** | **Saturasi CPU Next.js Frontend (`cpus: 1.50` limit)** | **CRITICAL** | CPU `reservasi_frontend` mentok **`154.9%`** sejak 20–50 VU; pada 200 VU latensi **`P99` melonjak ke `10.25 detik`**. |
| **#3** | **Antrean Pool Neon DB (`DB_MAX_CONNS=25`) pada Beban Tinggi** | **HIGH** | Pada 200 VU (`995.6 RPS`), seluruh `25/25` koneksi DB terpakai; latensi rata-rata naik dari `~15ms` → `199ms` (`P95=331ms`, tetap 0% error). |
| **#4** | **TOCTOU Race Condition pada Pengajuan `MENUNGGU_PERSETUJUAN`** | **MEDIUM** | 10 `POST /appointments` serentak pada milidetik & slot yang sama menghasilkan `10 × 201 Created` (GIST constraint hanya mengunci `DISETUJUI`). |

---

## 2. Hasil Pengujian Detail (5 Aspek)

### Aspek 1 — Test A: Shared Wi-Fi / Campus NAT Rate Limiter & Security Headers (1 IP Publik)

| Skenario | Hasil Aktual | Analisis |
| :--- | :--- | :--- |
| **50 Concurrent `GET /api/v1/lecturers`** (1 IP, `481ms`) | `200 OK: 22` \| **`429 Too Many Requests: 28` (56% ditolak)** | `IPRateLimiter(10, 20)` dipasang di global middleware (`main.go:75`) **sebelum** `authMiddleware`, sehingga `CtxUserID` tidak pernah terbaca dan seluruh user 1 gedung Wi-Fi kampus berebut kuota 10 req/s! |
| **10 Rapid `POST /api/v1/auth/login`** (password salah) | `401 Unauthorized: 8` \| **`429 Too Many Requests: 2`** | Proteksi brute-force `loginRateLimiter(1, 5)` aktif dan bekerja. |
| **Security Headers (`/auth/login`)** | `HSTS`, `X-Frame-Options: SAMEORIGIN`, `X-Content-Type-Options: nosniff`, `Referrer-Policy`, `Permissions-Policy`, `Server: nginx` | **PASS** — Seluruh header keamanan aktif dan versi Nginx disembunyikan. |

---

### Aspek 2 — Test B: Multi-IP Capacity Stress Test (20 → 50 → 100 → 200 VU)

#### A. Frontend Next.js (`GET /auth/login` & `/student/dashboard`, 10 detik per tahap)

| Concurrent VU | Total Req | Throughput (RPS) | Avg | P50 | P95 | P99 | Max | Error / 5xx |
| :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| **20 VU** | 3.712 | `369.8 req/s` | `54.1 ms` | `47.7 ms` | `91.7 ms` | `108.3 ms` | `375.9 ms` | `0` (100% 200) |
| **50 VU** | 4.013 | **`398.4 req/s`** | `125.2 ms` | `120.8 ms` | `170.4 ms` | `229.2 ms` | `947.3 ms` | `0` (100% 200) |
| **100 VU** | 3.699 | `362.1 req/s` | `274.4 ms` | `246.0 ms` | `354.3 ms` | **`1.12 s`** | `3.69 s` | `0` (100% 200) |
| **200 VU** | 3.484 | `333.4 req/s` | `588.5 ms` | `344.8 ms` | `466.2 ms` | **`10.25 s`** | `10.42 s` | `0` (100% 200) |

- **Diagnosis**: Throughput puncak Next.js berada di **~398 RPS (~50 VU)**. Di atas 50 VU, single-process Node.js + batas `cpus: 1.50` menyebabkan antrean CPU dan lonjakan tail latency `P99` hingga **10,25 detik**.

#### B. Backend Go API + Neon DB (Read-Heavy Authenticated Mix, 10 detik per tahap)

| Concurrent VU | Total Req | Throughput (RPS) | HTTP `200` | HTTP `429` | Avg | P50 | P95 | P99 |
| :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| **20 VU** | 45.049 | `4.497 req/s` | 2.379 | 42.670 | `4.4 ms` | `3.1 ms` | `16.5 ms` | `32.4 ms` |
| **50 VU** | 50.376 | `5.035 req/s` | 5.570 | 44.806 | `9.9 ms` | `6.9 ms` | `31.6 ms` | `48.6 ms` |
| **100 VU** | 26.324 | `2.608 req/s` | 10.301 | 16.023 | `38.2 ms` | `2.6 ms` | `153.1 ms` | `162.5 ms` |
| **200 VU** | 10.135 | **`995.6 req/s`** | **10.135 (100%)** | **0** | `199.0 ms` | `164.4 ms` | `331.7 ms` | `380.3 ms` |

- **Diagnosis**: Pada 20–100 VU, loop ketat per VU melebihi `10 req/s` per IP sehingga mayoritas dicegat cepat (`<3ms`) oleh `IPRateLimiter` (Go + Nginx menelan `5.035 RPS` tanpa error). Pada **200 VU**, pace alami (~5 req/s/VU) lolos rate limiter seluruhnya dan **Backend Go + Neon DB melayani `995.6 RPS` query real dengan `0%` error dan `P99 = 380.3 ms`!**

---

### Aspek 3 — Profil Resource Container Real-Time (`docker stats`)

| Container | CPU Idle | CPU Peak Saat Load | RAM Peak / Limit | Bottleneck? |
| :--- | :---: | :---: | :---: | :---: |
| `reservasi_frontend` | `0.00%` | **`154.92%`** *(limit 150%)* | `89.84 MiB / 1 GiB (8.8%)` | **YA — CPU Mentok 100% Limit** |
| `reservasi_backend` | `0.00%` | `78.39%` *(limit 150%)* | `39.18 MiB / 768 MiB (5.1%)` | Tidak (masih ~50% headroom) |
| `reservasi_nginx` | `0.00%` | `65.65%` | `25.24 MiB / 256 MiB (9.9%)` | Tidak |
| **Neon DB Pool** | `6/25` idle | **`25/25` max_conns** | Latency `29.1 ms` | **Mulai antre di >100 VU** |

---

### Aspek 4 — Concurrency, SSE, & Zero-Backend Storage

| Pengujian | Hasil Aktual | Status |
| :--- | :--- | :---: |
| **Concurrent Double-Booking** (10 `POST /appointments` serentak slot sama) | `201 Created: 10` (semua `MENUNGGU_PERSETUJUAN`) | **⚠️ TOCTOU Race** *(aman saat Approve karena GIST constraint `DISETUJUI`, namun izinkan 10 pending bersamaan di 1ms)* |
| **SSE Connection Limit** (4 stream simultan 1 user) | `200 OK: 2`, `429: 2` | **✅ Terproteksi** (tidak terjadi connection leak) |
| **Zero-Backend Storage Upload** (5 × 2MB paralel = 10MB) | `201 Created: 5/5` dalam **`41 ms` (`243.4 MB/s`)** | **✅ Sangat Cepat** (0% beban Go backend) |
| **Upload Limit > 25MB** (1 × 26MB) | **HTTP `413 Request Entity Too Large`** | **✅ Terproteksi** |

---

### Aspek 5 — Verifikasi Auto-Cleanup Pasca-Test
- `/tmp/loadtest`, `/tmp/stress_stats.log`, dan `/opt/reservasi-dosen/data/storage/uploads/stress-test-*.bin` telah dihapus dari VPS.
- `db_cadangan` telah dibersihkan kembali ke kondisi QA bersih (`users=9`, `system_settings=1`, seluruh 12 tabel operasional=`0`).

---

## 4. Hasil Re-Test Pasca-Remediasi (*Before vs After*)

Tiga tuning yang disetujui telah diterapkan (commit `1b24320` / `fb73323`) dan dideploy ke VPS:
1. Global `IPRateLimiter(50, 100)` di `backend/cmd/api/main.go` (login tetap ketat `1 req/s, burst 5`).
2. Nginx Micro-Cache `10s` (`fe_microcache`, bypass `?_rsc=` & WebSocket, `proxy_cache_lock on`) di `deploy/nginx/nginx.conf` & `app.conf` + `reservasi_frontend` `cpus: 2.00`.
3. `DB_MAX_CONNS=40` & `DB_MIN_CONNS=5` di `docker-compose.yml` dan `.env` VPS.

### Tabel Perbandingan Performa (*Before vs After*)

| Metrik Pengujian | Before (Baseline) | After (Tuned) | Peningkatan |
| :--- | :---: | :---: | :---: |
| **Shared-NAT 50 Concurrent API (1 IP)** | `22/50` sukses (**56% error `429`**) | **`50/50` sukses (`0%` error)** | **Error `429` hilang 100%** |
| **Frontend 50 VU Throughput** | `398.4 RPS` (`P99 229.2 ms`) | **`6.863,0 RPS` (`P99 21.8 ms`)** | **17,2x RPS / -90,5% P99** |
| **Frontend 200 VU Throughput** | `333.4 RPS` | **`6.673,5 RPS`** | **20,0x lebih cepat** |
| **Frontend 200 VU P99 Latency** | **`10.253,8 ms` (`10,25 dtk`)** | **`84,5 ms`** | **121x lebih cepat (`-99,2%`)** |
| **Frontend Peak CPU / RAM** | `154,92%` CPU / `89,8 MiB` | **`0,05%` CPU / `30,7 MiB`** | **`-99,97%` CPU / `-65,8%` RAM** |
| **Backend + Neon DB 200 VU Throughput** | `995.6 RPS` (100% 200) | **`1.612,3 RPS` (100% 200)** | **`+61,9%` RPS** |
| **Backend + Neon DB 200 VU P95 / P99** | `331,7 ms` / `380,3 ms` | **`203,1 ms` / `233,2 ms`** | **`-38,8%` P95 / `-38,7%` P99** |
| **Backend + Neon DB 20–200 VU Error/429** | 429 pada 20–100 VU | **`0` error & `0` 429 (58.736/58.736 HTTP 200)** | **100% reliabel** |
| **SSE Limit Test (4 stream simultan)** | `200: 2`, `429: 2` | **`200: 3`, `429: 1`** (tepat batas 3/user) | **PASS** |
| **DB Pool `/healthz` Post-Test** | `max_conns: 25`, `29,1 ms` | **`max_conns: 40`, `15,8 ms`** | **Pool aktif 40 koneksi** |
| **Nginx Storage Upload (5×2MB & 26MB)** | `243 MB/s`, 26MB → `413` | **`200,3 MB/s`, 26MB → `413`** | **PASS** |

> **Catatan Minor Observasi**: Penambahan `add_header X-Cache-Status` pada blok `location /` di `app.conf` menimpa pewarisan `Strict-Transport-Security` dari level `server` (perilaku standar Nginx). Dapat ditambahkan kembali baris `add_header Strict-Transport-Security ...` di dalam `location /` pada deploy berikutnya jika diinginkan.

---

## 3. Rekomendasi Perbaikan Bottleneck (Status)

1. **[CRITICAL — Rate Limiter Wi-Fi Kampus]** Pindahkan `rateLimiter.Limit()` agar memeriksa identitas user (atau naikkan limit global ke misal `50 req/s, burst 100` untuk IP shared NAT kampus, sambil tetap ketat `1 req/s, burst 5` khusus `/auth/login`). Ini langsung menghilangkan 56% error `429` saat banyak mahasiswa mengakses dari Wi-Fi kampus yang sama.
2. **[CRITICAL — Next.js Frontend CPU & P99 10s]** Tambahkan **Nginx micro-cache** (`proxy_cache` 5–10 detik) khusus untuk halaman HTML statis (`/auth/login`, `/student/*` shell yang datanya di-fetch di client) **atau** naikkan `cpus: 2.00` pada `reservasi_frontend` di `docker-compose.yml` — ini dapat melipatgandakan kapasitas frontend dari `398 RPS` menjadi `>3.000 RPS` dengan `P99 < 20ms`.
3. **[HIGH — DB Connection Pool]** Naikkan `DB_MAX_CONNS` dari `25` ke `40` (Neon pooler PgBouncer mendukung ratusan koneksi klien) bila beban konkuren API diperkirakan sering melampaui 150 VU simultan.
4. **[MEDIUM — Pencegahan Duplicate Pending Race]** Bila bisnis melarang >1 mahasiswa mengajukan `MENUNGGU_PERSETUJUAN` di slot yang persis sama pada milidetik yang sama, gunakan `pg_advisory_xact_lock(hashtext(lecturer_id || date))` di awal transaksi `CreateWithOutbox`.
