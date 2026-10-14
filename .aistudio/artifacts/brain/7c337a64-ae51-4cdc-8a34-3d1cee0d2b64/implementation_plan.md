# Feature: Download CSV for Microstock (Adobe Stock & Shutterstock)

Implementation plan to introduce a CSV export system for generated microstock metadata, featuring Adobe Stock and Shutterstock standard formatting, automated category generation, and direct saving to the downloads directory.

## User Review & Critical Decisions

> [!IMPORTANT]
> The user confirmed the following preferences:
> - **Export Scope**: Export all items currently in the list when downloading CSV.
> - **Save Destination**: Save directly and automatically to the same folder as metadata-injected files (`Download/WarMachineHybrid`).
> - **Categories Field**: Add a new editable "Categories" field located between "Description" and "Keywords" in the image detail card. This field will be generated alongside title, description, and keywords by Gemini AI and reflected in the CSV.
> - **CSV Button Visibility**: The Download CSV button appears in the preview header (to the left of the Vertical/Grid toggle) as soon as at least 1 image has completed metadata generation.

---

## 1. Overview & Core Concept

- **What It Does**: Microstock contributors can batch export generated asset metadata into standard, upload-ready CSV files tailored for Adobe Stock and Shutterstock.
- **Target Audience**: Microstock artists, vector illustrators, and stock photographers who batch upload assets to stock agencies.
- **Key Value**: Eliminates manual copy-pasting of metadata on microstock contributor portals by generating officially compliant CSV spreadsheets directly from War Machine.

---

## 2. User Experience & Visual Design

### Key User Flows
1. **Metadata Generation**: User loads images/vectors and runs Generate Metadata.
2. **Categories Display**: The detail card reveals the newly generated "Categories" field below "Description" and above "Keywords", editable with clear and copy shortcuts.
3. **Download CSV Trigger**: As soon as at least 1 item has metadata, a **Download CSV** button with download icon and "Csv" text appears to the left of the Grid/Vertical toggle.
4. **Platform Selection Modal**:
   - Tapping Download CSV opens a modal with Adobe Stock and Shutterstock selection cards styled like the Microstock cards in War Engine settings (`0xFF10B981` active green, rounded corners, responsive grid).
   - A prominent **"Save Csv"** button triggers CSV generation and downloads the file.
5. **Auto-Save & Notification**: The CSV is generated, written to `Download/WarMachineHybrid`, and a success toast confirms the file name and path.

### Visual Styling & Layout
- **CSV Button**: Compact button with `Icons.Default.Download` and bold "Csv" label in bright cyan (`0xFF38BDF8`) matching the preview controls.
- **Microstock Popup Dialog**: Dark futuristic surface (`0xFF0B132B` / `0xFF0F1E36`), 12dp rounded corners, animated card selection glow, and a high-contrast Emerald/Cyan action button for "Save Csv".

---

## 3. Key Product Decisions & Trade-Offs

- **Standardized CSV Formats**:
  - **Adobe Stock**:
    - Columns: `Filename,Title,Keywords,Category,Releases`
    - Adheres to Adobe Stock contributor bulk metadata upload guidelines.
  - **Shutterstock**:
    - Columns: `Filename,Description,Keywords,Categories`
    - Adheres to Shutterstock contributor bulk CSV upload specifications.
- **RFC 4180 Escaping**: Commas, double quotes, and line breaks in titles, descriptions, and keywords are properly wrapped and escaped (`""`) to prevent parsing corruption on microstock platforms.
- **Gemini Schema Extension**: The structured JSON prompt schema is updated to include `"categories"` (1–2 relevant microstock categories, e.g., "Vectors, Backgrounds", "Nature, Animals") during analysis.

---

## 4. Technical Architecture & Data Strategy

### System Flow Diagram
```
┌────────────────────────────────────────────────────────┐
│                      MainActivity                      │
│                                                        │
│  [Download CSV] ──► CsvExportDialog ──► Select Agency  │
│  (Left of Grid)                         (Adobe / SSTK) │
│                                                │       │
└────────────────────────────────────────────────┼───────┘
                                                 │
                                                 ▼
┌────────────────────────────────────────────────────────┐
│                     MainViewModel                      │
│                                                        │
│  • ImageItem: individualCategories                     │
│  • Gemini Schema: includes "categories"                │
│  • generateCsvBytes(agency: CsvAgency): ByteArray      │
│  • exportCsvToDownloads(agency: CsvAgency)             │
└────────────────────────────────────────────────┼───────┘
                                                 │
                                                 ▼
┌────────────────────────────────────────────────────────┐
│                      FileHelper                        │
│                                                        │
│  • saveBytesToDownloads(..., "Download/WarMachine...") │
│  • Writes: WarMachine_AdobeStock_Metadata.csv          │
│    or WarMachine_Shutterstock_Metadata.csv             │
└────────────────────────────────────────────────────────┘
```

### Data Model Additions
```kotlin
enum class CsvAgency(val displayName: String, val fileNamePrefix: String) {
    ADOBE_STOCK("Adobe Stock", "WarMachine_AdobeStock_Metadata"),
    SHUTTERSTOCK("Shutterstock", "WarMachine_Shutterstock_Metadata")
}

// In ImageItem:
val individualCategories: String = ""

// In GeneratedMetadata:
val categories: String? = null
```

### Component & State Mapping
- `MainActivity.kt`:
  - Preview header: Render `CsvDownloadButton` next to `toggle_grid_vertical_btn` when `imagesList.any { it.isGenerated }`.
  - `CsvExportModal`: Renders Adobe Stock and Shutterstock cards, selection state, and "Save Csv" action.
  - `FullImageCard`: Insert `Categories` `OutlinedTextField` between Description and Keywords.
- `MainViewModel.kt`:
  - `updateIndividualCategories(id: Int, categories: String)`
  - `exportCsv(agency: CsvAgency)`: Builds RFC 4180 CSV string, converts to UTF-8 bytes, and invokes `FileHelper.saveBytesToDownloads`.
