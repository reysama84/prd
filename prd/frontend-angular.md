# PRD — Sales Point (Frontend Angular)

> Product: **Sales Point** — aplikasi mobile-first untuk agen lapangan akuisisi Mini ATM (VisioNet).
> Platform: Angular 17+ (standalone components), PWA, target browser mobile (Chrome/Safari).
> Sumber: mockup HTML self-contained, 13 layar, bahasa Indonesia.

---

## 1. Tujuan & Lingkup Frontend

Membangun seluruh lapisan **frontend** (UI, state, service, routing) dari aplikasi Sales Point dengan Angular. Backend API diasumsikan tersedia (REST + JWT); dokumen ini hanya mencakup konsumsi dan presentasi.

### 1.1 Persona

| Kode | Persona | Konteks |
|------|---------|---------|
| AGENT | Sales Point Agent (lapangan) | Login, clock in/out GPS, daftar prospek toko dengan foto plang + selfie PIC, lihat history & notifikasi. |
| SUPV | Supervisor (out-of-scope UI) | Hanya muncul via data notifikasi/referral; tidak punya layar sendiri di rilis awal. |

### 1.2 Non-Goals (frontend)

- Tidak membangun dashboard admin / supervisor.
- Tidak membangun manajemen user/role (hanya read-only di profil).
- Tidak membangun report builder; chart pada profil bersifat read-only dari API.

---

## 2. Stack & Konvensi Teknis

| Area | Pilihan |
|------|---------|
| Framework | Angular 17 (standalone components, tanpa NgModules) |
| Bahasa | TypeScript strict mode |
| State | NgRx (store + effects + entity) |
| Forms | Reactive Forms |
| Styling | SCSS + CSS variables (port token dari mockup) |
| Icons | Inline SVG sprite (dari mockup) → bungkus dalam `IconComponent` |
| Charts | SVG custom (dari mockup) → komponen `MonthChartComponent` |
| HTTP | `HttpClient` + interceptor |
| PWA | `@angular/service-worker` (offline cache + splash) |
| Animasi | `@angular/animations` (screen-in, modal pop, toast) |
| Lint | ESLint + Prettier; aturan `@angular-eslint` |
| Test | Jest + `@testing-library/angular`; e2e Playwright |

### 2.1 Design Tokens (dari `:root` mockup)

Semua CSS variables pada mockup (`--navy-1`, `--accent`, `--orange`, `--good`, `--warn`, `--crit`, `--r-lg`, `--shadow`, dsb.) dipindahkan ke `src/styles/_tokens.scss` dan dipetakan ke Angular Material-like theme tanpa mengubah nama kelas visual.

```scss
// _tokens.scss (excerpt)
:root{
  --accent:#2a78d6;
  --accent-strong:#184f95;
  --orange:#f5821f;
  --good:#0ca30c; --warn:#fab219; --crit:#d03b3b;
  --hdr-1:#241566; --hdr-2:#2a1a72; --hdr-3:#1f1a6e;
  --surface:#ffffff; --surface-2:#f5f9fe;
  --gridline:#e1e8f2; --border:rgba(20,40,70,.09);
  --r-lg:18px; --r-md:12px; --r-sm:8px;
  --shadow:0 1px 2px rgba(20,40,70,.04),0 8px 24px -12px rgba(20,60,120,.14);
  --font:'Inter',system-ui,-apple-system,'Segoe UI',Roboto,Helvetica,Arial,sans-serif;
}
```

### 2.2 Konvensi Kode

- Prefix selector: `sp-` (sales point). Contoh: `<sp-app-bar>`, `<sp-nav-btn>`.
- Naming file: `kebab-case.ts`, class `PascalCaseComponent`.
- Setiap komponen standalone; `imports:` eksplisit.
- OnPush change detection untuk semua komponen presentasi.
- Tidak ada logika DOM manual (cek mockup JS vanilla); semua dipindahkan ke binding Angular.

---

## 3. Arsitektur Modul & Folder

