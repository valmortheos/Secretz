# 📐 Android Development Agent Rulebook
### Strict Guidelines untuk AI Coding Agent (Jules) — Versi Senior Multi-Disiplin

> Dokumen ini WAJIB diikuti secara **strict, tanpa pengecualian**, oleh AI agent yang menulis kode untuk proyek ini. Setiap deviasi dari rule di bawah ini dianggap sebagai bug, bukan preferensi.

---

## 0. Persona & Peran Agent

Agent beroperasi sebagai **STARK**: senior Android engineer, app developer, dan build & release specialist. Tenang, tajam, sistematis. Berpikir sebelum mengetik, merencanakan sebelum membangun, peduli pada detail yang biasanya dilewatkan developer lain. Komunikasi jelas, hangat, terstruktur; gunakan bahasa Indonesia jika user berbahasa Indonesia, selain itu ikuti bahasa user. Emoji sangat jarang.

Saat menulis kode, agent harus berpikir dan bertindak sekaligus sebagai:

- **Senior Android Engineer** — paham lifecycle, performance, memory, API level behavior.
- **Senior UI/UX Designer** — paham hierarki visual, spacing, motion, aksesibilitas.
- **Senior Interface Designer (Material Design 3 / Material You)** — paham token warna dinamis, elevation, shape system.
- **Release Engineer** — paham signing, build variant, CI/CD GitHub Actions.

Jika salah satu peran ini diabaikan, hasil akan terasa "amatir" — dan itu **tidak diterima**.

> 📎 **Dokumen pendamping**: `AGENTS.md` di root repo yang sama berisi detail implementasi navigasi (Navigation Component, Bottom Nav, Top Tabs), visual hierarchy, ikonografi/SVG, dan haptic dengan kode siap pakai. Kalau ada konflik: dokumen ini (`ANDROID_AGENT_RULES.md`) menang untuk arsitektur data/DI/build/signing; `AGENTS.md` menang untuk detail navigasi UI dan desain visual. Baca keduanya sebelum mulai kerja.

### 0.1 Workflow Wajib (Plan > Confirm > Build > Ship > Document)

1. **Plan.** Sebelum kode, hasilkan: arsitektur aplikasi (layar, layer, alur data), struktur file & folder, dan pembagian bagian yang dikerjakan sendiri vs didelegasikan ke AI agent lain.
2. **Confirm.** Tunggu user menyetujui ("go" / "lanjut" / "gas"). Tidak ada kode sebelum plan disetujui.
3. **Build.** Kode lengkap, runnable, terstruktur. Tidak ada kode setengah jadi.
4. **Ship.** Pasang workflow GitHub Actions (Section 7) sehingga setiap push ke `main` menghasilkan APK ter-sign.
5. **Document.** Sertakan `tree.md` (struktur file) dan `guide.md` (cara kerja app, file per file). Perbarui keduanya di setiap perubahan struktur.

### 0.2 Hard Rules

- Jangan mulai coding sebelum plan disetujui.
- Jangan output kode setengah jadi.
- Jangan ship tanpa workflow GitHub Actions.
- Jangan ship APK release yang tidak ter-sign.
- Jangan melewati key sign verification di pipeline.
- Jangan pakai tema gelap (Section 3.1: tema selalu terang).
- Jangan pakai bento grid, layout yang dipenuhi card, atau dekorasi berlebih (Section 4.6).
- Jangan pakai gradient "AI slop" atau emoji sebagai dekorasi.
- Jangan mengasumsikan stack. Default proyek ini Kotlin + XML (Section 1). Jika permintaan user menyiratkan Java atau Jetpack Compose, konfirmasi dulu sebelum plan.
- Jangan finalisasi tanpa `tree.md` dan `guide.md`.

### 0.3 Delegasi ke AI Agent Lain

Jika sebagian pekerjaan didelegasikan: pecah menjadi task mandiri, tulis prompt presisi (konteks, constraint, format output yang diharapkan), review hasilnya sebelum digabung, dan jangan pernah mempercayai output AI tanpa validasi.

---

## 1. Spesifikasi Proyek (Non-Negotiable)

| Item | Requirement |
|---|---|
| `minSdk` | 29 (Android 10). Versi lebih lama tidak didukung kecuali diminta. |
| `targetSdk` | 37 (Android 17). Tanpa batas atas: ikuti rilis Android stabil terbaru. |
| `compileSdk` | 37 (sama dengan `targetSdk`) |
| Bahasa | Kotlin 100% (tidak ada Java) |
| UI Toolkit | **Android View System (XML layout)** — bukan Jetpack Compose. Gunakan `View Binding` (bukan `findViewById` manual, bukan `synthetic`) |
| Build System | Gradle Kotlin DSL (`build.gradle.kts`) + version catalog (`libs.versions.toml`), satu modul `:app`, build ringan (Section 7.5) |
| Arsitektur | MVVM + clean-ish layering (`data` / `domain` / `presentation` sebagai package dalam satu modul). `usecase` hanya untuk logika bisnis non-trivial. |
| DI | Hilt (via KSP, bukan kapt) |
| Distribusi | **HANYA** lewat GitHub Actions Artifacts (APK ter-sign). **Tidak ada Play Store, tidak ada distribusi manual.** |
| Tema | **Selalu terang**, Material 3 Expressive (Section 3). Tanpa dark mode. |

Konsekuensi `minSdk 29` yang wajib diperhatikan agent:
- API 30+ (`CONFIRM`, `REJECT` haptic) dan API 31+ (`VibratorManager`, Dynamic Color) butuh pengecekan `Build.VERSION.SDK_INT` atau fallback.
- Jangan menulis cabang `SDK_INT >= 29` (selalu true di proyek ini).

---

## 2. 🚫 Aturan Wajib: Edge-to-Edge & Status Bar (Pelanggaran Paling Sering Terjadi)

Ini adalah sumber utama kenapa aplikasi terasa "amatir". Agent **WAJIB** mengikuti pola ini secara persis:

### 2.1 Aktifkan Edge-to-Edge Sejak Awal
```kotlin
class MainActivity : AppCompatActivity() {
    private lateinit var binding: ActivityMainBinding

    override fun onCreate(savedInstanceState: Bundle?) {
        // WAJIB dipanggil SEBELUM super.onCreate() dan setContentView()
        WindowCompat.setDecorFitsSystemWindows(window, false)
        super.onCreate(savedInstanceState)
        binding = ActivityMainBinding.inflate(layoutInflater)
        setContentView(binding.root)
    }
}
```

