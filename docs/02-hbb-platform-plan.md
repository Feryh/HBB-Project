# Rencana Pembangunan Platform HBB (Home Broadband Operations Analytics)

Dasar: hasil reverse engineering SYIFA (`01-reverse-engineering-syifa.md`).
Label: **[Pasti]** bukti kuat · **[Kemungkinan]** inferensi kuat · **[Menebak]** asumsi yang wajib divalidasi.

---

## 0. Hal yang tidak nyaman, di depan

1. **Menyalin SYIFA 1:1 adalah target yang salah.** [Kemungkinan] 80% usaha ada di pipa data (tiket, inventori OLT/FDT/FAT, presensi, telemetri ONT). Jika sumber data Anda belum punya ID FAT/cluster yang konsisten di tiket, dashboard secantik apa pun akan menunjukkan angka yang salah. **Fase 0 (data contract) bukan opsional.**
2. **Jangan memakai kode/bundle SYIFA.** Bundle hanya dipakai sebagai referensi fungsional. Bangun kode, brand, dan rumus sendiri — terutama jika SYIFA milik mitra/kompetitor.
3. **SYIFA reaktif.** [Pasti] Ia mengukur gangguan setelah tiket dibuat. Jika HBB Project hanya menyamai itu, Anda hanya mengejar. Diferensiasi yang layak: deteksi proaktif dari telemetri OLT/ONT (RX degradation, LOS massal) sebelum pelanggan menelepon.

Pertanyaan yang harus dijawab sebelum Fase 1 (celah terbesar dalam rencana ini):
- Sistem tiket apa yang dipakai (nama, akses API/DB replica)? Apakah setiap tiket punya `customer_id → FAT → FDT → cluster → OLT` yang valid?
- Apakah ada akses ke NMS/OLT (Huawei U2000/NCE, ZTE NetNumen, dsb.) untuk RX level & status ONT?
- Siapa pengguna utama: NOC, manajer area, vendor, atau manajemen? (menentukan prioritas modul)
- Delivery model Anda: inhouse, vendor/MSFO, NRO — sama atau berbeda?

---

## 1. Ruang lingkup modul (prioritas)

| # | Modul | Setara di SYIFA | Prioritas | Alasan |
|---|---|---|---|---|
| M1 | **Fault/WO Analytics** (backlog, aging, MTTR, SLA, area/vendor, RCA) | FAT Loss dashboard | **P0** | Inti operasional; dipakai harian |
| M2 | **Network Inventory & Fault Rate** (FAT population, fault rate, RFS cohort, utilization) | FAT RFS cohorts, Utilization | **P0** | Tanpa ini fault rate tidak bisa dihitung |
| M3 | **Repeat & Hotspot** (FAT/FDT/cluster berulang) | Repeat hotspots | P1 | Sasaran preventive maintenance |
| M4 | **Mass Problem** (dampak OLT/FDT/cluster → pelanggan) | Mass Problem dashboard | P1 | Kritis untuk NOC & komunikasi pelanggan |
| M5 | **Workforce Attendance** (GPS radius, selfie, face check) | MSFO Presensi | P2 | Hanya jika belum ada aplikasi HR/field yang menyediakan |
| M6 | **Capacity Planning & Forecast** (tim R2/R4 vs beban, proyeksi) | Team capacity, Plan R2 | P2 | Butuh data histori ≥ 6 bulan agar bermakna |
| M7 | **Proactive Detection** (RX/LOS dari OLT, auto-ticket, alert WA/Telegram) | Tidak ada | **P1 — diferensiator** | Mengurangi tiket & MTTR, bukan hanya mengukurnya |
| M8 | Admin, RBAC, audit, export | Admin users | P0 | Wajib dari awal (scope vendor/area) |

---

## 2. Arsitektur target

