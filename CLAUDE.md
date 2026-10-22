# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

UTTMXBridge is a Microsoft Dynamics 365 Business Central (AL) extension, published by "Barki". It is a Mexico-specific "bridge" app that sits on top of several other installed extensions and patches/extends their behavior for the UTT Mexico company:

- **KUMAVISION textile / factory / base / KVS Anpassungen** — the customer's industry-vertical BC extensions (textile manufacturing).
- **KVS Customization Bridge MX** — Kumavision's own Mexico localization bridge (this app overrides/hides some of its page actions).
- **Kosmos Tek CFDI Reports / Electronic Invoicing / Electronic Accounting / International Commerce / Base Library** — the Mexican e-invoicing (CFDI/PAC) stack. Several features in this app clean up or extend CFDI-related fields such as `Fiscal Invoice Number PAC` (the PAC/UUID stamped on invoices).
- **UTTAPP** (also by Barki) — the company's own cross-market base customization app.
- **Quality Management Extension**, **EDX**.

Object IDs are allocated from the declared range **50000–99999** (`app.json`). The `target` is `OnPrem`, `application`/`platform` is BC 18.5, and the `TranslationFile` feature is enabled (translations live in `Translations/*.xlf`, base file `UTTDE2MEXReport.g.xlf`).

## Commands

This is a VS Code **AL Language** extension project, not an npm/dotnet CLI project — there is no package.json/build script. Use the AL extension commands inside VS Code:

- **Download symbols**: `AL: Download Symbols` (needed after any dependency/app.json change; packages are cached in `.alpackages/`).
- **Build/package**: `AL: Package` (Ctrl+Shift+B) — produces a `.app` file at the repo root (e.g. `Barki_UTTMXBridge_<version>.app`).
- **Publish & run**: `F5`, which uses one of the named configurations in `.vscode/launch.json` (`TestNAVAuth`, `TestMX`, `Test`, `ProdDebug`, `BCMXProd_Debug`, `Test_Jobqueue`, `Prod_Jobqueue`, `Webservice`, `BCProd Webservice`, `WebserviceMXTest`). **`TestMX` is the Mexico-company test environment** and is the one most relevant to this app's purpose; several configs are `attach`-type (for debugging a running job queue/web service session, not for publishing).
- **Object naming convention**: enforced via `.vscode/settings.json` CRS settings — file names follow `<ObjectNameShort>.<ObjectTypeShortPascalCase>.al` and all new objects get the `UTT ` caption/name prefix.
- **Code analyzers**: `CodeCop` and `UICop` are enabled (`al.codeAnalyzers` in `.vscode/settings.json`); `PerTenantExtensionCop` is present but commented out.
- **Tests**: there is an `.altestrunner/config.json` for the AL Test Runner extension, but it is unconfigured (no test suite/container set) and **no test codeunits exist in `src/`**. There is currently no automated test suite for this app.

Note: two source files carry a `.txt` suffix (`src/codeunit/UTTEventSubscriber.Codeunit.al.txt`, `src/report/UTTVendorItemPurchases.Report.al.txt`). The AL compiler only picks up `.al` files, so **these are intentionally excluded from the build** — treat them as disabled/reference code, not dead files to delete without checking why they were disabled.

## Architecture

### `src/codeunit/` — install/upgrade plumbing and event hooks
- **`UTTMXBridgeUpgrade` (67017)** — `Subtype = Install` codeunit. On `OnInstallAppPerCompany`, branches into `HandleFreshInstall` (currently empty) or `HandleReinstall`. `HandleReinstall` calls into `UTTMXBridgeInstallUpgradeMgt` to (re)create a permission set and wire it into Sales & Receivables Setup.
- **`UTTMXBridgeInstallUpgradeMgt` (67015)** — `CreatePermission` creates/replaces a `tenant Permission Set` row scoped to this app's module ID for a given role ID; `AddSalesStpPermissionID` stores that role ID into the custom `Sales & Receivables Setup.UTTDeleteFolio` field. Together these set up the `UTTDELETEUUID` permission set used to gate the fiscal-folio-deletion feature below.
- **`UTT EventSubscriber` (67023, disabled — `.al.txt`)** — subscribes to `Release Purchase Document::OnCodeOnAfterCheck`. Intent: for Mexico-company purchases where the vendor's country is *not* MX, validate that received G/L/Item purchase lines have Pedimento (customs import declaration) fields filled in — but the actual `TestField` calls are commented out, so the check is a no-op even when re-enabled.