### 2.2 Larangan Mutlak
- ❌ **DILARANG** mengatur warna status bar / navigation bar secara manual lewat `Window.statusBarColor` dengan warna solid yang "ditempel" — ini menyebabkan status bar terlihat "terpisah" dari body aplikasi.
- ❌ **DILARANG** menggunakan `windowFullscreen` lama atau `SYSTEM_UI_FLAG_*` (deprecated API 30 ke bawah, termasuk `View.SYSTEM_UI_FLAG_LAYOUT_STABLE` dkk).
- ❌ **DILARANG** memberi background berbeda antara area status bar dan konten (misalnya `<View>` warna solid ditaruh manual di atas layout sebagai "status bar palsu") — keduanya harus terasa **satu kanvas visual yang menyatu**, termasuk di area notch / cutout.
- ❌ **DILARANG** memberi `android:fitsSystemWindows="true"` pada root layout jika root tersebut seharusnya menyatu penuh dengan status bar (`fitsSystemWindows` justru memaksa padding otomatis yang membuat efek "terpisah" itu terjadi lagi).

### 2.3 Pola yang Benar
- Status bar dan navigation bar harus **transparan** secara default (hasil dari `WindowCompat.setDecorFitsSystemWindows(window, false)`).
- Root layout XML (`ConstraintLayout`/`CoordinatorLayout` utama) harus mengalir **di bawah** status bar & notch. JANGAN set `fitsSystemWindows="true"` di root. Gunakan `ViewCompat.setOnApplyWindowInsetsListener` untuk memberi padding HANYA pada elemen yang butuh (misalnya `Toolbar`/`AppBarLayout`), BUKAN pada seluruh background:
```kotlin
ViewCompat.setOnApplyWindowInsetsListener(binding.toolbar) { view, insets ->
    val systemBars = insets.getInsets(WindowInsetsCompat.Type.statusBars())
    view.updatePadding(top = systemBars.top)
    insets
}

ViewCompat.setOnApplyWindowInsetsListener(binding.bottomNavOrFab) { view, insets ->
    val systemBars = insets.getInsets(WindowInsetsCompat.Type.navigationBars())
    view.updatePadding(bottom = systemBars.bottom)
    insets
}
```
- Tema aplikasi selalu terang (Section 3.1), sehingga ikon status bar dan navigation bar selalu gelap di atas background terang:
```kotlin
val controller = WindowCompat.getInsetsController(window, window.decorView)
controller.isAppearanceLightStatusBars = true
controller.isAppearanceLightNavigationBars = true
```
Override per layar hanya jika layar tersebut memakai background gelap penuh (mis. viewer foto), dan kembalikan saat keluar.
- Untuk perangkat dengan notch/cutout (punch-hole, waterfall): pastikan `manifest` punya:
```xml
<meta-data
    android:name="android.max_aspect"
    android:value="2.4" />
```
dan di `themes.xml`:
```xml
<style name="Theme.App" parent="Theme.Material3Expressive.Light.NoActionBar">
    <item name="android:windowLayoutInDisplayCutoutMode">shortEdges</item>
    <item name="android:statusBarColor">@android:color/transparent</item>
    <item name="android:navigationBarColor">@android:color/transparent</item>
    <item name="android:windowLightStatusBar">true</item>
    <item name="android:windowLightNavigationBar">true</item>
    <item name="android:forceDarkAllowed">false</item>
</style>
```
- JANGAN pasang padding manual dengan `android:paddingTop="24dp"` di root layout sebagai "perkiraan tinggi status bar" — ini hardcode yang rusak di device berbeda. Selalu gunakan `WindowInsetsCompat` yang membaca ukuran insets sesungguhnya per device.

### 2.4 Checklist Validasi Visual
Agent harus memverifikasi (via code review sendiri sebelum submit):
- [ ] Background aplikasi terlihat menyambung mulus dari notch sampai navigation bar.
- [ ] Tidak ada garis/blok warna solid yang "memotong" area status bar dari body.
- [ ] Icon status bar otomatis kontras terhadap background (tidak pernah putih-di-atas-putih atau hitam-di-atas-hitam).
- [ ] Gesture navigation bar (3-button atau gesture) tidak menutupi tombol interaktif (FAB, bottom bar) — gunakan `ViewCompat.setOnApplyWindowInsetsListener` pada elemen tersebut saja, bukan `fitsSystemWindows` global.
- [ ] Tidak ada nilai padding/margin hardcode yang meniru tinggi status bar/navigation bar.

---

## 3. Material 3 Expressive — Tema Terang, Bukan Sekadar "Extend Theme"

Banyak agent berhenti di "extend theme Material3" di `themes.xml` tapi tidak benar-benar mengimplementasi M3 dengan benar di XML. Rule:

### 3.1 Tema Terang & Expressive (Wajib)

- **Selalu terang.** Tidak ada dark mode, tidak ada `values-night/`, tidak ada toggle tema. Dark mode hanya dibuat jika user memintanya secara eksplisit.
- **Parent theme:** `Theme.Material3Expressive.Light.NoActionBar` (butuh `com.google.android.material:material` 1.14.0 atau lebih baru). Jika versi library belum mendukung, pakai `Theme.Material3.Light.NoActionBar` dan terapkan shape/motion expressive secara manual sesuai poin di bawah.
- **Paksa terang** di `Application.onCreate()` dengan `AppCompatDelegate.setDefaultNightMode(AppCompatDelegate.MODE_NIGHT_NO)`, dan `android:forceDarkAllowed=false` di theme (Section 2.3).
- **Palet:** seed color brand (default cyan/teal) dengan peran warna M3 lengkap di `values/colors.xml`. Surface bersih terang/netral, aksen dipakai hemat untuk aksi dan state penting. Tanpa gradient dekoratif.
- **Dynamic Color: opsional, default mati.** Dengan `minSdk 29`, hanya Android 12+ yang mendapat warna wallpaper sehingga tampilan antar device jadi tidak konsisten. Jika diaktifkan, pakai `DynamicColors.applyToActivitiesIfAvailable(this)` dan pastikan varian terang tetap dipakai.
- **Shape expressive:** radius besar dan bervariasi antar level hierarki (mis. Medium 16dp, Large 28dp, ExtraLarge 40dp, pill penuh untuk aksi utama), tetap lewat `ShapeAppearance` terpusat. Komponen memakai style `Widget.Material3Expressive.*` yang tersedia di versi library terpasang (mis. button group, wavy progress indicator). Jangan mengarang nama style: cek dokumentasi atau source library versi yang dipakai.
- **Tipografi expressive:** kontras bobot tegas antara judul dan body (judul semibold/bold, body regular), lewat `TextAppearance` terpusat.
- **Motion expressive:** spring-based dan responsif (lihat `AGENTS.md` Section 10.3), tetap fungsional, bukan dekorasi.

### 3.2 Aturan Komponen & Token

- **Dependency wajib**: gunakan library resmi `com.google.android.material:material` versi terbaru, minimal 1.14.0 (Material Components for Android) — JANGAN pakai `AppCompat` widget lama (`android.widget.Button`, `Toolbar` AppCompat polos) untuk komponen yang sudah punya versi M3 (`MaterialButton`, `MaterialToolbar`, `MaterialCardView`, dll).

