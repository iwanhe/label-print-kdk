[🇮🇩 Bahasa Indonesia](README.id.md) | 🇬🇧 English

# Label Print KDK (App ID 102)

Export of an **Oracle APEX** application for printing barcode labels on raw materials (tobacco leaf/clove) belonging to **PT Karunia Daun Kencana (KDK)** — part of Nojorono Group. The application handles staging of weighing data, barcode number generation, label printing, and barcode re-reading (scanning) at the warehouse.

## Application Info

| Item | Value |
|---|---|
| Application ID | `102` |
| Application Name | Label Print KDK |
| Alias | `LABEL-PRINT-KDK` |
| Schema/Owner | `KDK` |
| APEX Version | 24.2.0 (export format `2024.11.30`) |
| Flow Version | Release 1.0 |
| Language | English (`p_flow_language => 'en'`) |
| Authentication | Oracle APEX Accounts |

## Folder Structure

```
f102/
├── install.sql                        # Application installation entry point
└── application/
    ├── create_application.sql         # Main application definition
    ├── set_environment.sql            # Workspace/app id/owner context
    ├── comments.sql
    ├── plugin_settings.sql
    ├── user_interfaces.sql / user_interfaces/
    ├── pages/                         # Page definitions (page_XXXXX.sql)
    ├── shared_components/
    │   ├── navigation/                # Menu lists (navigation_menu, navigation_bar, home, access_control)
    │   ├── security/
    │   │   ├── authentications/       # oracle_apex_accounts
    │   │   ├── authorizations/        # 19 authorization schemes
    │   │   └── app_access_control/    # 6 ACL roles
    │   ├── plugins/
    │   │   ├── process_type/          # AOP (APEX Office Print) process
    │   │   ├── dynamic_action/        # AOP dynamic action
    │   │   └── region_type/           # BarcodeReader (apexbcreader)
    │   ├── data_loads/                # Data Load Definitions (Excel upload)
    │   ├── files/                     # Icons, logo, docx/xlsx templates
    │   └── globalization/
    └── deployment/
        ├── definition.sql             # Deinstall script (DROP objects)
        ├── buildoptions.sql
        ├── checks.sql
        └── install/
            ├── install_table.sql      # Table DDL
            ├── install_view.sql       # View DDL
            ├── install_function.sql   # Function DDL
            ├── install_procedure.sql  # Procedure DDL
            └── install_create_user_acces_role_psk.sql
```

## Main Features (Application Pages)

| Page ID | Page Name |
|---|---|
| 0 | Global Page |
| 1 | Home |
| 2 | Quick Usage Tips |
| 3 | Form Add User Application |
| 4 | Load and Print Old Barcode |
| 6 | Individual Barcode Print History |
| 8 | Form Load Barcode |
| 9 | Form Load Barcode 2024 |
| 10 | Barcode Info |
| 12 | Master Serial Barcode |
| 13 | Form Master Serial Barcode |
| 14 | Load and Print Old Barcode - Detail |
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

The overall business flow is roughly:
1. **Load** raw-material weighing data (via Excel — `load_staging_barcode` / `load_staging_barcode_2024`) into the staging table.
2. **Generate barcodes** from staging data based on a combination of attributes (`MAT_FORM`, `COMPANY_CODE`, `ORIGIN`, `QUALITY`, `SUPPLIER`, `CROP_YEAR`, etc.) via the `ARRANGE_BARCODE` function.
3. **Print** selected barcode labels using the **APEX Office Print (AOP)** plugin.
4. **Scan/Read Barcode** again at the warehouse using the **BarcodeReader** plugin (device camera).
5. **Master Serial Barcode** and **Barcode Info** for managing supporting master data.
6. **Administration** — manage access rights, roles, and application users.

## Database Objects

