# AGENTS.md — Android Native (Kotlin + XML) Advanced Reference

> **Status dokumen: PENDAMPING / OPSIONAL terhadap `ANDROID_AGENT_RULES.md`.**
> `ANDROID_AGENT_RULES.md` adalah rulebook strict-wajib project (spesifikasi, arsitektur
> data/DI, build & signing, dan garis besar desain/navigasi di Section 4 & 6.1).
> `AGENTS.md` ini adalah **referensi teknis lanjutan** yang dipakai agent untuk
> *implementasi detail* — contoh kode siap pakai, pola-pola lanjutan, dan area yang
> belum tercakup granular di rulebook utama (SVG/icon system, motion detail, advanced
> navigation patterns, design tokens, dsb).
>
> **Cara membaca:** kalau `ANDROID_AGENT_RULES.md` bilang "wajib pakai Navigation
> Component" (Section 6.1), dokumen ini yang menunjukkan *caranya* — nav_graph.xml
> lengkap, urutan file, pattern Bottom Nav vs Top Tabs, dst. Kalau ada bagian di sini
> yang terasa opsional/tambahan (bukan hard requirement), akan ditandai eksplisit
> dengan label **(Opsional)**. Selain itu, semua tetap dianggap wajib mengikuti level
> ketegasan yang sama dengan rulebook utama.
>
> **Kalau ada konflik aturan antara dua dokumen:** `ANDROID_AGENT_RULES.md` menang untuk
> spesifikasi proyek, arsitektur data/DI, dan build/signing. `AGENTS.md` (dokumen ini)
> menang untuk detail implementasi navigasi UI, desain visual, ikonografi, dan motion.

Dokumen ini dipakai AI agent (Jules, atau agent lain) saat membangun aplikasi Android
native berbasis **Kotlin + XML Views** (bukan Jetpack Compose). Tujuan: hasil akhir
harus terasa seperti dibuat oleh tim produk profesional — bukan "AI-generated looking" —
dengan arsitektur navigasi yang benar, desain modern, dan UX yang matang.

Semua aturan di sini bersifat **default wajib** kecuali ditandai **(Opsional)**, atau
project punya instruksi lain yang eksplisit mengesampingkannya (tulis pengecualian di
README project, bukan di sini).

---

## 0. Prinsip Umum

1. **Jangan pernah fallback ke "single screen, toggle visibility"** untuk mensimulasikan
   banyak halaman. Setiap "halaman" adalah `Fragment` sungguhan yang dikelola Navigation
   Component. Mengganti `View.VISIBLE`/`GONE` untuk berpindah "layar" adalah ANTI-PATTERN
   dan dilarang.
2. **Baca dulu, jangan generate dulu.** Sebelum menulis kode, cek struktur project yang
   sudah ada: `res/navigation/`, `res/menu/`, package `ui/`, `di/` (Hilt modules).
   Jangan duplikasi Fragment/ViewModel yang sudah ada dengan nama berbeda.
3. **Satu Activity untuk seluruh app** (Single-Activity Architecture), kecuali ada
   kebutuhan eksplisit untuk Activity terpisah (mis. full-screen player, onboarding
   flow yang benar-benar independen, atau proses yang butuh Task terpisah).
4. Setiap perubahan yang menyentuh navigasi **wajib** menyertakan update ke
   `nav_graph.xml` — tidak boleh ada Fragment "yatim" yang tidak terdaftar di graph.
5. Sebelum build, verifikasi keystore & signing config sesuai
   `ANDROID_AGENT_RULES.md` (minSdk 29 / targetSdk 37, MVVM + clean-ish layering, Hilt via KSP,
   tema selalu terang).
6. Alur kerja mengikuti persona STARK (`ANDROID_AGENT_RULES.md` Section 0): plan, confirm,
   build, ship, document (`tree.md`, `guide.md`).
7. Build ringan: satu modul, KSP, cache. Kecepatan build tidak boleh mengorbankan kode
   modern yang baik.

---

## 1. Arsitektur Navigasi (WAJIB)

### 1.1 Struktur file minimum untuk navigasi multi-halaman

```
app/src/main/
├── java/.../ui/
│   ├── MainActivity.kt
│   ├── home/HomeFragment.kt
│   ├── search/SearchFragment.kt
│   ├── library/LibraryFragment.kt
│   └── profile/ProfileFragment.kt
└── res/
    ├── layout/
    │   ├── activity_main.xml
    │   ├── fragment_home.xml
    │   ├── fragment_search.xml
    │   ├── fragment_library.xml
    │   └── fragment_profile.xml
    ├── navigation/
    │   └── nav_graph.xml
    └── menu/
        └── bottom_nav_menu.xml
```

### 1.2 Urutan pembuatan file (agent WAJIB ikuti urutan ini)

1. `res/menu/bottom_nav_menu.xml` — definisikan item menu dulu (id, icon, title).
2. `res/navigation/nav_graph.xml` — definisikan semua destination (fragment) + startDestination
   + action antar fragment (termasuk argumen kalau ada, pakai Safe Args).
3. Fragment `.kt` + layout `.xml` masing-masing — dibuat sesuai destination yang sudah
   didefinisikan di nav_graph.
4. `activity_main.xml` — host `FragmentContainerView` (dengan `app:navGraph`) +
   `BottomNavigationView`/`TabLayout`.
5. `MainActivity.kt` — wiring `NavController` ke `BottomNavigationView` via
   `bottomNav.setupWithNavController(navController)`.

