# Reverse Engineering — SYIFA (Systemic Impairment & Fault Analytics)

Target: `https://syifa.morarepublic.co.id/?source=myrepublic&delivery_model=all&date_scope=all`
Tanggal analisis: 2026-11-18

Label kepercayaan:
- **[Pasti]** terlihat langsung di HTML/JS bundle publik.
- **[Kemungkinan]** kesimpulan kuat dari beberapa bukti.
- **[Menebak]** pengisian celah, perlu divalidasi.

## 0. Batasan analisis (baca dulu)

- **[Pasti]** Semua URL dashboard redirect ke `/login`. Tidak ada akun, jadi **tidak ada data asli yang terlihat**.
- **[Pasti]** Analisis hanya memakai aset frontend yang dikirim server ke browser publik: HTML login, `app-*.js`, dan chunk halaman (`fat-loss-dashboard`, `mass-problem-dashboard`, `msfo-presensi`). Tidak ada percobaan bypass autentikasi.
- **[Kemungkinan]** Konsekuensinya: struktur UI, endpoint API, field data, dan filter cukup terang. Sumber data backend, skema DB, ETL, dan logika skor **tidak terlihat** dan harus dirancang sendiri.

## 1. Identitas & konteks bisnis

| Temuan | Bukti | Level |
|---|---|---|
| Nama internal app: **"FAT Loss Management"**, brand UI: SYIFA | Inertia props `name:"FAT Loss Management"` | [Pasti] |
| Operator: MoraRepublic (Moratelindo × MyRepublic, grup Sinarmas) | Logo, footer | [Pasti] |
| Fokus: gangguan jaringan FTTH di level **FAT/FDT/Cluster/OLT**, bukan pelanggan individual | Field `affected_fats`, `impacted_fdts`, `olt_code`, `homepass_count` | [Pasti] |
| Wilayah awal: Jabodetabek (Jakarta Barat/Pusat/Selatan/Timur/Utara terlihat sebagai opsi) | Label kota di bundle | [Pasti] |
| Zona waktu operasional WIB (`Asia/Jakarta`) | String di bundle | [Pasti] |

## 2. Tech stack

| Layer | Teknologi | Level |
|---|---|---|
| Backend | **Laravel** (Inertia server adapter, Fortify: 2FA, confirm-password) | [Kemungkinan] — pola `data-page`, `two-factor-challenge`, `confirm-password` identik dengan Laravel React Starter Kit |
| Frontend bridge | **Inertia.js** (`data-page`, `sharedProps`, `version`) | [Pasti] |
| UI | **React 19 + React Compiler** (`react.memo_cache_sentinel`), TypeScript (`.tsx`) | [Pasti] |
| Komponen | **shadcn/ui** (`data-slot="card"`, Radix popover/dialog), **Tailwind CSS v4** (`oklab`), **lucide-react** | [Pasti] |
| Chart | **Recharts** (`AreaChart`, `ComposedChart`, `ReferenceArea`) | [Pasti] |
| UX util | `sonner` (toast), `nprogress` (bar loading), dark/light mode | [Pasti] |
| Build | **Vite** (`/build/assets/*-hash.js`, `__vite__mapDeps`) | [Pasti] |
| Peta | Ada fitur "Network coverage and work-order locations" — library tidak teridentifikasi | [Menebak] Leaflet/MapLibre |
| Data store / ETL | Tidak terlihat | [Menebak] Postgres/MySQL + job terjadwal |

## 3. Peta halaman (Inertia pages)

**[Pasti]** dari resolver di `app-*.js`:

```
pages/auth/dashboard-login.tsx        login username+password (akun dibuat admin, tanpa self-signup / reset)
pages/auth/two-factor-challenge.tsx   2FA TOTP
pages/auth/confirm-password.tsx
pages/auth/required-password.tsx      paksa ganti password
pages/fat-loss-dashboard.tsx          MODUL UTAMA (~565 KB minified)
pages/mass-problem-dashboard.tsx      modul gangguan massal
pages/msfo-presensi.tsx               presensi teknisi lapangan
pages/admin/users.tsx                 manajemen user + scope
pages/settings/{profile,security,appearance}.tsx
pages/dashboard.tsx                   placeholder starter kit (tidak dipakai)
```