**Tables:**
- `XTD_KDK_LP_STAGING_BARCODE` — staging data from Excel loads (mat form, origin, quality, supplier, crop year, gross/tare/net, total barcode, prefix, etc.).
- `XTD_KDK_LP_BARCODE` — individual generated barcodes, linked to staging.
- `XTD_KDK_LP_BARCODE_INFO` — master data for additional barcode information.
- `XTD_KDK_LP_JOB` — asynchronous job queue (status `PENDING`, `attempt_count`, `payload`, `max_attempt`).
- `XTD_KDK_LP_JOB_RUN` — job execution history (progress %, total/processed/failed items, error code/message).

**View:**
- `V_KDK_LP_BARCODE`

**Function:**
- `ARRANGE_BARCODE(p_id, p_is_2024)` — builds the barcode number format from staging data.
- `IS_TRANSIENT_ERROR` — detects transient errors (used for job retry logic).

**Procedure:**
- `BARCODE_JOB_WORKER` — worker that processes the barcode generation job queue.
- `PROCESS_KDK_LP_BARCODE_JOB` — core logic for processing a single job.
- `MARK_FAILED_JOB_TO_ATTEMPT` — flags failed jobs for retry.
- `RECOVER_STALE_STAGING` — recovers stuck/stale staging data.

> The job-queue pattern (`XTD_KDK_LP_JOB` + `JOB_RUN` + worker/retry procedures) indicates the barcode generation process runs asynchronously with retry and progress-tracking mechanisms.

## Security & Access Control

**Authentication scheme:** Oracle APEX Accounts.

**ACL Roles (Access Control):**
| Static ID | Role Name |
|---|---|
| `DIREKTUR` | Director |
| `ADMINISTRATOR` | Administrator |
| `ADMIN_WH` | Warehouse Admin |
| `WH` | Warehouse |
| `CONTRIBUTOR` | Contributor |
| `READER` | Reader |

**Authorization Schemes** (19 schemes, action/menu-based): Administration Rights, Contribution Rights, Reader Rights, Load_barcode, Load_barcode_2024, Load_barcode_info, Edit_barcode, Delete_barcode, Generate_barcode, Retry_generate_barcode, Print_barcode, Read_barcode, Barcode_info, Update_barcode_info, Delete_barcode_info, Master_serial_barcode, Create/Update/Delete_master_serial_barcode, Setup_dan_master_data.

## Plugins Used

| Plugin | Type | Function |
|---|---|---|
| `apexbcreader` (BarcodeReader) | Region Type | Read/scan barcodes via device camera |
| `be_apexrnd_aop` (UC - APEX Office Print) | Process Type | Generate printable documents/labels (PDF) |
| `be_apexrnd_aop_da` / `be_apexrnd_aop_convert_da` | Dynamic Action | Trigger AOP process from the client side |

## Data Load Definitions

- `load_staging_barcode` — Excel load format `format_load_barcode_label_print.xlsx`
- `load_staging_barcode_2024` — Excel load format `format_load_barcode_label_print_2024.xlsx`
- `load_barcode_info` — Excel load format `format_load_barcode_info.xlsx`

Other supporting files: application icons (32/144/192/256/512 px), logo (`label_print_logo`), loading animation, and a `test_templete.docx` template (likely the AOP print template).

## Installation

Run as the `KDK` schema (or a user with the `APEX_ADMINISTRATOR_ROLE` role) using SQLcl/SQL*Plus:

```sql
@install.sql
```

This script sequentially runs: set environment → drop existing application (if any) → create application → shared components (navigation, files, plugin settings, authorization, ACL, etc.) → pages → deployment (tables, views, functions, procedures, indexes).

> **Note:** This file is an *automatic export* from Oracle APEX. Per Oracle's standard practice, it should not be edited manually — changes should be made via the APEX Builder and then re-exported (or via the ADT/Liquibase tooling used in the PANDAWA workflow).

## Prerequisites

- Oracle Database (compatible with APEX 24.2, supporting `IDENTITY` columns and `TIMESTAMP(6)`).
- Oracle APEX 24.2 installed on the target workspace.
- The **APEX Office Print (AOP)** plugin registered on the instance for the label printing feature.
- Device camera access for the barcode scanning feature (`apexbcreader` plugin) — typically run from a mobile device/tablet in the warehouse.