Jangan pernah menulis langkah 4-5 sebelum langkah 1-3 selesai — ini penyebab utama
navigasi yang "setengah jadi" (UI nav bar ada, tapi gak connect ke apa-apa).

### 1.3 Pattern A — Bottom Navigation (paling umum, default pilihan)

**`res/navigation/nav_graph.xml`**
```xml
<?xml version="1.0" encoding="utf-8"?>
<navigation xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:id="@+id/nav_graph"
    app:startDestination="@id/homeFragment">

    <fragment
        android:id="@+id/homeFragment"
        android:name="com.example.app.ui.home.HomeFragment"
        android:label="Home"
        tools:layout="@layout/fragment_home" />

    <fragment
        android:id="@+id/searchFragment"
        android:name="com.example.app.ui.search.SearchFragment"
        android:label="Search"
        tools:layout="@layout/fragment_search" />

    <fragment
        android:id="@+id/libraryFragment"
        android:name="com.example.app.ui.library.LibraryFragment"
        android:label="Library"
        tools:layout="@layout/fragment_library" />

    <fragment
        android:id="@+id/profileFragment"
        android:name="com.example.app.ui.profile.ProfileFragment"
        android:label="Profile"
        tools:layout="@layout/fragment_profile" />

    <!-- Contoh navigasi dengan argumen, mis. detail item dari Home -->
    <fragment
        android:id="@+id/detailFragment"
        android:name="com.example.app.ui.detail.DetailFragment"
        android:label="Detail">
        <argument
            android:name="itemId"
            app:argType="string" />
    </fragment>

    <action
        android:id="@+id/action_home_to_detail"
        app:destination="@id/detailFragment"
        app:enterAnim="@anim/slide_in_right"
        app:exitAnim="@anim/slide_out_left"
        app:popEnterAnim="@anim/slide_in_left"
        app:popExitAnim="@anim/slide_out_right" />
</navigation>
```

**`res/menu/bottom_nav_menu.xml`**
```xml
<?xml version="1.0" encoding="utf-8"?>
<menu xmlns:android="http://schemas.android.com/apk/res/android">
    <item
        android:id="@+id/homeFragment"
        android:icon="@drawable/ic_home"
        android:title="@string/nav_home" />
    <item
        android:id="@+id/searchFragment"
        android:icon="@drawable/ic_search"
        android:title="@string/nav_search" />
    <item
        android:id="@+id/libraryFragment"
        android:icon="@drawable/ic_library"
        android:title="@string/nav_library" />
    <item
        android:id="@+id/profileFragment"
        android:icon="@drawable/ic_profile"
        android:title="@string/nav_profile" />
</menu>
```
> **PENTING:** id item menu HARUS sama persis dengan id fragment di nav_graph.
> Ini yang membuat `setupWithNavController` bisa auto-sync tanpa listener manual.

**`res/layout/activity_main.xml`**
```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.coordinatorlayout.widget.CoordinatorLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:id="@+id/root"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:fitsSystemWindows="true">

    <androidx.fragment.app.FragmentContainerView
        android:id="@+id/nav_host_fragment"
        android:name="androidx.navigation.fragment.NavHostFragment"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        app:layout_behavior="@string/appbar_scrolling_view_behavior"
        app:defaultNavHost="true"
        app:navGraph="@navigation/nav_graph" />

    <com.google.android.material.bottomnavigation.BottomNavigationView
        android:id="@+id/bottom_nav"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_gravity="bottom"
        app:menu="@menu/bottom_nav_menu"
        app:labelVisibilityMode="labeled"
        style="@style/Widget.Material3.BottomNavigationView" />

</androidx.coordinatorlayout.widget.CoordinatorLayout>
```

**`MainActivity.kt`**
```kotlin
@AndroidEntryPoint
class MainActivity : AppCompatActivity() {

    private lateinit var binding: ActivityMainBinding

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityMainBinding.inflate(layoutInflater)
        setContentView(binding.root)

        val navHostFragment = supportFragmentManager
            .findFragmentById(R.id.nav_host_fragment) as NavHostFragment
        val navController = navHostFragment.navController

        binding.bottomNav.setupWithNavController(navController)

        // Sembunyikan bottom nav di layar tertentu (mis. detail/player full screen)
        navController.addOnDestinationChangedListener { _, destination, _ ->
            binding.bottomNav.isVisible = destination.id in setOf(
                R.id.homeFragment, R.id.searchFragment,
                R.id.libraryFragment, R.id.profileFragment
            )
        }
    }
}
```

### 1.4 Pattern B — Top Tabs (TabLayout + ViewPager2)

Dipakai untuk sub-navigasi *di dalam* satu section (mis. tab "Semua / Lagu / Album / Artis"
di dalam LibraryFragment), BUKAN pengganti bottom nav untuk top-level navigation.

```xml
<androidx.appcompat.widget.Toolbar
    android:id="@+id/toolbar"
    android:layout_width="match_parent"
    android:layout_height="?attr/actionBarSize" />

<com.google.android.material.tabs.TabLayout
    android:id="@+id/tab_layout"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    app:tabMode="fixed"
    app:tabGravity="fill"
    style="@style/Widget.Material3.TabLayout" />

<androidx.viewpager2.widget.ViewPager2
    android:id="@+id/view_pager"
    android:layout_width="match_parent"
    android:layout_height="0dp"
    android:layout_weight="1" />
```