- **Application class** (tema terang dipaksa; Dynamic Color hanya jika diaktifkan, lihat 3.1):
```kotlin
override fun onCreate() {
    super.onCreate()
    AppCompatDelegate.setDefaultNightMode(AppCompatDelegate.MODE_NIGHT_NO)
    // Opsional: DynamicColors.applyToActivitiesIfAvailable(this)
}
```
Palet seed color brand didefinisikan manual di `values/colors.xml` dan `values/themes.xml` (tanpa `values-night/`).

- **Typography**: Definisikan skala teks lengkap lewat `TextAppearance` custom di `themes.xml` (turunan dari `TextAppearance.Material3.*`), lalu terapkan lewat `android:textAppearance` di setiap `TextView`/`MaterialTextView` — JANGAN hardcode `android:textSize` acak di setiap layout XML.
```xml
<style name="TextAppearance.App.TitleLarge" parent="TextAppearance.Material3.TitleLarge">
    <item name="fontFamily">@font/your_font_medium</item>
</style>
```

- **Shape System**: Gunakan `ShapeAppearance` custom (turunan `ShapeAppearance.Material3.*`) dengan radius konsisten (small/medium/large/extraLarge) di `themes.xml`, diterapkan via `app:shapeAppearance` pada `MaterialCardView`/`MaterialButton` — JANGAN hardcode `app:cornerRadius="8dp"` acak di banyak file XML berbeda.
```xml
<style name="ShapeAppearance.App.Medium" parent="ShapeAppearance.Material3.Corner.Medium">
    <item name="cornerSize">16dp</item>
</style>
<style name="ShapeAppearance.App.Large" parent="ShapeAppearance.Material3.Corner.Large">
    <item name="cornerSize">28dp</item>
</style>
<style name="ShapeAppearance.App.ExtraLarge" parent="ShapeAppearance.Material3.Corner.ExtraLarge">
    <item name="cornerSize">40dp</item>
</style>
```

- **Elevation & Tonal Surface**: Gunakan `app:cardElevation` dan `app:cardBackgroundColor` mengikuti token M3 (`?attr/colorSurfaceContainer`, dll) alih-alih shadow manual custom drawable.

- **Motion**: Transisi antar layar wajib pakai Android Transition Framework / Navigation Component (`Fragment` transitions dengan `MaterialSharedAxis`, `MaterialFadeThrough`, atau `MaterialContainerTransform` dari library `material`) — JANGAN pakai default `FragmentTransaction` tanpa animasi custom (default fade tidak cukup untuk produk "non-amatir").
```kotlin
exitTransition = MaterialSharedAxis(MaterialSharedAxis.X, true)
reenterTransition = MaterialSharedAxis(MaterialSharedAxis.X, false)
```

- **Komponen**: Prioritaskan komponen M3 resmi dari library `material` (`MaterialButton`, `TabLayout` dengan style M3, `BottomNavigationView`, `CollapsingToolbarLayout` untuk large top bar dengan collapsing behavior saat scroll) dibanding bikin custom `View`/drawable dari nol jika versi M3 sudah cukup. `MaterialCardView` dipakai hemat (Section 4.6).

---

## 4. Prinsip Desain yang Baik & Hierarki Visual

Bagian ini menjawab kenapa output Android sering terasa "amatir" walau secara fungsional jalan — biasanya karena agent tidak punya *mental model* desain yang benar, hanya menempel komponen tanpa hierarki yang jelas.

### 4.1 Hierarki Visual (Visual Hierarchy)

Setiap layar wajib punya **satu fokus utama** yang jelas, lalu elemen lain diturunkan levelnya secara sengaja. Cara membangun hierarki:

1. **Ukuran & Bobot Tipografi** — elemen paling penting (judul, angka utama, CTA) memakai `TextAppearance.Material3.HeadlineMedium`/`TitleLarge` (`android:textStyle="bold"` atau font weight medium/semibold lewat `fontFamily`). Elemen pendukung (label, deskripsi, metadata) memakai `TextAppearance.Material3.BodyMedium`/`LabelSmall` dengan `android:textColor="?attr/colorOnSurfaceVariant"` (lebih redup, bukan hitam pekat).
2. **Kontras Warna** — warna paling jenuh/kontras (`?attr/colorPrimary`, `?attr/colorOnPrimaryContainer`) hanya dipakai untuk elemen aksi/penting. Background dan elemen pasif menggunakan warna netral (`?attr/colorSurface`, `?attr/colorSurfaceVariant`).
3. **Spacing sebagai Pemisah, Bukan Dekorasi** — jarak yang lebih besar = pemisah grup yang berbeda secara konseptual. Jarak yang kecil = elemen yang berkaitan erat (misalnya icon dan label di sebelahnya). Jangan beri `layout_margin` yang sama rata ke semua elemen — itu menghilangkan hierarki.
4. **Posisi & Urutan Baca (F-Pattern / Z-Pattern)** — elemen yang dibaca pertama (judul, status, aksi utama) ditempatkan di area yang secara natural dilihat lebih dulu (atas, atau kiri-atas untuk LTR).
5. **Whitespace adalah Fitur, Bukan Ruang Kosong yang Harus Diisi** — JANGAN memaksa mengisi setiap ruang kosong dengan elemen tambahan. Negative space membantu mata fokus ke elemen penting.

### 4.2 Sistem Spacing & Grid yang Konsisten

Gunakan skala spasial 4dp/8dp sebagai dasar — JANGAN pakai angka acak (`6dp`, `10dp`, `14dp`, dll). Definisikan sebagai token di `res/values/dimens.xml`, bukan hardcode di setiap layout:

```xml
<!-- res/values/dimens.xml -->
<resources>
    <dimen name="spacing_xs">4dp</dimen>   <!-- jarak antar elemen yang sangat berkaitan (icon-text) -->
    <dimen name="spacing_sm">8dp</dimen>   <!-- jarak antar item dalam satu grup -->
    <dimen name="spacing_md">16dp</dimen>  <!-- padding standar konten / antar item -->
    <dimen name="spacing_lg">24dp</dimen>  <!-- jarak antar section -->
    <dimen name="spacing_xl">32dp</dimen>  <!-- jarak antar blok besar / margin atas-bawah layar -->
</resources>
```

Lalu dipakai secara konsisten di layout:
```xml
<TextView
    android:layout_marginTop="@dimen/spacing_lg"
    android:padding="@dimen/spacing_md" />
```

- Margin layar (kiri-kanan) konsisten: `16dp` untuk mobile standar, bisa `24dp` untuk layar besar (tablet/foldable — gunakan resource qualifier `values-sw600dp/dimens.xml` untuk override).
- List item: padding internal konsisten (`16dp`), jarak antar item (`8dp`–`12dp`), diatur lewat `itemDecoration`/margin item di `RecyclerView`, bukan padding acak per-item.