```
src/
├─ app/
│  ├─ core/                       # singleton
│  │  ├─ services/                 # AuthService, AttendanceService, ...
│  │  ├─ guards/                   # AuthGuard, ClockInGuard, PendingPhotoGuard
│  │  ├─ interceptors/             # AuthInterceptor, ErrorInterceptor, LoadingInterceptor
│  │  ├─ models/                   # interfaces (User, Attendance, Prospek, ...)
│  │  └─ core.config.ts
│  ├─ state/                       # NgRx
│  │  ├─ auth/  attendance/  prospek/  notif/  profile/  ui/
│  │  └─ index.ts                 # meta-reducers, store config
│  ├─ shared/                     # dumb components, pipes, directives
│  │  ├─ components/  pipes/  directives/
│  │  └─ shared.config.ts         # export array SHARED_COMPONENTS
│  ├─ features/
│  │  ├─ auth/                    # splash, login, loading
│  │  ├─ attendance/              # gate, detail
│  │  ├─ home/
│  │  ├─ prospek/                 # check-in, history, detail
│  │  ├─ camera/
│  │  ├─ photo-viewer/
│  │  ├─ notifications/
│  │  └─ profile/
│  ├─ layout/                     # ShellComponent (device frame + bottom nav)
│  │  └─ shell.component.ts
│  ├─ app.routes.ts
│  └─ app.component.ts
├─ assets/  (logo webp, brand)
└─ styles/ (_tokens.scss, _reset.scss, _animations.scss, styles.scss)
```

### 3.1 Module Dependency Graph

```
core ───► state ───► features ───► layout ───► app
                       │
                       └──► shared
```

Lazy-load semua `features/*` kecuali `auth` (preload critical).

---

## 4. Routing

### 4.1 Route Tree

```ts
// app.routes.ts
export const APP_ROUTES: Routes = [
  { path: '',           component: SplashComponent,          title: 'Sales Point' },
  { path: 'auth/login', loadComponent: () => import('./features/auth/login/login.component').then(m => m.LoginComponent) },
  { path: 'loading',    loadComponent: () => import('./features/auth/loading/loading.component').then(m => m.LoadingComponent) },

  // pre-authenticated gate
  { path: 'clock-in',
    canActivate: [AuthGuard],
    loadComponent: () => import('./features/attendance/gate/clock-in-gate.component').then(m => m.ClockInGateComponent) },

  // authenticated shell with bottom nav
  {
    path: 'app',
    canActivate: [AuthGuard, ClockInGuard],
    component: ShellComponent,
    children: [
      { path: 'home',          loadComponent: () => import('./features/home/home.component').then(m => m.HomeComponent) },
      { path: 'attendance',     loadComponent: () => import('./features/attendance/detail/attendance-detail.component').then(m => m.AttendanceDetailComponent) },
      { path: 'prospek/new',    loadComponent: () => import('./features/prospek/checkin/prospek-checkin.component').then(m => m.ProspekCheckinComponent) },
      { path: 'prospek/history',loadComponent: () => import('./features/prospek/history/prospek-history.component').then(m => m.ProspekHistoryComponent) },
      { path: 'prospek/:id',    loadComponent: () => import('./features/prospek/detail/prospek-detail.component').then(m => m.ProspekDetailComponent) },
      { path: 'notifications',  loadComponent: () => import('./features/notifications/notifications.component').then(m => m.NotificationsComponent) },
      { path: 'profile',        loadComponent: () => import('./features/profile/profile.component').then(m => m.ProfileComponent) },
      { path: '', redirectTo: 'home', pathMatch: 'full' },
    ],
  },

  // overlays (outside shell — full-screen)
  { path: 'camera/:shot',     canActivate: [AuthGuard], loadComponent: () => import('./features/camera/camera.component').then(m => m.CameraComponent) },
  { path: 'photo/:id/:kind',  canActivate: [AuthGuard], loadComponent: () => import('./features/photo-viewer/photo-viewer.component').then(m => m.PhotoViewerComponent) },

  { path: '**', redirectTo: '' },
];
```

### 4.2 Route Guards