```kotlin
val adapter = LibraryPagerAdapter(this)
viewPager.adapter = adapter
TabLayoutMediator(tabLayout, viewPager) { tab, position ->
    tab.text = when (position) {
        0 -> "Semua"; 1 -> "Lagu"; 2 -> "Album"; else -> "Artis"
    }
}.attach()
```

### 1.5 Nested Graph untuk flow kompleks

Untuk flow multi-step (onboarding, checkout, auth), gunakan **nested navigation graph**
di dalam `nav_graph.xml` utama, bukan Activity terpisah:
```xml
<navigation
    android:id="@+id/auth_graph"
    app:startDestination="@id/loginFragment">
    <fragment android:id="@+id/loginFragment" .../>
    <fragment android:id="@+id/registerFragment" .../>
</navigation>
```

### 1.6 Anti-pattern yang DILARANG

| ❌ Dilarang | ✅ Yang benar |
|---|---|
| Satu Activity per "tab" (`HomeActivity`, `SearchActivity`, dst) | Satu Activity, banyak Fragment via nav_graph |
| Toggle `View.GONE`/`VISIBLE` untuk ganti "halaman" dalam 1 Fragment | Fragment terpisah + `navController.navigate()` |
| Manual `FragmentTransaction.replace()` tanpa Navigation Component | Selalu lewat `NavController` + `nav_graph.xml` |
| Back stack dikelola manual dengan `popBackStack()` sembarangan | Gunakan `popUpTo` + `popUpToInclusive` di action XML |
| Hardcode fragment id di banyak tempat | Reference `R.id.xxxFragment` dari nav_graph, konsisten di semua file |

---

## 2. Visual Hierarchy & Modern Design System

### 2.1 Material Design 3 sebagai fondasi wajib

- Selalu extend `Theme.Material3Expressive.Light.NoActionBar` (material 1.14.0+). Fallback
  `Theme.Material3.Light.NoActionBar` jika library lebih lama. Tema selalu terang: tanpa
  `DayNight`, tanpa `values-night/`.
- **Dynamic Color: opsional, default mati** (lihat `ANDROID_AGENT_RULES.md` Section 3.1). Jika
  diaktifkan: `DynamicColors.applyToActivitiesIfAvailable(this)` di `Application`, hanya
  Android 12+, dengan palet brand sebagai fallback.
- Definisikan color scheme terang lengkap di `res/values/colors.xml` mengikuti role Material 3:
  `primary`, `onPrimary`, `primaryContainer`, `onPrimaryContainer`, `secondary`, `surface`,
  `surfaceVariant`, `onSurface`, `outline`, dst. Jangan hardcode hex color langsung di layout XML.
- Accent palette default project ini: **cyan/teal**, konsisten di semua state
  (pressed, focused, disabled) — sesuai preferensi UI yang sudah ditetapkan.
- Template tema lengkap ada di Section 15.

### 2.2 Tipografi

- Gunakan Material 3 type scale (`displayLarge` s/d `labelSmall`) via `TextAppearance.Material3.*`.
  Jangan set `textSize` manual acak per screen.
- Maksimal 2 font family dalam satu app (mis. satu untuk display/headline, satu untuk body).
- Line height dan letter spacing ikut default Material 3 type scale kecuali ada alasan desain
  eksplisit untuk override.

### 2.3 Spacing & Grid

- Gunakan sistem spacing kelipatan **4dp** (4, 8, 12, 16, 24, 32, 48...). Jangan pakai angka
  sembarangan seperti `13dp` atau `21dp`.
- Margin horizontal standar layar: `16dp` (compact), `24dp` (medium/tablet width).
- Konsisten pakai `dimens.xml` untuk spacing yang dipakai berulang, jangan hardcode di
  banyak layout berbeda.

### 2.4 Elevation, Shape, dan Depth

- Gunakan `Shape` system Material 3 (`shapeAppearanceSmallComponent/MediumComponent/LargeComponent`)
  lewat theme overlay, bukan `cornerRadius` manual di tiap komponen.
- Elevation harus mencerminkan hierarki: surface dasar = 0dp, komponen terangkat = 1-3dp, FAB/dialog = 6-8dp,
  bottom sheet/nav saat scroll = dynamic elevation overlay (bukan shadow statis warna hitam).
- **Shape expressive:** variasikan radius antar level hierarki (Medium 16dp, Large 28dp,
  ExtraLarge 40dp, pill untuk aksi utama) lewat `ShapeAppearance` terpusat. Pakai
  `Widget.Material3Expressive.*` yang tersedia di versi library terpasang; jangan mengarang
  nama style.
- Elevasi di tema terang dibangun dari tonal surface (`?attr/colorSurfaceContainer*`) dan
  shadow tipis, bukan shadow berat.
- Larangan visual (bento grid, card-heavy, gradient dekoratif, emoji dekoratif): lihat
  `ANDROID_AGENT_RULES.md` Section 4.6.

### 2.5 Iconography & SVG

- Semua icon HARUS vector (`VectorDrawable`/`AnimatedVectorDrawable`), bukan PNG/raster.
- Style icon konsisten dalam satu app: pilih salah satu — **outlined** atau **filled** —
  jangan campur, kecuali untuk menandakan state aktif/tidak aktif (mis. outline = inactive,
  filled = active, ini pattern yang valid untuk bottom nav).
- Ukuran optical: 24dp x 24dp viewport standar untuk icon UI, dengan padding internal
  konsisten (biasanya 2dp dari edge) supaya optical weight seragam antar icon.