### `src/pageextension/` — UI integration with the base apps
- **`UTTLIB` (codeunit 67016, despite living under pageextension/)** — the core business logic of this app:
  - `DeleteFiscalFolio(var purchInvHeader)` — clears the Mexican CFDI `Fiscal Invoice Number PAC` (UUID) from a posted purchase invoice and cascades the clear across `Vendor Ledger Entry`, `Detailed Vendor Ledg. Entry`, `VAT Entry`, `G/L Entry` (matched by `Document No.`), and also finds the related `Purch. Cr. Memo Hdr.` by the old UUID and repeats the cascade for it. Gated by `Confirm`, `SalesSetup.UTTDeleteFolio` being set, and `IsUserHasPermission`.
  - `IsUserHasPermission(RoleID)` — checks `Access Control` for the current user having that role (company-specific or blank/global row); errors otherwise.
  - Declares `permissions = tabledata ... RM` for the tables it writes to, since it runs with elevated data permissions independent of the calling user's normal table permissions.
- **`UTTPostedPurchaseInvoiceExt` (67070)** / **`UTTPostedPurchaseInvoicesExt` (67072)** — hide the `DeleteFiscalFolio` action that ships from **KVS Customization Bridge MX** on the Posted Purchase Invoice (single/list) pages and replace it with this app's own `UTTDeleteFiscalFolio` action, which calls `UTTLib.DeleteFiscalFolio`. This is a deliberate override of another extension's UI, not a duplicate by accident.
- **`UTT PAymentJournalExt` (65054)** — adds a `Sign Payment` field to the Payment Journal page, placed before "Posting Date".

### `src/reportextension/`
- **`UTT GLEntryExtended` (67000)** extends the standard **G/L Register** report, adding `Fiscal Invoice Number PAC` and `Related UUID` columns from `G/L Entry` — i.e. exposing the CFDI UUID on the GL register for audit/reconciliation.

### `src/report/` — custom RDLC reports
Most of these are **re-implementations of standard Microsoft/Kumavision financial reports** under new UTT-prefixed object IDs (so they can be customized/retained independently of the base objects they shadow): `UTT Customer Detailed Aging` (50090), `UTT Customer Total-Balance` (67020), `UTT G/L Total-Balance` (50086), `UTT Vendor Detailed Aging` (50091), `UTT Vendor Total-Balance` (55085), `UTT Vendor/Item Purchases` (50093, disabled — `.al.txt`), `UTT Fixed Asset-Book Value 03` (50088), `UTT Fixed Assets - List AT` (50087). Their RDLC layouts live in `Layout/`.
- **`UTT Piece Pedimento` (67019, "Submaquila Report")** is the one genuinely custom report. It walks the textile production genealogy backwards from an `Item Ledger Entry` output record to find which import customs declaration ("Pedimento") the consumed yarn came from, for Mexican *submaquila* (maquiladora/export-manufacturing customs regime) reporting:
  `Output entry → Assigned Piece (Output) → consumed/Used Yarn → original input-yarn Purchase entry → Pedimento No.` (read from a custom field on the item ledger entry, falling back to the `Value Entry` → `Purch. Inv. Line` chain if blank).
- `src/report/report.7z` is an archived/backed-up copy of older report sources (not part of the compiled app).

### `src/DotNet.al`
Declares .NET interop types available to AL code: `System.Data.SqlClient` (`SqlCommand`/`SqlConnection`), `System.IO.MemoryStream`, `System.Drawing` (`Bitmap`, `ImageFormat`, `RotateFlipType`), and the `ZXing` barcode library (`BarcodeWriter`, `BarcodeFormat`, etc.). These are OnPrem-only interop declarations (direct SQL access and barcode/image generation); grep for usages before assuming any one of them is live, since not all are wired into the current report/codeunit set that's included in the build.

### Key cross-cutting concept: the Fiscal Folio / CFDI UUID
Several unrelated-looking pieces (`UTTLib.DeleteFiscalFolio`, the two Posted Purchase Invoice page extensions, the install/upgrade codeunits, the G/L Register report extension) all revolve around one feature: **letting an authorized user strip a Mexican CFDI UUID (`Fiscal Invoice Number PAC`) off a posted purchase invoice/credit memo and all of its downstream ledger entries**, typically to correct a bad e-invoice stamp. The permission role ID for who's allowed to do this is configurable per-company via `Sales & Receivables Setup.UTTDeleteFolio`, set up by the install/upgrade codeunits during reinstall.