| Guard | Tujuan |
|-------|--------|
| `AuthGuard` | Cek `auth.user$`; redirect ke `/auth/login` bila null. |
| `ClockInGuard` | Cek `attendance.today$.clockIn`; bila belum → redirect ke `/clock-in` (kecuali route `/app/attendance` agar bisa lihat history). |
| `PendingPhotoGuard` | Bila `prospek.draft.photos.{plang\|selfie}` null saat route ke prospek detail dari check-in → blok & toast. |

### 4.3 Strategi Preload

`PreloadAllModules` kecuali `camera/*` dan `photo/*` (berat, jarang). Gunakan custom `SelectivePreloadStrategy`.

---

## 5. State Management (NgRx)

### 5.1 Slices

| Slice | State utama | Actions |
|-------|-------------|---------|
| `auth` | `user`, `token`, `loading`, `error` | `Login`, `LoginSuccess`, `LoginFail`, `Logout`, `LogoutConfirm`, `SessionExpire` |
| `attendance` | `today`, `history`, `loading`, `error` | `ClockIn`, `ClockInSuccess`, `ClockOut`, `ClockOutSuccess`, `LoadHistory`, `LoadHistoryOlder` |
| `prospek` | `today: Prospek[]`, `history: DayGroup[]`, `draft: { photos, form }`, `selected: Prospek`, `loading` | `SaveProspek`, `SaveProspekSuccess`, `LoadHistory`, `LoadHistoryOlder`, `SearchProspek`, `SelectProspek`, `ResetDraft`, `SetDraftPhoto` |
| `notif` | `groups: NotifGroup[]`, `unread: number` | `LoadNotif`, `MarkRead`, `MarkAllRead`, `UnreadUpdated` |
| `profile` | `profile`, `chartData`, `loading` | `LoadProfile`, `LoadChart`, `CopyReferral` |
| `ui` | `activeScreen`, `toast`, `modal`, `camFacing`, `camTorch` | `Navigate`, `ShowToast`, `OpenModal`, `CloseModal`, `SetCamFacing`, `SetTorch` |

### 5.2 Effects (high-level)

- `AuthEffects.login$` → POST `/auth/login`; on success → `LoginSuccess` + `Navigate('/loading')`.
- `AuthEffects.loadingSequence$` → simulasikan step load (token: 18/42/63/84/100%) → `Navigate('/clock-in' | '/app/home')`.
- `AttendanceEffects.clockIn$` → POST `/attendance/clock-in` (body: `{lat,lng,accuracy,selfieBase64}`).
- `AttendanceEffects.clockOut$` → POST `/attendance/clock-out`.
- `ProspekEffects.save$` → POST `/prospek` (multipart: photos + JSON form).
- `ProspekEffects.search$` → debounce 250ms, filter lokal (data sudah di-cache).
- `NotifEffects.markAllRead$` → POST `/notifications/read-all`.
- `ProfileEffects.copyReferral$` → side-effect clipboard via `ClipboardService`.

### 5.3 Selectors utama

```ts
selectUser              // auth.user
selectIsAuthenticated   // !!auth.token
selectTodayAttendance   // attendance.today
selectHasClockedIn      // !!attendance.today?.clockIn
selectTodayProspekCount // prospek.today.length
selectProspekDraft      // prospek.draft
selectUnreadNotif       // notif.unread
selectToast             // ui.toast
```

### 5.4 Entity adapter

`prospek` pakai `@ngrx/entity` untuk `Prospek` (id = UUID). `notif` pakai struktur group biasa (urutan tetap, mark-read per item).

---

## 6. Model Data (interfaces)