- Stroke width konsisten (untuk outlined icons) — jangan campur icon dengan stroke 1.5 dan 2.
- Saat generate/import SVG → VectorDrawable, cek `pathData` tidak corrupt dan `viewportWidth/Height`
  konsisten 24x24 (atau grid lain yang dipilih, tapi harus seragam project-wide).

---

## 3. Interaksi, Motion & Haptic Feedback

### 3.1 Motion

- Semua transisi antar Fragment via nav_graph WAJIB pakai custom animasi
  (`enterAnim`/`exitAnim`/`popEnterAnim`/`popExitAnim`), bukan default instan tanpa transisi.
- Durasi standar Material 3: 200-300ms untuk transisi kecil (fade, elevation), 300-400ms
  untuk transisi besar (page transition, shared element).
- Gunakan `MaterialSharedAxis`, `MaterialFadeThrough`, atau `MaterialContainerTransform`
  dari `com.google.android.material.transition` untuk transisi antar Fragment, bukan
  Animator XML manual generik.
- Easing: gunakan curve standar Material (`FastOutSlowIn`, `emphasized` interpolators),
  hindari `LinearInterpolator` untuk transisi UI (terasa kaku/robotic).

### 3.2 Haptic Feedback (WAJIB diimplementasi, bukan opsional)

Gunakan `View.performHapticFeedback()` dengan constant yang sesuai konteks — jangan pakai
satu jenis haptic untuk semua interaksi:

| Aksi | Haptic Constant |
|---|---|
| Tap tombol biasa/navigasi | `HapticFeedbackConstants.VIRTUAL_KEY` |
| Toggle switch/checkbox | `HapticFeedbackConstants.TOGGLE_ON` / `TOGGLE_OFF` |
| Konfirmasi aksi penting (delete, submit) | `HapticFeedbackConstants.CONFIRM` |
| Reject/error/invalid action | `HapticFeedbackConstants.REJECT` |
| Long-press context menu | `HapticFeedbackConstants.LONG_PRESS` |
| Drag & drop reorder item | `HapticFeedbackConstants.GESTURE_START` saat grab, feedback tiap posisi berubah |

- Untuk API 30+, pertimbangkan `VibrationEffect.createPredefined()` (`EFFECT_TICK`,
  `EFFECT_CLICK`, `EFFECT_HEAVY_CLICK`) lewat `Vibrator`/`VibratorManager` untuk kontrol
  lebih presisi dibanding `performHapticFeedback` generik, khususnya di komponen custom.
- Haptic harus bisa di-disable lewat Settings app (respect `Settings.System.HAPTIC_FEEDBACK_ENABLED`
  dan preferensi in-app kalau ada) — jangan paksa vibrate kalau user sudah mematikannya di OS.
- Jangan overuse: haptic hanya untuk aksi yang punya makna (konfirmasi, state change, error),
  bukan untuk setiap scroll/tap kosmetik.

### 3.3 Ripple & Touch Feedback

- Semua elemen yang clickable wajib punya ripple (`?attr/selectableItemBackground` untuk
  yang punya background, `?attr/selectableItemBackgroundBorderless` untuk icon button tanpa
  background terlihat).
- Touch target minimum **48dp x 48dp** meski ukuran visual icon lebih kecil (pakai padding,
  bukan memperbesar icon-nya).

---

## 4. Edge-to-Edge & Status/Navigation Bar

- Semua Activity WAJIB edge-to-edge (`enableEdgeToEdge()` dari `androidx.activity`),
  konsisten dengan `ANDROID_AGENT_RULES.md`.
- Gunakan `WindowInsetsCompat`/`ViewCompat.setOnApplyWindowInsetsListener` untuk padding
  konten mengikuti system bars — jangan hardcode `statusBarHeight` manual.
- Root layout pakai `android:fitsSystemWindows` HANYA jika memang mengandalkan default
  inset handling; kalau custom (mis. gambar full bleed di belakang status bar), handle
  insets manual per-view yang butuh saja.
- Tema selalu terang, jadi ikon status bar dan navigation bar selalu gelap
  (`isAppearanceLightStatusBars = true`, `isAppearanceLightNavigationBars = true`). Override
  hanya untuk layar dengan background gelap penuh (mis. viewer foto), lalu kembalikan saat keluar.

---

## 5. Struktur Layar & Komponen Umum

### 5.1 Anatomi layar standar
1. **Top App Bar** (`MaterialToolbar` / `CollapsingToolbarLayout` untuk layar dengan hero image)
2. **Content area** — scrollable (`NestedScrollView`/`RecyclerView`), dengan state:
   loading (skeleton/shimmer, BUKAN spinner polos di tengah layar untuk list content),
   empty state (ilustrasi + copy jelas + CTA jika relevan), error state (pesan + retry button).
3. **Bottom Navigation** (top-level) — hanya muncul di top-level destinations.
4. **FAB** — hanya jika ada 1 aksi primer yang jelas untuk screen tersebut, jangan taruh FAB
   di setiap layar tanpa fungsi jelas.

### 5.2 List & RecyclerView

- Selalu pakai `DiffUtil`/`ListAdapter` — jangan `notifyDataSetChanged()` manual.
- Gunakan `ViewBinding` per item, bukan `findViewById` manual.
- Item touch target dan divider mengikuti spacing 4dp grid (lihat 2.3).
- Untuk list panjang, pertimbangkan pagination (`Paging 3` library) daripada load semua
  sekaligus.

### 5.3 Form & Input

