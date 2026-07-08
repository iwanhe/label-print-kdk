🇮🇩 Bahasa Indonesia | [🇬🇧 English](README.en.md)

# Label Print KDK (App ID 102)

Export aplikasi **Oracle APEX** untuk sistem pencetakan label barcode bahan baku (daun tembakau/cengkeh) milik **PT Karunia Daun Kencana (KDK)** — bagian dari Nojorono Group. Aplikasi ini menangani proses *staging* data timbangan, generate nomor barcode, pencetakan label, hingga pembacaan ulang (scan) barcode di gudang.

## Informasi Aplikasi

| Item | Nilai |
|---|---|
| Application ID | `102` |
| Nama Aplikasi | Label Print KDK |
| Alias | `LABEL-PRINT-KDK` |
| Schema/Owner | `KDK` |
| APEX Version | 24.2.0 (export format `2024.11.30`) |
| Flow Version | Release 1.0 |
| Bahasa | English (`p_flow_language => 'en'`) |
| Autentikasi | Oracle APEX Accounts |

## Struktur Folder

```
f102/
├── install.sql                        # Entry point instalasi aplikasi
└── application/
    ├── create_application.sql         # Definisi utama aplikasi
    ├── set_environment.sql            # Konteks workspace/app id/owner
    ├── comments.sql
    ├── plugin_settings.sql
    ├── user_interfaces.sql / user_interfaces/
    ├── pages/                         # Definisi tiap halaman (page_XXXXX.sql)
    ├── shared_components/
    │   ├── navigation/                # List menu (navigation_menu, navigation_bar, home, access_control)
    │   ├── security/
    │   │   ├── authentications/       # oracle_apex_accounts
    │   │   ├── authorizations/        # 19 authorization scheme
    │   │   └── app_access_control/    # 6 role ACL
    │   ├── plugins/
    │   │   ├── process_type/          # AOP (APEX Office Print) process
    │   │   ├── dynamic_action/        # AOP dynamic action
    │   │   └── region_type/           # BarcodeReader (apexbcreader)
    │   ├── data_loads/                # Data Load Definitions (Excel upload)
    │   ├── files/                     # Icon, logo, template docx/xlsx
    │   └── globalization/
    └── deployment/
        ├── definition.sql             # Deinstall script (DROP objects)
        ├── buildoptions.sql
        ├── checks.sql
        └── install/
            ├── install_table.sql      # DDL tabel
            ├── install_view.sql       # DDL view
            ├── install_function.sql   # DDL function
            ├── install_procedure.sql  # DDL procedure
            └── install_create_user_acces_role_psk.sql
```

## Fitur Utama (Halaman Aplikasi)

| Page ID | Nama Halaman |
|---|---|
| 0 | Global Page |
| 1 | Home |
| 2 | Tips Penggunaan Singkat |
| 3 | Form Add User Application |
| 4 | Load Dan Cetak Barcode Lama |
| 6 | History Cetak Barcode Individual |
| 8 | Form Load Barcode |
| 9 | Form Load Barcode 2024 |
| 10 | Barcode Info |
| 12 | Master Serial Barcode |
| 13 | Form Master Serial Barcode |
| 14 | Detail Load Dan Cetak Barcode Lama |
| 15 | Form Load Barcode Info |
| 16 | Print Selected Barcode |
| 17 | Form Barcode |
| 18 | Form Barcode Info |
| 19 | Read Barcode |
| 20 | Scanner |
| 9999 | Login Page |
| 10000 | Administration |
| 10010 | Configure Access Control |
| 10011–10012 | Manage User Access |
| 10013–10014 | Add Multiple Users (Step 1 & 2) |

Alur bisnis utamanya secara garis besar:
1. **Load data** timbangan bahan baku (via Excel — `load_staging_barcode` / `load_staging_barcode_2024`) ke tabel staging.
2. **Generate barcode** dari data staging berdasarkan kombinasi atribut (`MAT_FORM`, `COMPANY_CODE`, `ORIGIN`, `QUALITY`, `SUPPLIER`, `CROP_YEAR`, dll) melalui fungsi `ARRANGE_BARCODE`.
3. **Cetak label** barcode terpilih memakai plugin **APEX Office Print (AOP)**.
4. **Scan/Read Barcode** ulang di gudang menggunakan plugin **BarcodeReader** (kamera device).
5. **Master Serial Barcode** dan **Barcode Info** untuk pengelolaan data master pendukung.
6. **Administration** — pengaturan hak akses, role, dan user aplikasi.

## Objek Database