```ts
// core/models/user.model.ts
export interface User {
  id: string;
  username: string;
  fullName: string;
  email: string;
  referralCode: string;     // 'SP-RZK2041'
  role: 'AGENT';
  active: boolean;
}

// core/models/attendance.model.ts
export interface AttendanceToday {
  date: string;             // ISO
  shift: { name: string; start: string; end: string };  // '08:00','17:00'
  clockIn?: string;         // 'HH:mm'
  clockOut?: string;
  location: { lat: number; lng: number; label: string; accuracyM: number };
  method: 'GPS+SELFIE';
}
export interface AttendanceHistoryItem {
  dd: string; mm: string; day: string;
  in: string; out: string; dur: string;
  st: string; cls: 'good'|'warn'|'info'|'crit';
}

// core/models/prospek.model.ts
export interface Prospek {
  id: string;
  name: string;
  address: string;
  picName: string;
  picPhone: string;
  visitTime: string;        // 'HH:mm'
  visitDate: string;        // '4 September 2026'
  note?: string;
  photoPlang?: string;       // dataURL or asset URL
  photoSelfie?: string;
  status?: 'verified'|'pending'|'rejected';
  coords?: { lat: number; lng: number };
}

// core/models/notif.model.ts
export interface NotifItem {
  id: string;
  icon: 'blue'|'green'|'orange'|'red';
  title: string;
  body: string;
  time: string;
  unread: boolean;
  svgKey: string;           // key into ICON_REGISTRY
}
export interface NotifGroup { day: string; items: NotifItem[]; }

// core/models/chart.model.ts
export interface ChartPoint { m: string; v: number; current?: boolean; }
```

---

## 7. Services (core)

| Service | Tanggung jawab | Method utama |
|---------|----------------|--------------|
| `AuthService` | login/logout, simpan token (localStorage), expose `user$` | `login(u,p)`, `logout()`, `restoreSession()` |
| `AttendanceService` | clock in/out, history | `clockIn(payload)`, `clockOut()`, `getToday()`, `getHistory()`, `getOlder()` |
| `ProspekService` | CRUD prospek | `save(form,photos)`, `listToday()`, `listHistory(cursor)`, `search(q)`, `getById(id)` |
| `NotificationService` | list & mark-read | `list()`, `markRead(id)`, `markAllRead()` |
| `ProfileService` | profile + chart | `getProfile()`, `getMonthlyChart()` |
| `GeolocationService` | wrap `navigator.geolocation` → RxJS | `watchAccuracy$()`, `getCurrent()` |
| `CameraService` | wrap `getUserMedia`, capture, torch, flip | `open(kind)`, `capture()`, `toggleTorch()`, `flip()`, `stop()` |
| `ClipboardService` | copy text dengan fallback `execCommand` | `copy(text)` |
| `DownloadService` | unduh foto (dataURL → blob) | `save(filename, blob)` |
| `ToastService` | antrian toast (NgRx `ui.toast`) | `show(msg)`, `dismiss()` |
| `ModalService` | open/close overlay (NgRx `ui.modal`) | `open(id)`, `close(id)` |
| `LoadingService` | simulasikan/observ loading step | `run(steps: Step[])` |
| `ErrorHandlerService` | global error → toast + log | — |

### 7.1 HTTP Interceptors

1. **AuthInterceptor** — sisipkan `Authorization: Bearer <token>` ke semua request ke `/api/*`.
2. **ErrorInterceptor** — tangani 401 → dispatch `SessionExpire` + navigate `/auth/login`; ≥500 → toast generik.
3. **LoadingInterceptor** (opsional) — increment/decrement global loading counter untuk spinner.

### 7.2 Environment

```ts
export const environment = {
  production: false,
  apiBase: '/api',
  demo: true,                       // kredensial demo rizky.pratama/salespoint
  mapTile: 'https://.../...',       // tidak dipakai langsung; lokasi via label
  attendance: { shiftStart: '08:00', shiftEnd: '17:00', cutoffMin: 17*60 },
};
```

---

## 8. Shared Components

Semua standalone, OnPush, dengan `inputs()` / `outputs()` Angular 17.

### 8.1 Daftar Komponen