- Gunakan `TextInputLayout` (Material) dengan style `OutlinedBox`/`FilledBox` konsisten
  se-app, bukan `EditText` polos.
- Validasi realtime dengan pesan error jelas di `TextInputLayout.error`, bukan Toast.
- Keyboard `imeOptions` diset benar (`actionNext`/`actionDone`) supaya flow input antar
  field lancar tanpa user harus tap manual ke field berikutnya.

### 5.4 Feedback ke user

- Aksi ringan/reversible → `Snackbar` (dengan action "Undo" jika applicable).
- Aksi butuh konfirmasi destruktif (delete, logout) → `MaterialAlertDialogBuilder`, bukan
  langsung eksekusi.
- Jangan pakai `Toast` untuk error yang butuh aksi user — Toast tidak actionable dan
  gampang terlewat.

---

## 6. Aksesibilitas (Bukan Opsional)

- Semua `ImageView`/`ImageButton` fungsional wajib punya `contentDescription` (atau
  `android:importantForAccessibility="no"` jika memang dekoratif).
- Kontras warna teks terhadap background minimum rasio **4.5:1** (body text) dan **3:1**
  (large text/icon), sesuai WCAG AA — cek terutama untuk teks di atas warna accent/surface.
- Semua touch target ≥48dp (lihat 3.3).
- Dukung dynamic font scaling (`sp` untuk teks, jangan `dp`) — test layout di font scale 130%.
- TalkBack: pastikan urutan fokus logis (top-to-bottom, left-to-right) dan grup elemen
  terkait (mis. rating + label) digabung jadi satu accessibility node kalau relevan.

---

## 7. Checklist Sebelum Build/Commit

- [ ] Semua Fragment baru terdaftar di `nav_graph.xml`
- [ ] Bottom nav / tab item id sinkron dengan fragment id
- [ ] Tidak ada `View.GONE`/`VISIBLE` dipakai untuk simulasi navigasi antar "halaman"
- [ ] Transisi antar layar punya animasi (bukan default instan)
- [ ] Warna diambil dari theme attrs (`?attr/colorPrimary` dst), bukan hex hardcode
- [ ] Semua icon adalah VectorDrawable, style konsisten (outlined/filled tidak campur)
- [ ] Spacing pakai grid 4dp / `dimens.xml`
- [ ] Interaksi penting (konfirmasi, toggle, error) sudah punya haptic feedback yang sesuai
- [ ] Touch target ≥48dp, ripple ada di semua elemen clickable
- [ ] Edge-to-edge aktif, insets di-handle dengan benar (bukan hardcode padding)
- [ ] Empty/loading/error state ada untuk setiap screen dengan data async
- [ ] `contentDescription` lengkap untuk elemen fungsional
- [ ] Keystore & signing config terverifikasi sebelum build APK (lihat `ANDROID_AGENT_RULES.md`)
- [ ] Tema terang saja: tidak ada `values-night/` atau dark mode
- [ ] Tidak ada bento grid, layout card-heavy, gradient dekoratif, atau emoji sebagai elemen UI
- [ ] `tree.md` dan `guide.md` diperbarui
- [ ] Build ringan: satu modul, KSP (bukan kapt), tanpa plugin/library tak terpakai

---

## 8. Referensi Silang

Dokumen ini melengkapi, bukan menggantikan, `ANDROID_AGENT_RULES.md` yang sudah ada
(minSdk 29/targetSdk 37, MVVM + clean-ish layering, Hilt, tema terang, build ringan,
GitHub Actions + apksigner, Section 6.1 navigasi). Kalau ada konflik aturan, `ANDROID_AGENT_RULES.md` yang menang untuk hal
arsitektur data/DI/build/signing, dokumen ini yang menang untuk hal navigasi UI, desain
visual, ikonografi, dan motion.

---

## 9. Sistem Ikonografi & SVG Lanjutan

### 9.1 Anatomi & Grid SVG (24dp standar)

- Semua icon dibangun di atas grid **24x24dp** dengan **live area 20x20dp** (padding optik
  2dp tiap sisi), kecuali icon yang secara visual "penuh" (mis. lingkaran, kotak) boleh
  sedikit melebihi live area untuk optical balance — ini pola standar Material Symbols.
- Stroke width konsisten: **2dp** untuk outlined icons di grid 24dp (skala proporsional
  kalau grid berbeda, mis. 1.5dp untuk grid 20dp).
- Corner radius sudut icon: konsisten, biasanya `2dp` untuk sudut tajam yang dihaluskan.
- End caps garis: `round` untuk gaya "friendly/modern", `butt`/`square` untuk gaya
  "technical/sharp" — pilih satu gaya, jangan campur dalam satu icon set.

### 9.2 Konversi SVG → VectorDrawable

- Field wajib dicek saat generate/import: `android:width="24dp"`, `android:height="24dp"`,
  `android:viewportWidth="24"`, `android:viewportHeight="24"`.
- `pathData` HARUS pakai `android:fillColor="?attr/colorControlNormal"` atau
  `?attr/colorOnSurface` (theme-aware), bukan hex statis (`#000000`/`#FFFFFF`) — supaya
  icon otomatis mengikuti tint tema terang dan state komponen.
- Untuk icon dengan multi-warna (brand logo dsb, yang memang harus warna tetap), boleh
  pakai `fillColor` hex statis, tapi ini adalah **pengecualian eksplisit**, bukan default.
