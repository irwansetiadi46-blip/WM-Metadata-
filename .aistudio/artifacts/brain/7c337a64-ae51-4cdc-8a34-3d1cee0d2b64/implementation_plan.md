# Implementation Plan: Fitur Auto Rename Switch

Menambahkan pengaturan switch **Auto Rename** tepat di bawah baris **Auto Inject Metadata** di halaman Settings / War Engine. Fitur ini memungkinkan pengguna mengaktifkan atau menonaktifkan rename otomatis terhadap file dan hasil ekspor CSV, dengan status default **ON**.

---

## 1. User Review Required

> [!IMPORTANT]
> **Status Default & Posisi:**
> - Switch **Auto Rename** diposisikan persis di bawah baris **Auto Inject Metadata** dengan tampilan kartu baris yang identik (teks label `Auto Rename :` dan kontrol `Switch`).
> - Sesuai instruksi, **tidak ada caption/teks deskripsi tambahan** agar tampilan ringkas dan bersih.
> - Default status adalah **ON** dan tersimpan di `SharedPreferences` (`is_auto_rename`).

---

## 2. Proposed Changes

### ViewModel & Model (`app/src/main/java/com/example/MainViewModel.kt`)

1. **State & Persistence:**
   - Tambahkan `_isAutoRenameEnabled = MutableStateFlow(true)` dan publikasikan sebagai `isAutoRenameEnabled: StateFlow<Boolean>`.
   - Di `init`, baca nilai tersimpan dari `SharedPreferences("WarMachinePrefs")` dengan kunci `"is_auto_rename"`, default `true`.
   - Tambahkan fungsi `setAutoRenameEnabled(enabled: Boolean)` untuk mengupdate state dan menyimpan preferensi ke `SharedPreferences`.

2. **Preservasi Nama File Asli (`ImageItem`):**
   - Tambahkan properti `originalName: String = name` pada `ImageItem` saat berkas pertama kali diimpor, sehingga nama asli tidak hilang meskipun pernah direname.

3. **Logika Rename pada Injeksi & Download:**
   - Di `injectIndividualMetadata` & `injectMetadata`:
     - Jika `isAutoRenameEnabled == true`: gunakan nama baru berbasis AI (`newFileName`).
     - Jika `isAutoRenameEnabled == false`: pertahankan nama asli (`originalName` / `item.name`), tanpa mengubah nama berkas.
   - Di `getEffectiveBaseName(item)`:
     - Jika `isAutoRenameEnabled == false`, kembalikan nama dasar asli (`originalName` tanpa ekstensi).

4. **Logika Penamaan CSV (`resolveCsvFileName`):**
   - Jika `isAutoRenameEnabled == true`: gunakan nama file hasil generate AI (`item.individualFileName`).
   - Jika `isAutoRenameEnabled == false`: gunakan nama file asli (`item.originalName` / `item.name`) dengan format ekstensi target (SVG/EPS jika format vektor dipilih, atau ekstensi aslinya).

---

### UI (`app/src/main/java/com/example/MainActivity.kt`)

1. **Koleksi State:**
   - Tambahkan `val isAutoRenameEnabled by viewModel.isAutoRenameEnabled.collectAsStateWithLifecycle()` di dalam composable Settings / War Engine.

2. **Komponen Switch Row:**
   - Di bawah baris `Auto Injection Row`, tambahkan spacer pemisah (`10.dp`).
   - Tambahkan baris `Auto Rename Row` dengan gaya kartu yang konsisten:
     - Background gelap transparan dengan sudut melengkung `RoundedCornerShape(10.dp)` dan border `1.dp Color(0x20FFFFFF)`.
     - Label: `Text("Auto Rename :", fontSize = 13.sp, fontWeight = FontWeight.Bold, color = Color.White)`.
     - Kontrol: `Switch(checked = isAutoRenameEnabled, onCheckedChange = { viewModel.setAutoRenameEnabled(it) })`.
     - `Modifier.testTag("auto_rename_switch")`.
     - **Tanpa caption sampah/teks keterangan tambahan**.

---

## 3. Verification Plan

### Automated Build & Compilation
- Jalankan `compile_applet` untuk memverifikasi bahwa perubahan kode di `MainActivity.kt` dan `MainViewModel.kt` berhasil dikompilasi tanpa error.

### Manual / Scenario Verifications
1. **Verifikasi Tampilan UI:**
   - Buka halaman Settings / War Engine.
   - Pastikan switch `Auto Rename :` muncul tepat di bawah `Auto Inject Metadata :` dengan style yang rapi dan konsisten, tanpa caption tambahan.
2. **Verifikasi State Default & Persistensi:**
   - Buka aplikasi pertama kali -> switch berada pada posisi ON.
   - Ubah switch ke OFF -> tutup dan buka kembali -> switch tetap OFF.
3. **Verifikasi Saat Auto Rename ON:**
   - Generate metadata & inject -> nama file berubah sesuai filename hasil AI.
   - Ekspor CSV -> nama file di kolom CSV menggunakan filename hasil generate AI.
4. **Verifikasi Saat Auto Rename OFF:**
   - Generate metadata & inject -> nama file tetap menggunakan nama asli.
   - Download file -> berkas tersimpan dengan nama asli.
   - Ekspor CSV -> kolom filename di file CSV menggunakan nama file asli.