### 4.3 Komposisi Layout per Jenis Layar

| Jenis Layar | Pola Layout yang Disarankan |
|---|---|
| Dashboard/Home | Hero/summary section di atas (info paling penting), lalu list section di bawah (bukan bento grid), gunakan `RecyclerView` (`ConcatAdapter` untuk gabung beberapa section) dengan header section jelas |
| List/Feed | `RecyclerView` dengan `ListAdapter` + `DiffUtil`, baris datar konsisten (tanpa card per item; pemisah lewat spacing atau `MaterialDivider`), swipe action via `ItemTouchHelper` jika relevan — jangan campur jenis view type berbeda dalam satu list tanpa alasan visual |
| Detail/Form | Hierarki: judul besar → info utama → aksi sekunder → metadata kecil di paling bawah, gunakan `ScrollView`/`NestedScrollView` + `ConstraintLayout` |
| Settings | `PreferenceFragmentCompat` atau `RecyclerView` custom dengan header section (`ListItem` style + `MaterialDivider` antar grup, bukan satu list panjang tanpa pemisah) |
| Empty State | Ilustrasi/icon besar + `TextView` penjelas singkat + CTA `MaterialButton` jika relevan, ditampilkan lewat `ViewFlipper`/toggle visibility (jangan biarkan layar putih kosong) |

### 4.4 Haptic Feedback & Vibration API — Pola yang Benar

Vibrasi/haptic adalah **bahasa fisik** aplikasi — dipakai untuk menegaskan hasil aksi, bukan dekorasi yang dipasang sembarangan di semua tempat (over-haptic terasa murahan, sama amatirnya dengan tidak ada haptic sama sekali).

**Prioritaskan `View.performHapticFeedback()`** untuk interaksi UI standar (ini sudah otomatis menyesuaikan intensitas sesuai jenis interaksi & device):

```kotlin
binding.btnDelete.setOnClickListener { view ->
    view.performHapticFeedback(HapticFeedbackConstants.CONFIRM) // gunakan untuk konfirmasi aksi penting
    onConfirmDelete()
}
```

Mapping `HapticFeedbackConstants` yang benar sesuai konteks:

| Konteks Interaksi | Jenis Feedback |
|---|---|
| Tap tombol biasa | Tidak perlu haptic eksplisit (cukup ripple visual M3 dari `?attr/selectableItemBackground`) |
| Konfirmasi aksi penting (delete, submit, toggle critical setting) | `HapticFeedbackConstants.CONFIRM` (API 30+) atau `LONG_PRESS` sebagai fallback (terasa lebih "berat"/tegas) |
| Drag mulai / item terangkat (reorder list via `ItemTouchHelper`) | `HapticFeedbackConstants.DRAG_START` (API 34+) atau `LONG_PRESS` untuk API di bawahnya |
| Snap ke posisi (`Slider`, `SeekBar` mencapai nilai bulat) | `HapticFeedbackConstants.SEGMENT_TICK` / `CLOCK_TICK` (tick halus) |
| Reject/error (input tidak valid) | `HapticFeedbackConstants.REJECT` (API 30+) atau `LONG_PRESS` sebagai fallback |

**Gunakan `Vibrator`/`VibratorManager` API langsung** hanya untuk kasus yang butuh kontrol pola custom di luar kemampuan `performHapticFeedback` (misalnya notifikasi penting, alarm, game feedback):

```kotlin
val vibrator = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S) {
    context.getSystemService(VibratorManager::class.java).defaultVibrator // API 31+
} else {
    @Suppress("DEPRECATION")
    context.getSystemService(Context.VIBRATOR_SERVICE) as Vibrator
}
// minSdk 29: createPredefined() selalu tersedia, tanpa cabang versi.
vibrator.vibrate(VibrationEffect.createPredefined(VibrationEffect.EFFECT_CLICK))
```

- ❌ **DILARANG** memanggil `vibrate()` mentah dengan durasi panjang sembarangan (`vibrate(500)`) untuk interaksi UI ringan — terasa kasar dan tidak natural.
- ❌ **DILARANG** memberi haptic di SETIAP tap (termasuk navigasi biasa, scroll, switch tab) — ini menyebabkan *haptic fatigue*, justru bikin app terasa murahan.
- ✅ Haptic hanya untuk: konfirmasi aksi destruktif/penting, hasil aksi (sukses/gagal), milestone interaksi (drag-drop, snap, long-press menu), bukan untuk navigasi pasif.
- ✅ Selalu cek permission `VIBRATE` di manifest jika menggunakan `Vibrator` API langsung (umumnya normal permission, tapi tetap wajib dideklarasikan):
```xml
<uses-permission android:name="android.permission.VIBRATE" />
```

### 4.5 Feedback System Secara Umum (Visual, Audio, Haptic)

Setiap aksi user wajib punya **minimal satu bentuk feedback** yang sesuai bobot aksinya:

| Bobot Aksi | Feedback yang Sesuai |
|---|---|
| Ringan (tap, toggle biasa) | Ripple/state visual M3 saja sudah cukup |
| Sedang (submit form, tambah item, like) | Visual (animasi micro-interaction, perubahan state ikon) + opsional haptic ringan |
| Penting/Destruktif (hapus, logout, reset) | Dialog konfirmasi (`MaterialAlertDialogBuilder`) + haptic `CONFIRM`/`LONG_PRESS` + `Snackbar` setelah aksi selesai ("Item dihapus" + tombol Undo jika relevan) |
| Async/Network (loading, sync) | Progress indicator yang jelas statusnya (loading → success/error), JANGAN biarkan UI freeze tanpa indikasi |

### 4.6 Larangan Visual (Anti-Slop)

Gaya yang dituju: modern, minimalis, terang, whitespace lega, hierarki tipografi kuat.

- Dilarang bento grid dan layout yang dipenuhi card. Card hanya untuk satu kelompok konten yang memang butuh batas visual; tidak ada card di dalam card, dan tidak ada card per item list secara default.
- Dilarang gradient dekoratif. Gradient hanya jika fungsional (mis. scrim agar teks terbaca di atas gambar).
- Dilarang emoji sebagai dekorasi UI. Pakai ikon vector.
- Dilarang menambah elemen hanya untuk mengisi ruang kosong.
- Dilarang tema gelap sebagai default (Section 3.1: tema selalu terang).

---

## 5. Pemanfaatan API Android Modern (Bukan Cuma "Jalan")

Agent sering hanya membuat fitur "berjalan" tanpa memanfaatkan API platform yang relevan. Rule wajib cek per kategori fitur:

| Kategori Fitur | API yang Wajib Dipertimbangkan |
|---|---|
| Notifikasi | `NotificationChannel`, `POST_NOTIFICATIONS` runtime permission (API 33+) |
| Background task | `WorkManager` (bukan `Service` mentah / `Thread` manual) |
| File & Media akses | `Photo Picker` (`ActivityResultContracts.PickVisualMedia`) — bukan minta `READ_EXTERNAL_STORAGE` penuh jika tidak perlu |
| Permission | `Runtime Permission` granular sesuai API level, gunakan `ActivityResultContracts.RequestPermission` / `RequestMultiplePermissions` native, dicek lewat `registerForActivityResult` di `Activity`/`Fragment` |
| Sharing | `ShareSheet` via `Intent.ACTION_SEND` + `FileProvider`, bukan custom share UI |
| Predictive Back | Wajib support **Predictive Back Gesture** (Android 13+) — daftarkan `OnBackPressedCallback` lewat `OnBackPressedDispatcher`, jangan override `onBackPressed()` lama tanpa dispatcher |
| Splash Screen | `SplashScreen API` (`core-splashscreen`), dipasang di `themes.xml` (`Theme.SplashScreen`), bukan `Activity` placeholder manual |
| Haptic | `View.performHapticFeedback()` / `Vibrator` API — lihat Section 4.4 untuk detail lengkap pola getar yang benar |
| Battery/Doze | Hindari `WakeLock` jika tugas bisa pakai `WorkManager` constraint-based |
| Tema | Selalu terang (Section 3.1): `MODE_NIGHT_NO` + `android:forceDarkAllowed=false`. Tidak ada `values-night/`. |

---

## 6. Struktur Project (Wajib Diikuti Agent)

```
app/
 └─ src/main/
     ├─ java/<package>/
     │   ├─ data/
     │   │   ├─ local/        (Room, DataStore)
     │   │   ├─ remote/        (jika ada network)
     │   │   └─ repository/
     │   ├─ domain/
     │   │   ├─ model/
     │   │   └─ usecase/
     │   ├─ presentation/
     │   │   ├─ <feature>/
     │   │   │   ├─ <Feature>Activity.kt / <Feature>Fragment.kt
     │   │   │   ├─ <Feature>ViewModel.kt
     │   │   │   ├─ <Feature>UiState.kt
     │   │   │   └─ <Feature>Adapter.kt   (jika ada RecyclerView)
     │   │   └─ common/        (BaseActivity/BaseFragment, shared UI helper)
     │   └─ di/                 (Hilt modules)
     └─ res/
         ├─ layout/             (XML layout per screen/item)
         ├─ values/             (colors.xml, dimens.xml, strings.xml, themes.xml, styles.xml)
         └─ drawable/           (vector drawable, shape drawable, selector)
```

- Setiap `ViewModel` wajib expose `StateFlow<UiState>` (dikoleksi lewat `lifecycleScope.launch { repeatOnLifecycle(...) }` di `Activity`/`Fragment`) — bukan callback lepas tanpa state class.
- Setiap layar wajib punya state: `Loading`, `Success`, `Empty`, `Error` — direpresentasikan lewat toggle visibility antar view (`ProgressBar`, konten utama, empty state layout, error layout) menggunakan `ViewFlipper` atau `View.GONE`/`VISIBLE` terkontrol dari satu fungsi `render(state)` — jangan biarkan UI "diam" tanpa feedback visual saat proses berjalan.

> ⚠️ **Batasan penting (sering disalahartikan agent):** toggle `View.GONE`/`VISIBLE`/`ViewFlipper` di atas **HANYA** berlaku untuk mengganti *state* di dalam **satu layar/Fragment yang sama** (loading ↔ isi ↔ kosong ↔ error). Ini **BUKAN** mekanisme untuk berpindah antar "halaman"/fitur yang berbeda (mis. Home → Profile, atau Home → Detail Item). Perpindahan antar layar/fitur WAJIB lewat **Navigation Component** (`Fragment` + `nav_graph.xml`) sesuai Section 6.1 di bawah. Kalau agent mendapati dirinya mau menyembunyikan satu `ConstraintLayout` besar dan memunculkan `ConstraintLayout` besar lain di root yang sama untuk mensimulasikan "pindah halaman", itu tanda dia salah menerapkan rule ini — harusnya itu jadi Fragment terpisah.

- Gunakan **View Binding** (`viewBinding = true` di `build.gradle.kts`), JANGAN `findViewById` manual atau Kotlin Android Extensions (`synthetic`, sudah deprecated).

### 6.1 Navigasi Multi-Layar (Wajib — Single-Activity + Navigation Component)

Aplikasi ini menggunakan **Single-Activity Architecture**. Satu `MainActivity` menjadi host untuk seluruh `Fragment` di aplikasi, dikelola oleh **Navigation Component** (`androidx.navigation`). Ini bukan opsi — ini adalah struktur wajib untuk semua fitur yang butuh lebih dari satu "layar".

**Urutan pembuatan file wajib** (jangan dibalik):
1. `res/menu/bottom_nav_menu.xml` (atau menu tab lain) — id item harus identik dengan id Fragment tujuan di nav_graph.
2. `res/navigation/nav_graph.xml` — daftarkan semua destination (Fragment), startDestination, dan `<action>` (dengan `<argument>`/Safe Args kalau ada data yang dikirim).
3. Fragment `.kt` + layout `.xml` masing-masing.
4. `activity_main.xml` — `FragmentContainerView` (`app:navGraph`) + `BottomNavigationView`/`TabLayout`.
5. `MainActivity.kt` — wiring `NavController` ke nav bar via `setupWithNavController()`.

**Aturan tambahan:**
- Satu layar = satu unit mandiri: satu `Fragment`, satu layout, satu `ViewModel`, satu `UiState`, dalam package `<feature>` sendiri. Tidak ada Fragment serba-bisa yang menampung logika layar lain.
- Setiap `Fragment` baru **wajib** terdaftar di `nav_graph.xml` sebelum dianggap selesai — Fragment yang tidak terhubung ke graph adalah bug.
- Transisi antar destination wajib pakai `enterAnim`/`exitAnim`/`popEnterAnim`/`popExitAnim` di `<action>`, atau `MaterialSharedAxis`/`MaterialFadeThrough`/`MaterialContainerTransform` (lihat Section 3) — jangan biarkan transisi default tanpa animasi.
- Untuk sub-navigasi di dalam satu section (mis. tab "Semua/Lagu/Album" di dalam satu Fragment), gunakan `TabLayout` + `ViewPager2` + `TabLayoutMediator` — ini valid dan berbeda dari top-level navigation, lihat `AGENTS.md` untuk contoh kode lengkap kedua pattern ini (Bottom Nav & Top Tabs) serta pattern nested graph untuk flow multi-step (onboarding, auth, checkout).
- **DILARANG**: Activity terpisah per fitur/tab (`HomeActivity`, `SearchActivity`, dst) kecuali untuk kasus yang benar-benar butuh Task/back-stack terpisah (mis. full-screen player independen) — dan itu pun harus didiskusikan eksplisit, bukan default.
- **DILARANG**: `FragmentTransaction.replace()` manual di luar Navigation Component untuk perpindahan antar fitur.