- Icon dengan animasi state (mis. play↔pause, hamburger↔close) gunakan
  `AnimatedVectorDrawable` dengan `AnimatedVectorDrawableCompat`, bukan crossfade dua
  drawable statis terpisah — hasilnya jauh lebih halus dan terasa "native".

```xml
<!-- res/drawable/ic_play_pause.xml -->
<animated-vector xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:aapt="http://schemas.android.com/aapt"
    android:drawable="@drawable/ic_play_pause_base">
    <target android:name="play_path" android:animation="@animator/play_to_pause" />
</animated-vector>
```

### 9.3 Tinting & State-based Icon (Selector)

Gunakan `ColorStateList` untuk icon yang berubah warna sesuai state (aktif/nonaktif,
selected/unselected di bottom nav) — bukan swap drawable manual per state:

```xml
<!-- res/color/bottom_nav_icon_tint.xml -->
<selector xmlns:android="http://schemas.android.com/apk/res/android">
    <item android:color="?attr/colorPrimary" android:state_checked="true" />
    <item android:color="?attr/colorOnSurfaceVariant" android:state_checked="false" />
</selector>
```

### 9.4 Custom Illustration/Empty State Graphics (Opsional)

- Untuk ilustrasi empty state/onboarding yang kompleks (bukan icon UI sederhana), tetap
  pakai format vector (`VectorDrawable` hasil konversi dari SVG) untuk resolusi tajam di
  semua density, kecuali ilustrasi punya gradient/detail raster kompleks — baru pakai
  WebP (bukan PNG, untuk ukuran file lebih kecil) dengan varian density (`drawable-xxhdpi`
  dst) atau lebih baik lagi taruh di `drawable` sebagai adaptive vector jika memungkinkan.
- Ilustrasi custom sebaiknya ikut warna aksen aplikasi (cyan/teal) secara subtle, bukan
  ilustrasi generik stock yang tidak match dengan palette app.

---

## 10. Advanced Motion Choreography (Opsional, untuk polish tingkat lanjut)

### 10.1 Shared Element Transition

Untuk transisi dari list/card ke detail (mis. tap album art → full player), gunakan
`MaterialContainerTransform` sebagai shared element transition antar Fragment — ini yang
membuat transisi terasa "menyatu" alih-alih dua layar independen yang saling fade:

```kotlin
// Di Fragment tujuan
sharedElementEnterTransition = MaterialContainerTransform().apply {
    duration = 350L
    scrimColor = Color.TRANSPARENT
}
```
```xml
<!-- Kedua sisi transisi (source & target) beri transitionName yang sama -->
<ImageView
    android:id="@+id/albumArt"
    android:transitionName="album_art_transition" />
```

### 10.2 Staggered List Entrance (Opsional)

Untuk RecyclerView yang baru dimuat, animasi masuk item secara staggered (delay singkat
antar item, bukan semua muncul serentak) memberi kesan halus:

```kotlin
private fun runLayoutAnimation(recyclerView: RecyclerView) {
    val context = recyclerView.context
    val controller = AnimationUtils.loadLayoutAnimation(context, R.anim.layout_animation_fall_down)
    recyclerView.layoutAnimation = controller
    recyclerView.scheduleLayoutAnimation()
}
```
> Gunakan sekali saat initial load list, JANGAN dipicu ulang setiap `notifyDataSetChanged`
> atau scroll — ini bikin annoying, bukan elegan.

### 10.3 Spring-based Motion (Opsional, untuk interaksi custom seperti drag/swipe)

Untuk interaksi custom (bottom sheet drag, swipe-to-dismiss card), pertimbangkan
`DynamicAnimation.SpringForce` dari `androidx.dynamicanimation` alih-alih
`ObjectAnimator` linear biasa — hasilnya terasa lebih fisik/natural, konsisten dengan
motion language Material 3 yang "expressive":

```kotlin
SpringAnimation(view, DynamicAnimation.TRANSLATION_Y).apply {
    spring = SpringForce(0f).apply {
        stiffness = SpringForce.STIFFNESS_MEDIUM
        dampingRatio = SpringForce.DAMPING_RATIO_LOW_BOUNCY
    }
    start()
}
```

### 10.4 Predictive Back Gesture (Android 13+)

- Wajib support **Predictive Back** (selaras dengan `ANDROID_AGENT_RULES.md` Section 5) —
  pastikan setiap Fragment yang meng-override perilaku back button pakai
  `OnBackPressedCallback` terdaftar lewat `requireActivity().onBackPressedDispatcher`,
  bukan override `onBackPressed()` legacy.
- Untuk transisi custom saat predictive back (preview animasi saat user swipe-hold dari
  edge), pastikan `android:enableOnBackInvokedCallback="true"` ada di manifest dan
  transisi Fragment yang dipakai kompatibel dengan predictive back preview (Navigation
  Component versi terbaru sudah handle ini otomatis, jangan override manual kecuali perlu).

---

## 11. Tema Terang & Theming Lanjutan

- Tema selalu terang. Tidak ada dark mode, `values-night/`, atau toggle tema kecuali user
  memintanya eksplisit. Jika diminta: buat sebagai fitur terpisah dengan `values-night/`
  lengkap dan uji kontras terpisah.
- Paksa terang dengan `AppCompatDelegate.MODE_NIGHT_NO` dan `android:forceDarkAllowed=false`.
- Pastikan teks dan ikon kontras (WCAG AA) terhadap surface terang, terutama aksen cyan/teal
  yang mudah terlalu pucat di atas putih. Pakai varian `primary` yang cukup gelap untuk teks.