| Selector | Input / Output | Deskripsi |
|----------|----------------|-----------|
| `<sp-app-bar>` | `title`, `subtitle?`, `showBack?` | Header gradient + back button + vec bg. |
| `<sp-icon-btn>` | `icon: string`, `ariaLabel?`, `variant?` | Tombol icon bulat (lihat `IconComponent`). |
| `<sp-nav-btn>` | `icon`, `label`, `active?`, `badge?` | Item bottom nav. |
| `<sp-bottom-nav>` | `active: 'home'\|'notif'\|'profile'`, `unread$` | Container nav 3 kolom. |
| `<sp-card>` | `class?`, projected content | Card surface with shadow. |
| `<sp-badge>` | `variant: 'good'\|'warn'\|'crit'\|'info'`, `pip?` | Pill badge. |
| `<sp-btn>` | `variant: 'primary'\|'ghost'\|'danger'\|'red'\|'green'`, `icon?`, `loading?`, `disabled?` | Tombol utama. |
| `<sp-input>` | Reactive `ControlValueAccessor`, `icon?`, `type`, `placeholder`, `error?` | Wrap input + icon + hint-error. |
| `<sp-textarea>` | CVA, `rows`, `placeholder`, `error?` | Textarea variant. |
| `<sp-field>` | `label`, `required?`, `invalid?`, `hint?` | Wrapper label + control + hint-error. |
| `<sp-search-bar>` | CVA, `placeholder` | Search input with clear button. |
| `<sp-photo-slot>` | `kind: 'plang'\|'selfie'`, `filled?`, `photoUrl?`, `time?` | Slot foto 3:4 dengan tombol retake. |
| `<sp-stat-cell>` | `value`, `label`, `color?` | Stat cell dalam grid 3-kolom. |
| `<sp-meter>` | `value: number`, `max: number` | Progress meter horizontal. |
| `<sp-avatar>` | `initials`, `size?: 'sm'\|'lg'` | Avatar gradient orange. |
| `<sp-daygroup>` | `label`, `count?` | Group label dengan garis & chip count. |
| `<sp-empty>` | `icon`, `title`, `desc?` | Empty state (search no result). |
| `<sp-load-more>` | `loading?`, `disabled?` | Tombol dashed "Muat sebelumnya". |
| `<sp-spinner-dots>` | — | 3 dots animation. |
| `<sp-progress-bar>` | `value: 0..100` | Progress bar gradient. |
| `<sp-modal>` | `open`, `variant: 'success'\|'warn'`, `icon`, `title`, `body`, `recap?` | Modal overlay dengan animasi pop. |
| `<sp-toast>` | `open`, `message`, `icon?` | Toast bottom. |
| `<sp-month-chart>` | `points: ChartPoint[]`, `total: number` | SVG bar chart interaktif + tooltip. |
| `<sp-icon>` | `name: string` | SVG sprite registry (lihat §8.2). |
| `<sp-vec-bg>` | `variant: 'dotwave'\|'ringwave'` | Background SVG dekoratif (splash/login/loading). |

### 8.2 Icon Registry

Mockup menyertakan banyak inline SVG. Buat `ICON_REGISTRY: Record<string, string>` di `shared/components/icon/icons.ts`:

```ts
export const ICON_REGISTRY = {
  back: '<path d="M15 6l-6 6 6 6"/>',
  user: '<path d="M20 21v-2a4 4 0 0 0-4-4H8a4 4 0 0 0-4 4v2"/><circle cx="12" cy="7" r="4"/>',
  bell: '<path d="M18 8.5a6 6 0 1 0-12 0c0 6-2.2 7.5-2.2 7.5h16.4S18 14.5 18 8.5Z"/><path d="M13.7 19.5a2 2 0 0 1-3.4 0"/>',
  home: '<path d="m3 10.5 9-7 9 7V20a1.5 1.5 0 0 1-1.5 1.5h-15A1.5 1.5 0 0 1 3 20Z"/><path d="M9.5 21.5v-7h5v7"/>',
  // ... semua ikon dari mockup
};
```

`IconComponent` render `<svg viewBox="0 0 24 24" [innerHTML]="path">` (sanitize via `DomSanitizer`).

### 8.3 Pipes & Directives