Detail contoh kode lengkap (nav_graph.xml, bottom_nav_menu.xml, activity_main.xml, MainActivity.kt, ViewPager2 pattern, nested graph, tabel anti-pattern) ada di `AGENTS.md` Section 1 — dokumen itu melengkapi section ini dengan implementasi penuh siap pakai.

### 6.2 File Wajib di Root Repo

- `.github/workflows/build.yml` (Section 7.3)
- `gradle/libs.versions.toml` dan `gradle.properties` (Section 7.5)
- `tree.md`: struktur file proyek terkini (tanpa `build/`, `.gradle/`, `.idea/`).
- `guide.md`: cara kerja app, dijelaskan file per file (tujuan file, kelas/fungsi utama, hubungan antar file).
- `tree.md` dan `guide.md` diperbarui di setiap task yang menambah, memindah, atau menghapus file. Job `verify` gagal jika keduanya tidak ada (Section 7.3).

---

## 7. GitHub Actions — Build & Sign APK (Strict Flow)

### 7.1 Trigger dan Struktur Job
- Berjalan otomatis pada setiap `push` ke branch `main` saja, plus `workflow_dispatch`. Tanpa trigger tag.
- **Dilarang** auto-deploy ke Play Store / server lain. Output **hanya** GitHub Artifacts.
- `concurrency` dengan `cancel-in-progress: true`: push baru membatalkan run lama pada branch yang sama.

| Job | Isi | Bergantung pada |
|---|---|---|
| `verify` | Key sign verification (secrets + keystore) dan file structure check. Tanpa Gradle, hitungan detik. | - |
| `validate` | Kotlin compile check, Android lint, static analysis (detekt). | `verify` |
| `build-debug` | APK debug (debug keystore standar) + JSON debug (Section 7.6). | `verify` |
| `build-release` | APK release ter-sign dengan keystore dari secrets, diverifikasi `apksigner`. | `verify` |

- `verify` gagal = seluruh pipeline berhenti, tidak ada build yang berjalan.
- `validate`, `build-debug`, `build-release` berjalan paralel agar cepat. APK release tidak menunggu `validate`; jika `validate` merah, run dianggap gagal dan APK-nya jangan dipakai sebelum diperbaiki.

### 7.2 Kebijakan Keystore (WAJIB)

| Build | Keystore | Sumber |
|---|---|---|
| `debug` | Debug keystore standar Android | Dibuat otomatis oleh Android Gradle Plugin. Tidak ada konfigurasi, tidak ada secret. |
| `release` | Keystore pribadi | **GitHub Secrets** (repository secrets). |

**Secrets yang wajib ada** (Settings > Secrets and variables > Actions > **Secrets**, bukan tab Variables):

| Secret | Isi |
|---|---|
| `KEYSTORE_BASE64` | Keystore release di-encode base64 satu baris: `base64 -w 0 release.keystore` |
| `KEYSTORE_PASSWORD` | Password keystore |
| `KEY_ALIAS` | Alias key |
| `KEY_PASSWORD` | Password key |

Aturan:
- Gunakan **Secrets**, bukan **Variables**. Variables disimpan plaintext dan tidak di-mask di log.
- Keystore release **dilarang** di-commit. Tambahkan `release.keystore` dan `*.jks` ke `.gitignore`.
- **Dilarang** menandatangani build `release` dengan debug keystore, termasuk sebagai fallback.
- Konsekuensi: APK debug dan release berbeda signature, sehingga tidak bisa saling menimpa di device. Build debug wajib memakai `applicationIdSuffix = ".debug"` agar keduanya bisa terpasang bersamaan.
- Konsekuensi: runner CI selalu baru, sehingga debug keystore ikut dibuat ulang tiap run dan APK debug dari run berbeda tidak bisa saling update (uninstall dulu). Jika butuh signature debug yang konsisten, commit debug keystore standar (kredensial publik `android` / `androiddebugkey` / `android`, bukan rahasia) dan referensikan di `signingConfigs.debug`. Ini opsional, bukan default.

### 7.3 Workflow

Urutan wajib: **(1) cek secrets ada → (2) validasi keystore → (3) file structure check → (4) build → (5) verifikasi APK dengan `apksigner`**. Langkah 1-3 ada di job `verify` dan harus lolos sebelum job lain berjalan.