- Hierarki surface lewat `?attr/colorSurfaceContainerLowest` s/d `colorSurfaceContainerHighest`,
  bukan alpha hitam custom.
- **(Opsional)** Opsi "Dynamic Color On/Off" di Settings. Default off; jika off pakai seed
  color brand (cyan/teal). Jika on, tetap varian terang.
- Icon status bar & elemen sistem: rujuk Section 4 (selalu gelap di atas background terang).

---

## 12. Micro-interactions & Delight Details (Opsional)

Bagian ini murni untuk polish tingkat lanjut — tidak wajib untuk MVP, tapi sangat
direkomendasikan untuk fitur-fitur "hero" aplikasi (yang paling sering dipakai user):

- **Button press scale**: tombol utama (CTA besar) beri efek scale-down halus (0.96x)
  saat pressed via `StateListAnimator`, bukan hanya ripple polos — memberi kesan "fisik".
- **Pull-to-refresh custom**: kalau pakai `SwipeRefreshLayout`, sesuaikan warna indicator
  dengan accent app (cyan/teal), jangan biarkan warna default hijau Android generik.
- **Number/counter animation**: perubahan angka penting (skor, jumlah item, harga)
  animasikan dengan `ValueAnimator` count-up/down, bukan langsung snap ke angka baru.
- **Success/error micro-animation**: gunakan `Lottie` (jika sudah jadi dependency project)
  atau `AnimatedVectorDrawable` untuk checkmark/error icon setelah aksi selesai, alih-alih
  icon statis polos.
- **Skeleton shimmer**: arah shimmer konsisten (biasanya kiri ke kanan), durasi ~1.2-1.5s
  per cycle, warna shimmer sedikit lebih terang dari `colorSurfaceVariant` dasarnya.

> Prinsip pengendali: setiap micro-interaction harus punya **alasan fungsional** (memberi
> feedback, menarik perhatian ke perubahan penting) — bukan ditambahkan sekadar karena
> "terlihat keren". Kalau dihapus dan user tidak akan bingung/kehilangan informasi, berarti
> itu dekorasi berlebihan dan sebaiknya disederhanakan.

---

## 13. Checklist Tambahan — Advanced Polish (Opsional)

- [ ] Semua icon pakai `colorFillType` theme-aware (bukan hex statis), kecuali brand logo
- [ ] Icon dengan transisi state pakai `AnimatedVectorDrawable`, bukan swap drawable statis
- [ ] Transisi list→detail pakai `MaterialContainerTransform` untuk fitur-fitur utama
- [ ] Predictive back gesture sudah didaftarkan lewat `OnBackPressedDispatcher`
- [ ] Kontras teks/ikon pada surface terang dicek (WCAG AA), terutama warna aksen
- [ ] Micro-interaction yang ada punya fungsi jelas, tidak berlebihan/tanpa makna

---

## 14. Debug Tools: `debug.json` (Referensi Implementasi)

Spesifikasi ada di `ANDROID_AGENT_RULES.md` Section 7.6. Kelas yang sama (package dan API identik)
ada di dua source set; sengaja tidak ada di `src/main`.

**`app/src/debug/java/<package>/debug/DebugTools.kt`**
```kotlin
package com.example.app.debug

import android.app.Application
import android.os.Build
import android.util.Log
import org.json.JSONArray
import org.json.JSONObject
import java.io.File
import java.util.concurrent.Executors

object DebugTools {
    private const val MAX_EVENTS = 200
    private val io = Executors.newSingleThreadExecutor()
    private val events = ArrayDeque<JSONObject>()
    private var lastCrash: JSONObject? = null
    private lateinit var file: File
    private lateinit var info: JSONObject

    fun init(app: Application) {
        // getExternalFilesDir(null) = Android/data/<package>/files, tanpa permission storage.
        file = File(app.getExternalFilesDir(null) ?: app.filesDir, "debug.json")
        val pkg = app.packageManager.getPackageInfo(app.packageName, 0)
        info = JSONObject()
            .put("applicationId", app.packageName)
            .put("versionName", pkg.versionName)
            .put("versionCode", pkg.longVersionCode)
            .put("device", "${Build.MANUFACTURER} ${Build.MODEL}")
            .put("android", Build.VERSION.RELEASE)
            .put("sdk", Build.VERSION.SDK_INT)
        // Pertahankan crash dari sesi sebelumnya sampai ada crash baru.
        lastCrash = runCatching { JSONObject(file.readText()).optJSONObject("lastCrash") }.getOrNull()

        val previous = Thread.getDefaultUncaughtExceptionHandler()
        Thread.setDefaultUncaughtExceptionHandler { thread, error ->
            lastCrash = JSONObject()
                .put("time", System.currentTimeMillis())
                .put("thread", thread.name)
                .put("stackTrace", Log.getStackTraceString(error))
            writeNow() // sinkron: proses akan mati setelah ini
            previous?.uncaughtException(thread, error)
        }
        log("INIT", "session started")
    }

    fun log(tag: String, message: String) {
        synchronized(events) {
            events.addLast(
                JSONObject()
                    .put("time", System.currentTimeMillis())
                    .put("tag", tag)
                    .put("message", message)
            )
            while (events.size > MAX_EVENTS) events.removeFirst()
        }
        io.execute { writeNow() }
    }

    @Synchronized
    private fun writeNow() {
        val snapshot = synchronized(events) { JSONArray(events.toList()) }
        val root = JSONObject()
            .put("session", info)
            .put("events", snapshot)
            .put("lastCrash", lastCrash ?: JSONObject.NULL)
        runCatching { file.writeText(root.toString(2)) }
    }
}
```