```
 Sumber data                     Ingestion                 Storage                   App
 ┌──────────────────┐   CDC /   ┌──────────────┐   ┌──────────────────────┐   ┌────────────────────┐
 │ Ticketing/CRM    │──API pull─▶│ Connectors   │──▶│ Postgres (OLTP/raw)  │   │ Laravel 12 +       │
 │ Inventory (NRO)  │──daily────▶│ (scheduler,  │   │  raw_* → stg_* → mart│◀──│ Inertia + React 19 │
 │ Field app/presensi│─API──────▶│  retry,      │   │ ClickHouse/Timescale │   │ shadcn + Recharts  │
 │ OLT/NMS (SNMP/   │──5-15 min─▶│  watermark)  │   │  (telemetri ONT/RX)  │   │ MapLibre (GIS)     │
 │  NBI/TR-069)     │           └──────────────┘   │ Redis (cache/snapshot)│   └────────┬───────────┘
 └──────────────────┘                              └──────────────────────┘            │
                                                         ▲                              ▼
                                               Job agregasi (15 menit)        Alert: WA/Telegram/Email
```

Keputusan & alasan:

| Keputusan | Rekomendasi | Level |
|---|---|---|
| App tier | **Laravel 12 + Inertia + React + shadcn/Tailwind v4 + Recharts** — stack yang sama dengan SYIFA terbukti cukup untuk kasus ini, dan starter kit-nya sudah menyediakan auth/2FA | [Kemungkinan] tepat jika tim PHP; jika tim Python, ganti backend ke FastAPI + React SPA |
| Store analitik | Jangan hitung MTTR/aging dari tabel tiket mentah per request. Gunakan **mart table / materialized view** yang di-refresh tiap 15 menit | [Pasti] — SYIFA sendiri memakai snapshot & fallback cache |
| Telemetri | RX level/status ONT bervolume tinggi → **TimescaleDB atau ClickHouse** | [Kemungkinan] |
| Peta | **MapLibre GL** + vector tiles (gratis, tanpa kuota Google) | [Kemungkinan] |
| Zona waktu | Simpan UTC, tampilkan WIB/WITA/WIT per area | [Pasti] kebutuhan nasional |

---

## 3. Model data inti (dirancang ulang, bukan disalin)

```
olt(id, code, site_id, vendor_model, area_id)
fdt(id, code, olt_id, cluster_id, lat, lon)
fat(id, code, fdt_id, cluster_id, capacity_ports, lat, lon)
cluster(id, name, area_id, rfs_date, homepass_count, delivery_model)
area(id, name, city, region, tz)
customer(id, cid, fat_id, ont_sn, package, status)
work_order(id, wo_number, source_system, created_at, confirmed_at, closed_at,
           status, work_type, category, source_type, vendor_id, area_id,
           cluster_id, fdt_id, fat_id, customer_id, done_remark,
           rca_group, closing_reason, delivery_model, is_out_of_scope)
wo_status_history(wo_id, status, changed_at)          -- SYIFA tidak jelas punya ini; penting untuk pending-time/SLA clock
mass_problem(id, mp_number, issue_family, asset_type, asset_id, started_at, recovered_at, root_cause)
mass_problem_impact(mp_id, cluster_id, fdt_id, fat_id, customer_id)
ont_telemetry(ts, ont_sn, rx_dbm, tx_dbm, oper_status)  -- hypertable
technician(id, name, vendor_id, site_id, team_type R2/R4)
attendance(id, technician_id, check_in_at, lat, lon, distance_m, selfie_url, face_result, flags[])
user(id, ...) + user_scope(user_id, workspace, area_id, vendor_id) + abilities
```