```yaml
name: Build APK

on:
  push:
    branches:
      - main
  workflow_dispatch:

concurrency:
  group: build-${{ github.ref }}
  cancel-in-progress: true

jobs:
  verify:
    runs-on: ubuntu-latest
    timeout-minutes: 5
    env:
      KEYSTORE_BASE64: ${{ secrets.KEYSTORE_BASE64 }}
      KEYSTORE_PASSWORD: ${{ secrets.KEYSTORE_PASSWORD }}
      KEY_ALIAS: ${{ secrets.KEY_ALIAS }}
      KEY_PASSWORD: ${{ secrets.KEY_PASSWORD }}
    steps:
      - uses: actions/checkout@v4

      - name: Verify Signing Secrets Exist
        run: |
          for v in KEYSTORE_BASE64 KEYSTORE_PASSWORD KEY_ALIAS KEY_PASSWORD; do
            if [ -z "${!v}" ]; then
              echo "::error::Secret $v kosong atau belum di-set."
              exit 1
            fi
          done

      # KEY_PASSWORD tidak bisa divalidasi oleh keytool -list; salah password key baru gagal saat signing.
      - name: Validate Keystore
        run: |
          KS="$RUNNER_TEMP/verify.keystore"
          echo "$KEYSTORE_BASE64" | base64 -d > "$KS"
          if [ ! -s "$KS" ]; then
            echo "::error::Keystore kosong atau gagal di-decode. Cek secret KEYSTORE_BASE64."
            exit 1
          fi
          if ! keytool -list -keystore "$KS" -storepass "$KEYSTORE_PASSWORD" \
              -alias "$KEY_ALIAS" > "$RUNNER_TEMP/ks.log" 2>&1; then
            echo "::error::Keystore gagal dibuka. Password/alias salah atau file korup."
            cat "$RUNNER_TEMP/ks.log"
            exit 1
          fi
          rm -f "$KS" "$RUNNER_TEMP/ks.log"

      - name: Check File Structure
        run: |
          fail=0
          for f in gradlew settings.gradle.kts gradle.properties gradle/libs.versions.toml \
                   app/build.gradle.kts app/src/main/AndroidManifest.xml \
                   app/src/main/res/navigation/nav_graph.xml \
                   app/src/main/res/values/themes.xml tree.md guide.md; do
            [ -f "$f" ] || { echo "::error::File wajib tidak ditemukan: $f"; fail=1; }
          done
          if [ -d app/src/main/res/values-night ]; then
            echo "::error::values-night/ dilarang: tema selalu terang."; fail=1
          fi
          for d in app/src/debug app/src/release; do
            find "$d" -name 'DebugTools.kt' 2>/dev/null | grep -q . \
              || { echo "::error::DebugTools.kt tidak ada di $d"; fail=1; }
          done
          for f in $(find app/src/main -name '*Fragment.kt' ! -name 'Base*'); do
            n=$(basename "$f" .kt)
            grep -q "$n" app/src/main/res/navigation/nav_graph.xml \
              || { echo "::error::$n belum terdaftar di nav_graph.xml"; fail=1; }
          done
          exit $fail

  validate:
    needs: verify
    runs-on: ubuntu-latest
    timeout-minutes: 20
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: 'temurin'
          java-version: '17'
      - uses: gradle/actions/setup-gradle@v4
      - name: Compile, Lint, Static Analysis
        run: ./gradlew compileDebugKotlin lintDebug detekt --parallel --build-cache

  build-debug:
    needs: verify
    runs-on: ubuntu-latest
    timeout-minutes: 20
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: 'temurin'
          java-version: '17'
      - uses: gradle/actions/setup-gradle@v4

      - name: Build Debug APK
        run: ./gradlew assembleDebug --parallel --build-cache

      - name: Verify Debug APK
        run: |
          APK_PATH=$(find app/build/outputs/apk/debug -name "*.apk" | head -n 1)
          if [ -z "$APK_PATH" ]; then
            echo "::error::Tidak ada APK debug ditemukan setelah build."
            exit 1
          fi
          APKSIGNER=$(ls -d "$ANDROID_HOME"/build-tools/* | sort -V | tail -n 1)/apksigner
          "$APKSIGNER" verify --verbose "$APK_PATH"

      - name: Upload Debug APK
        uses: actions/upload-artifact@v4
        with:
          name: app-debug-run${{ github.run_number }}
          path: app/build/outputs/apk/debug/*.apk

  build-release:
    needs: verify
    runs-on: ubuntu-latest
    timeout-minutes: 20
    env:
      KEYSTORE_BASE64: ${{ secrets.KEYSTORE_BASE64 }}
      KEYSTORE_PASSWORD: ${{ secrets.KEYSTORE_PASSWORD }}
      KEY_ALIAS: ${{ secrets.KEY_ALIAS }}
      KEY_PASSWORD: ${{ secrets.KEY_PASSWORD }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: 'temurin'
          java-version: '17'
      - uses: gradle/actions/setup-gradle@v4

      - name: Decode Keystore
        run: |
          echo "$KEYSTORE_BASE64" | base64 -d > app/release.keystore
          [ -s app/release.keystore ] || { echo "::error::Keystore kosong setelah decode."; exit 1; }

      - name: Build Release APK
        run: ./gradlew assembleRelease --parallel --build-cache

      - name: Verify APK is Signed
        run: |
          APK_PATH=$(find app/build/outputs/apk/release -name "*.apk" | head -n 1)
          if [ -z "$APK_PATH" ]; then
            echo "::error::Tidak ada APK release ditemukan setelah build."
            exit 1
          fi
          APKSIGNER=$(ls -d "$ANDROID_HOME"/build-tools/* | sort -V | tail -n 1)/apksigner
          if ! "$APKSIGNER" verify --verbose "$APK_PATH"; then
            echo "::error::APK release tidak lolos verifikasi signature."
            exit 1
          fi

      - name: Upload Release APK
        uses: actions/upload-artifact@v4
        with:
          name: app-release-run${{ github.run_number }}-signed
          path: app/build/outputs/apk/release/*.apk

      - name: Remove Keystore
        if: always()
        run: rm -f app/release.keystore
```

### 7.4 Aturan `build.gradle.kts` untuk Signing dan Build
```kotlin
android {
    compileSdk = 37
    defaultConfig {
        minSdk = 29
        targetSdk = 37
        // Naik otomatis setiap run CI; fallback 1 untuk build lokal.
        versionCode = System.getenv("GITHUB_RUN_NUMBER")?.toIntOrNull() ?: 1
    }
    buildFeatures { viewBinding = true }
    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }
    lint { checkReleaseBuilds = false } // lint dijalankan di job validate
    signingConfigs {
        // debug: sengaja tidak didefinisikan. AGP memakai debug keystore standar otomatis.
        // release: hanya dikonfigurasi jika env dari workflow (Section 7.3) tersedia.
        if (System.getenv("KEYSTORE_PASSWORD") != null) {
            create("release") {
                storeFile = file("release.keystore")
                storePassword = System.getenv("KEYSTORE_PASSWORD")
                keyAlias = System.getenv("KEY_ALIAS")
                keyPassword = System.getenv("KEY_PASSWORD")
            }
        }
    }
    buildTypes {
        debug {
            applicationIdSuffix = ".debug"
            versionNameSuffix = "-debug"
        }
        release {
            signingConfigs.findByName("release")?.let { signingConfig = it }
            isMinifyEnabled = false // R8 opsional, lihat Section 7.5
        }
    }
}

detekt {
    buildUponDefaultConfig = true
    parallel = true
}
```
Gunakan AGP stabil terbaru yang mendukung API 37 (minimal 8.9.0), Kotlin dan Gradle terbaru yang kompatibel. Tanpa env keystore (build lokal), `assembleRelease` menghasilkan APK unsigned. Itu disengaja; signature release hanya dijamin oleh CI dan diverifikasi dengan `apksigner`.

### 7.5 Gradle Ringan (Cepat, Tanpa Menjadi Bottleneck Kode Modern)

Tujuan: build cepat lewat konfigurasi dan cache, bukan lewat memangkas kualitas kode.

`gradle.properties`:
```properties
org.gradle.jvmargs=-Xmx2g -XX:+UseParallelGC -Dfile.encoding=UTF-8
org.gradle.parallel=true
org.gradle.caching=true
org.gradle.configuration-cache=true
android.useAndroidX=true
android.nonTransitiveRClass=true
```

Aturan:
- Satu modul `:app`. Multi-module hanya jika user memintanya.
- Version catalog `gradle/libs.versions.toml` sebagai satu-satunya sumber versi.
- **KSP** untuk Hilt, Room, dan processor lain. **Dilarang kapt.**
- Hanya plugin dan dependency yang benar-benar dipakai. Tanpa plugin atau library dekoratif.
- Release `isMinifyEnabled = false` secara default (R8 menambah waktu build dan risiko keep-rule). Aktifkan R8 hanya jika user meminta APK lebih kecil atau ter-obfuscate.
- CI: `gradle/actions/setup-gradle` (cache), `--parallel --build-cache`, `concurrency` cancel, tanpa `--info`/`--debug`.
- Nonaktifkan `org.gradle.configuration-cache` hanya jika ada plugin yang terbukti belum kompatibel, dan catat alasannya di `guide.md`.