| Nama | Tipe | Fungsi |
|------|------|--------|
| `InitialsPipe` | pure | `'Rizky Pratama' → 'RP'` |
| `SlugPipe` | pure | `'Toko Berkah Jaya' → 'Toko_Berkah_Jaya'` (untuk filename unduh) |
| `FormatTimePipe` | pure | ISO → `'HH:mm'` |
| `SafeHtmlDirective` | attr | bypass sanitizer untuk SVG string |

---

## 9. Komponen per Layar (Detail)

### 9.1 Splash (`features/auth/splash`)

**Selector:** `sp-splash`

**UI:**
- Background SVG `<sp-vec-bg variant="ringwave">`.
- Brand logo (webp asset, 210px).
- Divider orange (52×3).
- App name "SALES POINT" uppercase, letter-spacing 0.24em.
- Tagline "Aplikasi akuisisi agen Mini ATM di lapangan".
- `<sp-spinner-dots>` (bounce animation).
- Footer: "VisioNet Mini ATM · v1.0.0".

**Behavior:**
- On init: dispatch `AuthEffects.restoreSession$`.
- Auto-advance ke `/auth/login` setelah 2.4s (sama dengan mockup `setTimeout`).
- Jika session valid → `/loading` → `/app/home`.
- Animasi `rise` (opacity + translateY + scale).

**Inputs/Outputs:** — (route-level component)

**A11y:** `role="status"`, `aria-live="polite"`.

---

### 9.2 Login (`features/auth/login`)

**Selector:** `sp-login`

**Form (Reactive):**

```ts
form = this.fb.group({
  username: ['', [Validators.required]],
  password: ['', [Validators.required]],
  remember: [true],
});
```

**UI Structure:**
- `.login-head.has-vec` → logo (132px) + judul "Masuk ke Sales Point" + subjudul.
- `.sheet` (white rounded-top) berisi:
  - Alert error (`*ngIf="loginError"`) — `<sp-icon name="alert">` + pesan.
  - `<sp-field label="Username" required>` + `<sp-input icon="user" ...>`.
  - `<sp-field label="Kata Sandi" required>` + `<sp-input icon="lock" type="password" ...>` + toggle password button.
  - `.checkline`: checkbox "Ingat saya" + link "Lupa kata sandi?".
  - `<sp-btn variant="primary" icon="arrow-right" [loading]="loading" [disabled]="form.invalid">Masuk</sp-btn>`.
  - `.demo-note`: petunjuk kredensial demo.

**Behavior:**
- Submit → dispatch `Login({username, password})`.
- Pada `AuthEffects` → POST `/auth/login`. Mock demo: validasi `rizky.pratama / salespoint` lokal.
- Error → set `loginError` dan tampilkan alert merah.
- Success → `Navigate('/loading')`.
- Toggle password: ganti `type` antara `password` / `text`, ubah ikon mata.
- Link "Lupa kata sandi?" → `ToastService.show('Hubungi supervisor area untuk reset kata sandi.')`.

**Validations:**
- `username`: required, pesan "Username wajib diisi."
- `password`: required, pesan "Kata sandi wajib diisi."
- Tidak ada validasi panjang minimum (mockup tidak mensyaratkan).

**A11y:**
- Label terhubung via `for`/`id`.
- Tombol toggle password punya `aria-label="Tampilkan/Sembunyikan kata sandi"`.
- Alert `role="alert"`.

---

### 9.3 Loading (`features/auth/loading`)

**Selector:** `sp-loading`

**UI:**
- Background ringwave.
- Logo sm.
- `<sp-progress-bar [value]="pct">`.
- Persentase besar (`--fs-3xl`).
- Status text (changes per step).

**Behavior:**
- On init: subscribe `LoadingService.run(steps)`.
- Steps (dari mockup JS):
  ```
  18%  "Memverifikasi kredensial agen…"
  42%  "Memuat profil & area kerja…"
  63%  "Sinkronisasi data absensi…"
  84%  "Menarik data prospek terbaru…"
  100% "Siap digunakan"
  ```