## 4. Kontrak API (endpoint yang dipanggil frontend)

**[Pasti]** — semua JSON, dipanggil via fetch dengan query string filter:

| Modul | Endpoint | Fungsi (inferensi dari label) |
|---|---|---|
| FAT Loss | `GET /api/dashboard` | KPI headline, status mix, area, kategori |
| | `GET /api/drilldown` | daftar WO per bucket/klik chart |
| | `GET /api/records` | pencarian WO / Customer ID / ONT / serial number |
| | `GET /api/export` | CSV / XLSX (dibatasi ability `exportRawData`) |
| | `GET /api/active-aging-matrix` | matriks umur backlog × kategori |
| | `GET /api/monthly-work-orders` | tren bulanan opened vs solved |
| | `GET /api/repeat-hotspots` | Cluster/FDT/FAT yang berulang (FAT 5+ hari) |
| | `GET /api/fat-rfs-cohorts` | **FAT Loss: New RFS (<90 hari) vs RFS +90 Days** |
| | `GET /api/impact-map/drilldown` | peta WO & coverage |
| | `GET /api/utilization` | History FAT Utilization (homepass terpakai) |
| | `GET /api/team-capacity-v2`, `/api/team-roster` | kapasitas tim R2/R4 vs beban |
| | `GET /api/team-capacity/plan-r2-year-end-2026` | proyeksi kebutuhan tim s/d akhir 2026 |
| Mass Problem | `GET /api/mass-problem/{dashboard,drilldown,export}` | MP aktif, dampak cluster/FDT/FAT/pelanggan, OLT mapping |
| MSFO | `GET /api/msfo/presensi`, `/api/msfo/presensi/export.csv` | presensi teknisi (GPS, radius, selfie, face verification) |

## 5. Dimensi filter global

**[Pasti]** dari state filter (`deliveryModels, projects, vendors, areas, cityNames, workTypes, categoryTypes, sourceTypes, ticketStatuses`):

| Parameter | Nilai |
|---|---|
| `source` | `myrepublic` (live), `mora` ("MORA (Under Development)", sumber Mora CRM) |
| `delivery_model` | `all` = NRO + MSFO + Inhouse; `nro` ("Clusters RFS under 90 days"), `msfo`, `inhouse`, Partner (Mora CRM) |
| `date_scope` | `today`, `month` (MTD), `all`, custom range, `last_3/6/12`, `last_date` |
| status WO | `open`, `in_progress`, `pending`, `done`, `closed`, `cancelled` |
| lain | vendor (bisa dikunci admin), area (maks 5 untuk perbandingan), kota, project, work type (Maintenance/Installation/Dismantle), kategori, source type |

**[Kemungkinan]** "Delivery model" = siapa yang mengerjakan: **NRO** (tim rollout, menangani cluster yang baru RFS < 90 hari / masa garansi), **MSFO** (managed service field operation / vendor), **Inhouse** (teknisi internal).

## 6. Model data (field yang muncul di payload)

**[Pasti]** nama field; **[Kemungkinan]** pengelompokan entitas:

- **WorkOrder**: `wo_number, created_at, ticket_status, work_type, category, source_type, vendor, area, city_name, cluster, fdt, fat, customer_id, done_remark, closing_reason, reason_key, resolution_seconds, age_seconds, solved_date, delivery_flag, out_of_scope`
- **Perangkat pelanggan**: `ont_status, rx_level, serial_number` (dipakai di lookup "Check the Customer ID, ONT status, or serial number"; label "Auto-recovery RX/ONT")
- **Inventori jaringan (NRO)**: OLT → FDT → FAT → Homepass; `cluster_id, rfs_date, homepass_count, olt_code, olt_mapping_status` — "NRO infrastructure is refreshed once every 24 hours"
- **MassProblem**: `MP number, issue_family, link_type, impacted_clusters/fdts/fats/customers, cid_impacted, service_impact, known_root_cause_wo`
- **Repeat**: `repeat_count, repeat_episodes, first_seen, last_seen, median_gap_days`
- **Presensi**: `check_in, check_out, distance, radius (500 m), gps_invalid, selfie_missing, face_failed, future_time, site/zone/vendor`
- **Kapasitas**: `R2 teams, R4 teams, joined teams, fulfillment gap, target_utilization, capacity buffer, solved/team/day`