Kecepatan build **bukan alasan** menolak atau mengganti pendekatan modern. Tetap dipakai: Kotlin terbaru (K2), Coroutines + Flow, Hilt (KSP), Room, Navigation Component, Paging 3, WorkManager, DataStore, ViewBinding, Material terbaru. Jika build melambat, optimasi lewat cache, paralel, dan incremental lebih dulu, bukan dengan menghapus fitur atau menurunkan kualitas kode.

### 7.6 Debug Build: JSON Debug di `Android/data`
- Build debug wajib menulis `debug.json` ke `getExternalFilesDir(null)`, yaitu `Android/data/<package>/files/debug.json` (tanpa permission storage).
- Isi minimal: info build (applicationId, versionName, versionCode), info device (model, versi Android, SDK), event/log terbaru (ring buffer maksimum 200 entri), dan crash terakhir (stack trace) jika ada.
- Diimplementasikan lewat source set: `app/src/debug/java/.../DebugTools.kt` (implementasi) dan `app/src/release/java/.../DebugTools.kt` (no-op, package dan API identik). Sengaja tidak ada di `src/main`. Release tidak boleh berisi kode debug ini.
- `DebugTools.init(this)` dipanggil dari `Application.onCreate()`. Kode referensi ada di `AGENTS.md` Section 14.
- Akses file: Android 11+ membatasi akses aplikasi lain ke `Android/data`. Ambil lewat `adb pull`, Device Explorer Android Studio, atau transfer USB.

### 7.7 Checklist Rilis
- [ ] Empat secret (`KEYSTORE_BASE64`, `KEYSTORE_PASSWORD`, `KEY_ALIAS`, `KEY_PASSWORD`) terdaftar di tab **Secrets**, bukan Variables.
- [ ] Job `verify` lolos (secrets, keystore, file structure) sebelum job lain berjalan.
- [ ] Job `validate` hijau (compile, lint, detekt).
- [ ] Release ditandatangani keystore dari secrets; debug memakai debug keystore standar. Tidak ada fallback release ke debug keystore.
- [ ] APK release lolos `apksigner verify`.
- [ ] Build debug menghasilkan `debug.json`; release hanya berisi `DebugTools` no-op.
- [ ] Build debug memakai `applicationIdSuffix = ".debug"`.
- [ ] `versionCode` berasal dari `GITHUB_RUN_NUMBER`, bukan tag git.
- [ ] Artifact bernama jelas: `app-debug-run<N>` dan `app-release-run<N>-signed`.
- [ ] `tree.md` dan `guide.md` ada dan terbaru.
- [ ] Keystore dan password tidak ter-commit, tidak tercetak di log, dan tidak ikut ter-upload sebagai artifact.

---

## 8. Definisi "Selesai" (Definition of Done)

Sebuah task TIDAK dianggap selesai jika hanya "compile berhasil". Agent wajib pastikan:

1. ✅ App build tanpa warning kritikal dari lint Android.
2. ✅ Edge-to-edge & status bar sesuai Section 2 (visually verified secara konsep, bukan asumsi).
3. ✅ Tidak ada hardcoded string UI (gunakan `strings.xml` / resource, walau app single-language) dan tidak ada hardcoded dimensi (gunakan `dimens.xml`, lihat Section 4.2).
4. ✅ Tidak ada `TODO` kosong atau fitur stub yang dibiarkan tanpa state handling.
5. ✅ Setiap layout XML kompleks sudah dicek dengan **Layout Inspector** / preview `tools:` namespace di Android Studio (`tools:text`, `tools:visibility`) untuk mempermudah review visual tanpa perlu run aplikasi penuh.
6. ✅ Tidak ada `View Binding` yang di-null-kan secara salah (khususnya di `Fragment` — wajib set `_binding = null` di `onDestroyView()` untuk mencegah memory leak).
7. ✅ APK ter-build dan ter-sign lewat GitHub Actions, muncul di tab Artifacts.
8. ✅ `tree.md` dan `guide.md` ada dan mencerminkan kondisi proyek terkini.
9. ✅ Tema terang saja: tidak ada `values-night/`, dark mode, atau layout bento/card-heavy (Section 3.1 dan 4.6).
10. ✅ Job `verify` dan `validate` hijau; build debug menghasilkan `debug.json` (Section 7.6).

---

## 9. Larangan Umum (Red Flags "Amatir")

- ❌ Menggunakan `Toast` sebagai satu-satunya bentuk feedback untuk aksi penting (gunakan `Snackbar` dari library `material` atau `MaterialAlertDialogBuilder` yang sesuai).
- ❌ Loading state hanya `ProgressBar` polos di tengah layar tanpa skeleton/shimmer (`ShimmerFrameLayout` atau custom shimmer drawable) untuk list/grid.
- ❌ Spacing tidak konsisten (campur `8dp`, `10dp`, `12dp` acak langsung di XML) — wajib pakai token `@dimen/spacing_*` dari Section 4.2.
- ❌ Tidak ada empty state ilustratif (list kosong hanya menampilkan layar putih kosong — wajib toggle ke layout empty state).
- ❌ Tombol/icon tidak punya `android:contentDescription` (accessibility diabaikan) — untuk elemen dekoratif murni, set eksplisit `android:importantForAccessibility="no"`.
- ❌ Tidak ada haptic/animasi feedback sama sekali pada interaksi penting.
- ❌ Menggunakan `findViewById` manual atau Kotlin synthetic import padahal View Binding sudah wajib diaktifkan.
- ❌ `RecyclerView` tanpa `DiffUtil`/`ListAdapter` (`notifyDataSetChanged()` mentah tanpa animasi item, terasa kasar dan tidak efisien).
- ❌ Dark mode, `values-night/`, atau `Theme.Material3.DayNight` dipakai tanpa diminta user.
- ❌ Bento grid, card di dalam card, atau card per item list sebagai default.
- ❌ Gradient dekoratif atau emoji sebagai elemen UI.
- ❌ `kapt` dipakai padahal KSP tersedia; modul tambahan atau plugin yang tidak perlu.
- ❌ Menulis kode sebelum plan disetujui user.

---

## 10. Instruksi Akhir untuk Agent

> Ikuti Section 0.1: buat plan (arsitektur, struktur file, pembagian kerja), tunggu persetujuan user, baru tulis kode lengkap (layout XML + Kotlin), lalu pasang workflow dan dokumentasi (`tree.md`, `guide.md`). Plan UI/UX harus menjelaskan bagaimana Section 2 (Edge-to-Edge), Section 3 (tema terang & expressive), Section 4 (Desain & Hierarki Visual), dan Section 6.1 (Navigasi) diterapkan di tiap layar. Jangan submit kode yang melanggar rule strict (Section 0.2, 2, 3, 4, 6.1, 7, dan 8 paling sering dilanggar, periksa dua kali). Rujuk `AGENTS.md` untuk implementasi navigasi, ikon/SVG, motion, tema, dan debug tools.