- Tiap step delay 430–690ms (random).
- Setelah 100% → wait 420ms → `Navigate('/clock-in')` (atau `/app/home` bila sudah clock-in).

---

### 9.4 Clock-In Gate (`features/attendance/gate`)

**Selector:** `sp-clock-in-gate`

**UI:**
- `.gate-top.has-vec` (gradient navy): logo xs, greeting "Halo, {firstName}", tanggal formatted (`Jumat, 4 September 2026`).
- `.clock-card` (overlap −30px): badge "Belum absen hari ini", live clock `HH:mm:ss` (font 44px), timezone "WIB · Waktu Indonesia Barat", location chip dengan akurasi ±8m.
- Card shift info: "Shift hari ini · Reguler · 08:00 – 17:00 WIB".
- `.gate-actions`: `<sp-btn variant="primary" icon="clock">Clock In Sekarang</sp-btn>` + note kecil.

**Behavior:**
- Subscribe `GeolocationService.watchAccuracy$()` → update label akurasi.
- Clock tick via `interval(1000)` (stop saat `OnDestroy`).
- Tombol Clock In → dispatch `ClockIn({lat,lng,accuracy,selfie?})` (mockup tidak ambil selfie di gate; real impl: redirect ke `/camera/selfie?purpose=clockin`).
- On success → open modal `<sp-modal variant="success" title="Clock In Berhasil">` dengan recap waktu & lokasi → tombol "Lanjut ke Beranda" → `Navigate('/app/home')`.

**State binding:**
- `attendance.today$` → greeting, badge, tombol.
- `ui.clock` (real-time) → tampilan jam.

---

### 9.5 Home / Dashboard (`features/home`)

**Selector:** `sp-home`

**UI Sections:**

1. **Header (`.home-head.has-vec`):**
   - Toprow: logo xs (center) + avatar button (right) → navigate `/app/profile`.
   - Welcome: "Welcome back," + nama user.
   - Referral chip (button): icon tag + "Kode Referral" + kode + icon copy. Click → `ClipboardService.copy(referralCode)` + toast.

2. **Body (`.home-body`, margin-top −32px):**
   - **Stat-solo card:** angka besar prospek hari ini + tanggal.
   - **Menu grid (2-kolom):**
     - "Prospek Toko" → `/app/prospek/new`. Icon: ilustrasi webp.
     - "History Prospek" → `/app/prospek/history`. Icon: ilustrasi webp.
   - **Attendance card:**
     - Header (button ke `/app/attendance`): icon OK/warn + "Sudah clock in · {time} WIB" + subjudul + chevron.
     - Footer: tombol contextual:
       - Belum clock-in → `btn-green` "Clock In Sekarang".
       - Sudah in, belum out → `btn-red` "Clock Out Sekarang".
       - Sudah out → footer hidden.

3. **Bottom nav** (`<sp-bottom-nav active="home" [unread]="unread$ | async">`).

**Behavior:**
- `selectTodayProspekCount` → angka stat.
- `selectTodayAttendance` → status tombol.
- Avatar button → route.
- Referral chip → `ProfileEffects.copyReferral$`.

**State binding:** `auth.user$`, `prospek.today$`, `attendance.today$`, `notif.unread$`.

---

### 9.6 Attendance Detail (`features/attendance/detail`)

**Selector:** `sp-attendance-detail`

**UI:**
- `<sp-app-bar title="Clock In" subtitle="Absensi kehadiran harian" showBack>` dengan back → `/app/home`.
- `.today-attend.card`:
  - Top: tanggal + shift info.
  - `tt-grid` 2-kolom: Clock In time + Clock Out time (atau "Belum absen" warna abu).
  - `tt-meta`: lokasi, durasi ("5 jam 34 menit berjalan"), metode "GPS + Selfie · akurasi ±8 m".
- Tombol `<sp-btn variant="red" icon="logout">Clock Out</sp-btn>` (disabled bila sudah out).
- Section "Riwayat Absen": list `.attend-row` (date + duration + times arrow).
- `<sp-load-more>` → load `absenOlder`.