## 7. KPI & logika bisnis yang terlihat

| KPI | Definisi (dari teks UI) | Level |
|---|---|---|
| Active WO / Backlog | Open + In Progress + Pending saat ini | [Pasti] |
| MTTR | WO dibuat → confirm time (Done/Closed) valid; avg + P90; Mora pakai waktu tutup tiket | [Pasti] |
| Aging bucket | <1d, 1–3d, 3–7d, 7–14d, 14–30d, 30d+; versi harian & per jam | [Pasti] |
| Solved CID <24h / ≥24h | Customer ID terselesaikan di bawah/atas 24 jam | [Pasti] |
| SLA | Hijau ≥ 95.0%, Merah < 90.0% | [Pasti] |
| Fault Rate / FAT impact rate | Unique FAT ber-WO ÷ populasi FAT bulan itu; ada versi "Repeat" | [Pasti] |
| FAT Loss New RFS vs RFS+90 | Gangguan di cluster baru vs cluster matang — indikator kualitas rollout | [Pasti] |
| Repeat | 1 repeat per aset per tanggal insiden tambahan; "FAT 5+ Days" = persisten | [Pasti] |
| Performance score vendor | Gabungan solved rate tinggi + MTTR pendek | [Pasti] (bobot tidak terlihat) |
| Remark classification | Done remark diklasifikasi ke grup RCA; "No Issue Found" dideteksi dari teks | [Kemungkinan] rule-based keyword |
| Capacity plan R2 | Required teams = beban (unique FAT / repeat) ÷ solved/team/day, + buffer %, dibanding existing roster | [Kemungkinan] |
| Forecast | "Observed + forecast remaining" untuk bulan berjalan | [Pasti] ada; metode [Menebak] run-rate |

Penyegaran data: Mass Problem "Updates every 15 minutes"; presensi auto-refresh 5 menit; NRO 24 jam; fallback "most recent cached records are still shown" / "Saved data / snapshot". **[Kemungkinan]** backend menarik data dari sistem sumber secara terjadwal ke tabel snapshot, bukan query live ke sistem tiket.

## 8. Keamanan & RBAC

**[Pasti]**:
- Ability: `exportRawData`, `manageUsers`, `accessMassProblem`, `accessMsfoAttendance`
- Scope data per user: `allowed_workspaces`, `allowed_areas`, `vendor_scope` ("Scope vendor dikunci oleh administrator")
- Admin bisa **preview sebagai role lain** (`preview_role`, "Return to Admin")
- 2FA, paksa ganti password, session expiry, akun hanya dibuat admin, CSP nonce di script

## 9. Kesimpulan untuk HBB Project

1. **[Kemungkinan]** Nilai SYIFA ada di **integrasi data** (tiket MyRepublic, Mora CRM, inventori NRO, aplikasi presensi MSFO, status ONT/RX), bukan di UI-nya. UI bisa dibangun ulang dalam beberapa minggu; integrasi datanya yang memakan bulan.
2. **[Pasti]** SYIFA adalah alat **reaktif/analitik** (apa yang sudah rusak, seberapa cepat diperbaiki). Tidak terlihat fitur prediktif (degradasi RX sebelum pelanggan komplain), alerting push, atau ticket automation. Di situ ruang diferensiasi.
3. **[Pasti]** Modul "MORA" masih *Under Development* — sumber data kedua belum matang bahkan di sistem mereka.