Definisi KPI (tulis sebagai dokumen kontrak, disetujui ops sebelum coding):
- **MTTR** = `confirmed_at − created_at` untuk status Done/Closed dengan timestamp valid; laporkan **avg dan P90**; tentukan apakah waktu Pending dikurangi (SYIFA tampaknya tidak — [Menebak]).
- **Fault rate** = unique FAT ber-WO (maintenance) ÷ populasi FAT aktif pada bulan tersebut.
- **Repeat** = FAT/FDT/cluster dengan insiden di ≥ 2 tanggal berbeda dalam window N hari; "persisten" ≥ 5 tanggal.
- **SLA** = % CID solved < 24 jam; hijau ≥ 95%, kuning 90–95%, merah < 90% (selaraskan dengan SLA kontrak Anda).
- **Performance score vendor** = bobot eksplisit, misal 50% solved rate + 30% MTTR ternormalisasi + 20% repeat rate (SYIFA tidak menampilkan bobot; Anda harus transparan agar vendor menerima).

---

## 4. Roadmap bertahap

| Fase | Durasi* | Output | Exit criteria |
|---|---|---|---|
| **F0 Discovery & Data Contract** | 3–4 minggu | Inventaris sumber data, kualitas data (% tiket dengan FAT valid), kamus KPI disetujui, wireframe | ≥ 90% tiket bisa dipetakan ke FAT/cluster, atau rencana perbaikan data disetujui |
| **F1 Foundation + M1 + M8** | 6–8 minggu | Connector tiket, mart WO, dashboard backlog/aging/MTTR/SLA, drilldown, export, RBAC area/vendor, 2FA | Angka MTTR & backlog cocok ±2% dengan laporan manual ops selama 2 minggu |
| **F2 M2 + M3** | 4–6 minggu | Sinkron inventori, fault rate bulanan, RFS cohort (<90 hari vs matang), repeat hotspot, peta | Daftar top-50 FAT repeat divalidasi tim lapangan |
| **F3 M4 + M7 (pilot)** | 6–8 minggu | Mass problem + dampak pelanggan; poller OLT 1 area pilot; alert RX < −27 dBm / LOS massal | Pilot: ≥ 30% gangguan massal terdeteksi sebelum tiket pertama masuk [Menebak target] |
| **F4 M5 + M6** | 6 minggu | Presensi (jika perlu), kapasitas tim & forecast | Forecast bulanan error < 15% pada backtest 3 bulan |
| **F5 Hardening & scale** | berkelanjutan | Multi-region, audit log, observability, SSO | — |

\*Asumsi tim: 1 PM/analis, 2 backend/data engineer, 1 frontend, 0.5 QA. [Menebak] — sesuaikan.

---

## 5. Tim & peran minimum

- **Product owner dari Ops/NOC** (bukan dari IT) — pemilik definisi KPI.
- Data engineer (connector, mart, kualitas data).
- Full-stack Laravel/React.
- Network engineer paruh waktu untuk integrasi OLT/NMS (M7).

---

## 6. Risiko utama

| Risiko | Dampak | Mitigasi |
|---|---|---|
| Tiket tidak punya FAT/cluster yang valid | Fault rate & repeat salah total | F0: ukur kualitas; wajibkan field FAT di aplikasi tiket; pakai mapping CID→FAT dari inventori |
| Akses API sistem sumber ditolak/lambat | Fase molor | Minta read-replica DB atau export terjadwal sejak F0 |
| Definisi KPI diperdebatkan setelah rilis | Kepercayaan hilang | Kamus KPI ditandatangani, setiap angka punya tooltip definisi (SYIFA melakukan ini) |
| Vendor menolak skor | Konflik komersial | Bobot skor transparan & bisa diaudit |
| Data pribadi (selfie, GPS, CID) | Risiko UU PDP | Minimisasi, retensi terbatas, akses berbasis ability, audit log |
| Polling OLT membebani perangkat | Gangguan produksi | Interval konservatif, pakai NBI/NMS bukan SNMP langsung bila ada |

---

## 7. Langkah berikutnya (minggu ini)

1. Jawab 4 pertanyaan di bagian 0.
2. Tarik sampel 1 bulan tiket → ukur % yang punya FAT/cluster valid.
3. Workshop 2 jam dengan Ops untuk menyetujui definisi MTTR, SLA, fault rate, repeat.
4. Setelah itu: scaffold repo (Laravel React starter kit) + connector pertama.