**Behavior:**
- Durasi live update via `interval(60000)` bila `clockIn && !clockOut`.
- Clock-out → dispatch `ClockOut()` → toast "Clock out tercatat pukul HH:mm WIB."
- Load older → dispatch `LoadHistoryOlder()`.

---

### 9.7 Prospek Check-In Form (`features/prospek/checkin`)

**Selector:** `sp-prospek-checkin`

**Route:** `/app/prospek/new`

**UI:**
- `<sp-app-bar title="Prospek Toko" subtitle="Kunjungan ke-{n+1} hari ini · {time} WIB" showBack>`.
- Location chip: "Lokasi terdeteksi" + coords + alamat reverse + akurasi ±6m.
- Form:
  - Nama Toko (required).
  - Alamat (textarea, required).
  - Nama PIC (required).
  - No. Telp PIC (tel, required, min 9 digit numeric).
  - Photo grid 2-kolom:
    - `<sp-photo-slot kind="plang" (click)="openCamera('plang')">`.
    - `<sp-photo-slot kind="selfie" (click)="openCamera('selfie')">`.
    - Bila sudah ada photo → filled state + tombol retake + tag timestamp.
  - Catatan (textarea, optional).
- Sticky footer `.form-foot`: `<sp-btn variant="primary" icon="save">Simpan Prospek</sp-btn>`.

**Reactive Form:**

```ts
form = this.fb.group({
  name:    ['', Validators.required],
  address: ['', Validators.required],
  picName: ['', Validators.required],
  picPhone:['', [Validators.required, Validators.pattern(/^[0-9]{9,15}$/)]],
  note:    [''],
});
photos = this.fb.group({
  plang:  [null as string | null, Validators.required],
  selfie: [null as string | null, Validators.required],
});
```

**Behavior:**
- Open camera → `Navigate('/camera/plang')` (atau `'selfie'`). Camera return → `SetDraftPhoto({kind, dataUrl})` → update photo slot.
- Save:
  - Validasi semua field + 2 foto.
  - Bila invalid → mark `.invalid`, scroll ke field pertama, toast "Lengkapi data bertanda * sebelum menyimpan."
  - Bila valid → dispatch `SaveProspek(form.value, photos.value)`.
  - On success → open modal success:
    - Title "Prospek Berhasil Disimpan".
    - Recap: Nama toko, PIC, No. telp, Waktu, Dokumentasi "2 foto terlampir".
    - Tombol "Selesai" → reset form + `Navigate('/app/home')`.
    - Tombol "Check In Lagi" → reset form + scroll top.

**Guards:** `PendingPhotoGuard` tidak relevan di sini (form sendiri), tapi guard dipasang pada route detail.

---

### 9.8 Prospek History (`features/prospek/history`)

**Selector:** `sp-prospek-history`

**UI:**
- `<sp-app-bar title="History Prospek" subtitle="{count} prospek hari ini" showBack>`.
- `<sp-search-bar placeholder="Cari nama toko, alamat, atau PIC">`.
- Results: grup per hari (`<sp-daygroup>` + list `.pitem`).
- Tiap item: thumbnail (initials atau foto), nama toko, alamat, meta (time + pic), chevron.
- `<sp-load-more>` → dispatch `LoadHistoryOlder()`.
- Empty state bila search no-result.

**Behavior:**
- Search debounce 250ms via `ProspekEffects.search$` → filter lokal di store (data cached).
- Clear button → reset query, focus search.
- Click item → `Navigate('/app/prospek/{id}')`.
- Load older: tambah group dari `prospekOlder` array. Bila habis → disable button + ubah label.

---

### 9.9 Prospek Detail (`features/prospek/detail`)

**Selector:** `sp-prospek-detail`

**Route:** `/app/prospek/:id`

**UI:**
- `<sp-app-bar title="Detail Prospek" subtitle="{storeName}" showBack>` → back ke `/app/prospek/history`.
- Section "Dokumentasi Foto": 2 `.photo-card` 3:4 (plang + selfie). Click → `Navigate('/photo