**`app/src/release/java/<package>/debug/DebugTools.kt`** (no-op)
```kotlin
package com.example.app.debug

import android.app.Application

object DebugTools {
    fun init(app: Application) = Unit
    fun log(tag: String, message: String) = Unit
}
```

Pemakaian:
```kotlin
class App : Application() {
    override fun onCreate() {
        super.onCreate()
        AppCompatDelegate.setDefaultNightMode(AppCompatDelegate.MODE_NIGHT_NO)
        DebugTools.init(this)
    }
}

// Di titik penting (navigasi, aksi, error yang ditangani):
DebugTools.log("NAV", "open detail id=$id")
```
Ambil hasilnya: `adb pull /sdcard/Android/data/<package>.debug/files/debug.json` (build debug memakai
`applicationIdSuffix = ".debug"`, jadi foldernya berakhiran `.debug`).

---

## 15. Template Tema Terang Material 3 Expressive

Nilai warna di bawah adalah contoh seed cyan/teal; boleh diganti, struktur token tetap.

**`res/values/colors.xml`**
```xml
<resources>
    <color name="md_primary">#006874</color>
    <color name="md_on_primary">#FFFFFF</color>
    <color name="md_primary_container">#97F0FF</color>
    <color name="md_on_primary_container">#001F24</color>
    <color name="md_secondary">#4A6267</color>
    <color name="md_on_secondary">#FFFFFF</color>
    <color name="md_secondary_container">#CCE7EC</color>
    <color name="md_on_secondary_container">#051F23</color>
    <color name="md_surface">#F5FAFB</color>
    <color name="md_on_surface">#171D1E</color>
    <color name="md_surface_variant">#DBE4E6</color>
    <color name="md_on_surface_variant">#3F484A</color>
    <color name="md_outline">#6F797A</color>
    <color name="md_outline_variant">#BFC8CA</color>
    <color name="md_error">#BA1A1A</color>
    <color name="md_on_error">#FFFFFF</color>
    <color name="md_surface_container_lowest">#FFFFFF</color>
    <color name="md_surface_container_low">#EFF5F6</color>
    <color name="md_surface_container">#E9EFF0</color>
    <color name="md_surface_container_high">#E3E9EA</color>
    <color name="md_surface_container_highest">#DEE3E5</color>
</resources>
```

**`res/values/themes.xml`**
```xml
<resources>
    <style name="Theme.App" parent="Theme.Material3Expressive.Light.NoActionBar">
        <item name="colorPrimary">@color/md_primary</item>
        <item name="colorOnPrimary">@color/md_on_primary</item>
        <item name="colorPrimaryContainer">@color/md_primary_container</item>
        <item name="colorOnPrimaryContainer">@color/md_on_primary_container</item>
        <item name="colorSecondary">@color/md_secondary</item>
        <item name="colorOnSecondary">@color/md_on_secondary</item>
        <item name="colorSecondaryContainer">@color/md_secondary_container</item>
        <item name="colorOnSecondaryContainer">@color/md_on_secondary_container</item>
        <item name="colorSurface">@color/md_surface</item>
        <item name="colorOnSurface">@color/md_on_surface</item>
        <item name="colorSurfaceVariant">@color/md_surface_variant</item>
        <item name="colorOnSurfaceVariant">@color/md_on_surface_variant</item>
        <item name="colorOutline">@color/md_outline</item>
        <item name="colorOutlineVariant">@color/md_outline_variant</item>
        <item name="colorError">@color/md_error</item>
        <item name="colorOnError">@color/md_on_error</item>
        <item name="colorSurfaceContainerLowest">@color/md_surface_container_lowest</item>
        <item name="colorSurfaceContainerLow">@color/md_surface_container_low</item>
        <item name="colorSurfaceContainer">@color/md_surface_container</item>
        <item name="colorSurfaceContainerHigh">@color/md_surface_container_high</item>
        <item name="colorSurfaceContainerHighest">@color/md_surface_container_highest</item>

        <item name="shapeAppearanceMediumComponent">@style/ShapeAppearance.App.Medium</item>
        <item name="shapeAppearanceLargeComponent">@style/ShapeAppearance.App.Large</item>

        <item name="android:statusBarColor">@android:color/transparent</item>
        <item name="android:navigationBarColor">@android:color/transparent</item>
        <item name="android:windowLightStatusBar">true</item>
        <item name="android:windowLightNavigationBar">true</item>
        <item name="android:forceDarkAllowed">false</item>
        <item name="android:windowLayoutInDisplayCutoutMode">shortEdges</item>
    </style>

    <style name="ShapeAppearance.App.Medium" parent="ShapeAppearance.Material3.Corner.Medium">
        <item name="cornerSize">16dp</item>
    </style>
    <style name="ShapeAppearance.App.Large" parent="ShapeAppearance.Material3.Corner.Large">
        <item name="cornerSize">28dp</item>
    </style>
    <style name="ShapeAppearance.App.ExtraLarge" parent="ShapeAppearance.Material3.Corner.ExtraLarge">
        <item name="cornerSize">40dp</item>
    </style>
</resources>
```
Catatan: tidak ada `values-night/`. Komponen berstyle `Widget.Material3Expressive.*` dipasang per
komponen sesuai versi library terpasang (cek dokumentasi library, jangan menebak nama style).