**Tabel:**
- `XTD_KDK_LP_STAGING_BARCODE` — data staging hasil load Excel (mat form, origin, quality, supplier, crop year, gross/tare/net, total barcode, prefix, dll).
- `XTD_KDK_LP_BARCODE` — barcode individual hasil generate, terhubung ke staging.
- `XTD_KDK_LP_BARCODE_INFO` — data master informasi tambahan barcode.
- `XTD_KDK_LP_JOB` — antrian job asynchronous (status `PENDING`, `attempt_count`, `payload`, `max_attempt`).
- `XTD_KDK_LP_JOB_RUN` — histori eksekusi job (progress %, total/processed/failed items, error code/message).

**View:**
- `V_KDK_LP_BARCODE`

**Function:**
- `ARRANGE_BARCODE(p_id, p_is_2024)` — menyusun format nomor barcode dari data staging.
- `IS_TRANSIENT_ERROR` — deteksi error yang bersifat sementara (untuk retry job).

**Procedure:**
- `BARCODE_JOB_WORKER` — worker pemroses antrian job generate barcode.
- `PROCESS_KDK_LP_BARCODE_JOB` — logika inti pemrosesan satu job.
- `MARK_FAILED_JOB_TO_ATTEMPT` — menandai job gagal untuk retry.
- `RECOVER_STALE_STAGING` — recovery data staging yang macet/stale.

> Pola job queue (`XTD_KDK_LP_JOB` + `JOB_RUN` + worker/retry procedure) mengindikasikan proses generate barcode berjalan secara asynchronous dengan mekanisme retry dan tracking progress.

## Keamanan & Hak Akses

**Authentication scheme:** Oracle APEX Accounts.

**ACL Role (Access Control):**
| Static ID | Nama Role |
|---|---|
| `DIREKTUR` | Direktur |
| `ADMINISTRATOR` | Administrator |
| `ADMIN_WH` | Admin Warehouse |
| `WH` | Warehouse |
| `CONTRIBUTOR` | Contributor |
| `READER` | Reader |

**Authorization Scheme** (19 scheme, berbasis aksi/menu): Administration Rights, Contribution Rights, Reader Rights, Load_barcode, Load_barcode_2024, Load_barcode_info, Edit_barcode, Delete_barcode, Generate_barcode, Retry_generate_barcode, Print_barcode, Read_barcode, Barcode_info, Update_barcode_info, Delete_barcode_info, Master_serial_barcode, Create/Update/Delete_master_serial_barcode, Setup_dan_master_data.

## Plugin yang Digunakan

| Plugin | Tipe | Fungsi |
|---|---|---|
| `apexbcreader` (BarcodeReader) | Region Type | Membaca/scan barcode via kamera perangkat |
| `be_apexrnd_aop` (UC - APEX Office Print) | Process Type | Generate dokumen/label cetak (PDF) |
| `be_apexrnd_aop_da` / `be_apexrnd_aop_convert_da` | Dynamic Action | Trigger proses AOP dari sisi client |

## Data Load Definitions

- `load_staging_barcode` — format load Excel `format_load_barcode_label_print.xlsx`
- `load_staging_barcode_2024` — format load Excel `format_load_barcode_label_print_2024.xlsx`
- `load_barcode_info` — format load Excel `format_load_barcode_info.xlsx`

File pendukung lain: icon aplikasi (32/144/192/256/512 px), logo (`label_print_logo`), animasi loading, dan template `test_templete.docx` (kemungkinan template cetak AOP).

## Instalasi

Jalankan sebagai schema `KDK` (atau user dengan role `APEX_ADMINISTRATOR_ROLE`) menggunakan SQLcl/SQL*Plus:

```sql
@install.sql
```

Script ini akan berurutan menjalankan: set environment → hapus aplikasi lama (bila ada) → create application → shared components (navigation, files, plugin settings, authorization, ACL, dsb.) → pages → deployment (tabel, view, function, procedure, index).

> **Catatan:** File ini adalah *export otomatis* dari Oracle APEX. Sesuai standar Oracle, sebaiknya tidak diedit manual — perubahan dilakukan melalui APEX Builder lalu di-export ulang (atau via tooling ADT/Liquibase pada workflow PANDAWA).

## Prasyarat

- Oracle Database (kompatibel dengan APEX 24.2, mendukung `IDENTITY` column, `TIMESTAMP(6)`).
- Oracle APEX 24.2 ter-install pada workspace tujuan.
- Plugin **APEX Office Print (AOP)** ter-registrasi di instance untuk fitur cetak label.
- Akses kamera device untuk fitur scan barcode (plugin `apexbcreader`) — biasanya dijalankan dari perangkat mobile/tablet di gudang.
