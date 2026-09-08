# PRD: Sales Point (Angular Frontend)

## 1. Overview

**Product:** Sales Point — a mobile-first Angular SPA for field sales agents who acquire and manage Mini ATM merchant prospects in the field.

**Scope:** This PRD covers the **Angular frontend only**: application shell, routing, state management, services, components, forms, camera/geo integration, and UI/UX behavior. Backend API contracts are referenced as integration points but are out of scope for implementation.

**Target Device:** Mobile-first PWA, optimized for 412 px width (max), 100 dvh height, with a simulated device shell on desktop (≥ 560 px viewport).

---

## 2. Tech Stack & Architecture

| Concern | Choice |
|---|---|
| Framework | Angular (standalone components, Signals for state) |
| Routing | Angular Router with lazy-loaded route groups |
| State | Signal-based stores (no NgRx); `AttendanceStore`, `ProspectStore`, `NotificationStore`, `SessionStore` |
| Forms | Reactive Forms (template-driven only for login) |
| Styling | Global SCSS with CSS custom properties (design tokens from mockup) |
| Charts | Custom SVG component (no chart library) |
| Camera | `getUserMedia` + `canvas` capture service |
| Icons | Inline SVG sprite / component |
| PWA | service worker, manifest, offline cache of shell |

### 2.1 Design Tokens (extracted from mockup)

All colors, radii, shadows, and font sizes are defined as CSS custom properties in `:root` and consumed globally. Key tokens:

```
--hdr-1:#241566  --accent:#2a78d6  --orange:#f5821f
--good:#0ca30c   --warn:#fab219    --crit:#d03b3b
--r-lg:18px --r-md:12px --r-sm:8px
--font:'Inter',system-ui,...
--fs-2xs:11px ... --fs-3xl:28px
--shadow / --shadow-up
```

---

## 3. Application Shell

### 3.1 `AppComponent` (root)

- Hosts the device frame container (`.device`) which includes an optional status bar (desktop only) and the router outlet for screens.
- Manages `data-chrome` attribute on the device: `"navy"` for normal screens, `"black"` for camera and photo-viewer screens (hides status bar text contrast).
- Provides a global `ToastService` and `ModalService` via dependency injection.
- Listens to router navigation events to:
  - Reset scroll position of `.scroll` containers to top on navigation.
  - Sync bottom-nav `active` state with the current route.
- Handles `prefers-reduced-motion` globally (disables animations).

### 3.2 Device Frame

A presentational `DeviceFrameComponent` that:
- Renders the status bar (time, signal icons) — visible only at ≥ 560 px viewport.
- Renders a hidden SVG `<defs>` block (`#dotwave`, `#ringwave`) used as vector backgrounds by header sections.
- Projects `<router-outlet>` inside `.screens`.

### 3.3 Bottom Navigation (`BottomNavComponent`)

- A 3-tab bar: **Home**, **Notifikasi**, **Profil**.
- Each tab is a router link with `routerLinkActive`.
- Notifikasi tab shows a red badge with unread count (Signal from `NotificationStore`).
- Active tab indicator: a top accent bar (`::before`) and accent-colored icon/label.
- Visible only on Home, Notifications, and Profile screens (hidden on camera, form, detail, etc.).

---

## 4. Routing

```ts
const routes: Routes = [
  { path: 'splash',   component: SplashComponent },
  { path: 'login',    component: LoginComponent },
  { path: 'loading',  component: LoadingComponent },
  { path: 'gate',     component: ClockInGateComponent, canActivate: [AuthGuard] },
  {
    path: 'app',
    canActivate: [AuthGuard],
    children: [
      { path: 'home',          component: HomeComponent },
      { path: 'attendance',    component: AttendanceDetailComponent },
      { path: 'history',       component: HistoryComponent },
      { path: 'checkin',       component: CheckInFormComponent },
      { path: 'camera/:kind',  component: CameraComponent },
      { path: 'detail/:id',    component: ProspectDetailComponent },
      { path: 'viewer/:id/:kind', component: PhotoViewerComponent },
      { path: 'notifications', component: NotificationsComponent },
      { path: 'profile',       component: ProfileComponent },
    ]
  },
  { path: '',   redirectTo: 'splash', pathMatch: 'full' },
  { path: '**', redirectTo: 'splash' },
];
```

### 4.1 Navigation Guards

- **`AuthGuard`**: Checks `SessionStore.isAuthenticated()`. If false → redirect to `/login`.
- **`ClockInGuard`** (optional): If user hasn't clocked in today and route is not `gate` or `home` → redirect to `/gate`. The gate screen is shown after login until clock-in is performed.

### 4.2 Screen Transitions

- Each screen is an Angular component with a host class `.screen`.
- On activation, `.active` class is applied → triggers a `screen-in` animation (`translateY(8px) → 0`, fade in, 0.26s).
- Navigation between screens uses the Angular Router; the device shell does not unmount.

---

## 5. State Management (Signal Stores)

All stores are injectable services using Angular Signals. No external state library.

### 5.1 `SessionStore`

| Signal | Type | Description |
|---|---|---|
| `user` | `User \| null` | Logged-in agent profile |
| `isAuthenticated` | `boolean` | Derived from `user` |
| `rememberMe` | `boolean` | Persist session flag |

**Methods:** `login(username, password)`, `logout()`.

### 5.2 `AttendanceStore`

| Signal | Type | Description |
|---|---|---|
| `todayRecord` | `AttendanceRecord \| null` | `{ in: "07:48", out: null }` |
| `history` | `AttendanceRecord[]` | Past attendance |
| `olderHistory` | `AttendanceRecord[]` | Archived (previous month) |
| `status` | `'none' \| 'clocked_in' \| 'clocked_out'` | Derived |

**Methods:** `clockIn()`, `clockOut()`, `loadOlder()`.

### 5.3 `ProspectStore`

| Signal | Type | Description |
|---|---|---|
| `today` | `Prospect[]` | Today's prospects |
| `olderGroups` | `DayGroup[]` | Grouped by day, loaded on demand |
| `olderLoaded` | `number` | Count of older groups loaded |
| `searchQuery` | `string` | Filter text |
| `filteredResults` | `Prospect[]` | Derived from query across all loaded |
| `todayCount` | `number` | `today().length` |
| `nextVisitNumber` | `number` | `todayCount + 1` |

**Methods:** `add(prospect)`, `loadOlder()`, `search(q)`, `getById(id)`.

### 5.4 `NotificationStore`

| Signal | Type | Description |
|---|---|---|
| `groups` | `NotifGroup[]` | Grouped by day |
| `unreadCount` | `number` | Derived |
| `hasUnread` | `boolean` | Derived |

**Methods:** `markAsRead(item)`, `markAllRead()`.

---

## 6. Services

### 6.1 `AuthService`

- `login(username: string, password: string): Observable<LoginResult>`
- Validates against API; on success, stores token in `localStorage` (or sessionStorage if not "remember me").
- Demo mode accepts `rizky.pratama` / `salespoint`.
- `logout()`: clears session, stops camera stream, resets stores.

### 6.2 `AttendanceApiService`

- `submitClockIn(): Observable<AttendanceRecord>`
- `submitClockOut(): Observable<AttendanceRecord>`
- `getHistory(month?: string): Observable<AttendanceRecord[]>`

### 6.3 `ProspectApiService`

- `saveProspect(payload: ProspectPayload): Observable<Prospect>`
- `getToday(): Observable<Prospect[]>`
- `getOlder(page: number): Observable<DayGroup[]>`
- `getById(id: string): Observable<Prospect>`

### 6.4 `CameraService`

Encapsulates all `getUserMedia` logic:

| Method | Description |
|---|---|
| `open(facing: 'user' \| 'environment')` | Requests stream, attaches to `<video>` element |
| `capture(videoEl, facing): string` | Draws frame to canvas, mirrors if front cam, stamps geo+time, returns data URL |
| `toggleTorch(enabled: boolean)` | Applies torch constraint if supported |
| `flip()` | Toggles facing mode, restarts stream |
| `stop()` | Stops all tracks, clears srcObject |
| `hasCameraSupport` | Getter: checks `navigator.mediaDevices` |

**Error handling:** If camera permission denied or not found → emits error signal; UI shows fallback "Gunakan Foto Contoh" and "Ambil dari Galeri" buttons.

### 6.5 `GeolocationService`

- `getCurrentPosition(): Observable<{lat, lng, accuracy, address}>`
- Used by: Clock-In Gate, Check-In Form (location chip), photo stamping.

### 6.6 `ClipboardService`

- `copy(text: string): Promise<void>` — uses `navigator.clipboard.writeText` with `document.execCommand('copy')` fallback.
- Used by referral code chip on Home and Profile.

### 6.7 `DownloadService`

- `save(filename: string, blob: Blob): Promise<void>` — uses browser download API or anchor download.
- Used by photo viewer download buttons.

### 6.8 `ToastService`

- `show(message: string, duration?: number)`: Displays a toast at the bottom of the device (above navbar). Auto-hides after ~2.6s.
- Only one toast at a time (replaces previous).

### 6.9 `ModalService`

- `open(modalId: string)`, `close(modalId: string)`.
- Modals: `success`, `clockInConfirm`, `logoutConfirm`.
- Each overlay modal is a component with `@Input()` data and `@Output()` close/dismiss.

---

## 7. Screens & Components

### 7.1 Splash Component (`/splash`)

**Purpose:** Brand intro screen shown on app launch.

**UI Elements:**
- Full-screen gradient background with `#ringwave` SVG vector.
- Brand logo (WebP, 210px).
- Orange divider line.
- App name: "SALES POINT" (uppercase, letter-spaced).
- Tagline: "Aplikasi akuisisi agen Mini ATM di lapangan".
- Three-dot bounce spinner animation.
- Footer: "VisioNet Mini ATM · v1.0.0".

**Behavior:**
- Auto-navigates to `/login` after ~2.4s.
- If already authenticated (remembered session), skip to `/loading` → `/gate` or `/home`.

---

### 7.2 Login Component (`/login`)

**Purpose:** Agent authentication.

**UI Elements:**
- Header: gradient background with `#dotwave` SVG, brand logo (132px), title "Masuk ke Sales Point", subtitle.
- Bottom sheet (rounded top): white card containing the login form.
- Fields:
  - **Username** (text, required, icon: user). Pre-filled `rizky.pratama` in demo.
  - **Password** (password, required, icon: lock). Pre-filled `salespoint` in demo. Toggle eye button to show/hide.
  - **Remember me** checkbox + **Forgot password** link.
  - **Submit button** ("Masuk") with arrow icon.
- Error alert banner (hidden by default, red soft background).
- Demo note at bottom: credentials hint + version.

**Form:** Template-driven or Reactive form with validators:
- `username`: `required`
- `password`: `required`
- On submit: validate, show inline `.invalid` states, call `AuthService.login()`.
- On failure: show alert with message.
- On success: navigate to `/loading`.

**Interactions:**
- Password toggle: switches `type` between `password` and `text`, swaps eye icon.
- Forgot password link: shows toast "Hubungi supervisor area untuk reset kata sandi."

---

### 7.3 Loading Component (`/loading`)

**Purpose:** Simulated bootstrap/loading sequence.

**UI Elements:**
- Same gradient background as splash with `#ringwave`.
- Brand logo (132px).
- Large percentage display (e.g., "0%").
- Progress bar (track + fill, gradient blue→orange).
- Status text (updates per step).
- Footer: "Mohon tunggu, jangan tutup aplikasi".

**Behavior:**
- Sequential steps with increasing percentage:
  1. 18% — "Memverifikasi kredensial agen…"
  2. 42% — "Memuat profil & area kerja…"
  3. 63% — "Sinkronisasi data absensi…"
  4. 84% — "Menarik data prospek terbaru…"
  5. 100% — "Siap digunakan"
- Each step: ~430–690ms with random jitter.
- On 100%: navigate to `/gate` (if not clocked in) or `/home`.

---

### 7.4 Clock-In Gate Component (`/gate`)

**Purpose:** Pre-work attendance gate. Blocks access to main app until clock-in.

**UI Elements:**
- Header: gradient with `#dotwave`, small brand logo (104px), greeting ("Halo, Rizky"), date ("Jumat, 4 September 2026").
- **Clock card** (overlapping header):
  - Badge: "Belum absen hari ini" (info style).
  - Large live clock (HH:MM:SS, updates every second).
  - Timezone label: "WIB · Waktu Indonesia Barat".
  - Location chip: branch name, address, accuracy.
- Shift info card: "Reguler · 08:00 – 17:00 WIB".
- **Clock In button** (primary, full-width, clock icon).
- Helper text: "Absen wajib dilakukan sebelum memulai kunjungan prospek."

**Behavior:**
- Live clock updates via `setInterval(1000)` — only when this component is active.
- On clock-in: calls `AttendanceStore.clockIn()`, shows `clockInConfirm` modal with time + location recap, then navigates to `/home`.

---

### 7.5 Home Component (`/app/home`)

**Purpose:** Main dashboard after clock-in.

**UI Elements:**
- **Header** (gradient, `#dotwave`):
  - Top row: brand logo (xs) + avatar button (profile link).
  - Welcome: "Welcome back," / "Rizky Pratama".
  - **Referral code chip** (pill, clickable): tag icon, "Kode Referral", code "SP-RZK2041", copy icon. Tapping copies code.
- **Body** (overlaps header by -32px):
  - **Stat solo card**: large number (today's prospect count) + label + date.
  - **Menu grid** (2 columns):
    - "Prospek Toko" → `/checkin`
    - "History Prospek" → `/history`
    Each tile has a 64px illustration image, title, and description.
  - **Attendance card**:
    - Header: status icon (check/pending) + status text + chevron → `/attendance`.
    - Footer: Clock In/Out button (green if not clocked in, red if clocked in but not out, hidden if fully clocked out).
- **Bottom nav**: Home (active), Notifikasi (badge), Profil.

**Behavior:**
- Prospect count derived from `ProspectStore.todayCount`.
- Attendance card state derived from `AttendanceStore.status`.
- Avatar button → `/profile`.
- Referral chip → `ClipboardService.copy('SP-RZK2041')` + toast.

---

### 7.6 Attendance Detail Component (`/app/attendance`)

**Purpose:** Detailed attendance view with today's record and history.

**UI Elements:**
- App bar: back button, "Clock In" title, subtitle "Absensi kehadiran harian".
- **Today card**:
  - Date + shift info.
  - Two-column grid: Clock In time, Clock Out time (or "Belum absen").
  - Meta rows: Location, Duration (live if clocked in), Method ("GPS + Selfie · akurasi ±8 m").
- **Clock Out button** (red, if clocked in and not out).
- **History section**: list of past attendance rows (date, duration, in→out times). Each row is a styled card.
- **Load older button**: loads previous month's records.

**Behavior:**
- Clock Out: calls `AttendanceStore.clockOut()`, shows toast.
- Duration: if currently clocked in, updates live (elapsed time since clock-in).
- Load older: appends `absenOlder` to list, disables button when exhausted.

---

### 7.7 History (Prospek) Component (`/app/history`)

**Purpose:** Searchable, grouped list of all prospect visits.

**UI Elements:**
- App bar: back button, "History Prospek", subtitle (count or search result count).
- **Search bar**: search icon, input, clear button (appears when query non-empty).
- **Results**: grouped by day sections. Each group has a label (day name + count badge) and a list of prospect items.
- **Prospect item** (`ProspectItemComponent`):
  - Thumbnail (initials or photo).
  - Title (store name), subtitle (address), meta (time + PIC name).
  - Chevron → navigates to `/detail/:id`.
- **Load older button**: appends older day groups.
- **Empty state**: icon + message when no results match search.

**Behavior:**
- Search filters across store name, address, and PIC name (case-insensitive).
- Search clears badge count in subtitle.
- Load older: pushes next `DayGroup` from `ProspectStore.olderGroups`, disables button when all loaded.

---

### 7.8 Check-In Form Component (`/app/checkin`)

**Purpose:** Create a new prospect record with required photos.

**UI Elements:**
- App bar: back button, "Prospek Toko", subtitle ("Kunjungan ke-5 hari ini · 13:40 WIB").
- **Location chip**: detected coordinates, address, accuracy.
- **Form** (Reactive Form, `.form-stack`):
  - Nama Toko (text, required)
  - Alamat Toko (textarea, required)
  - Nama PIC (text, required)
  - No. Telp PIC (tel, numeric, required, min 9 digits)
  - **Photo grid** (2 slots):
    - "Foto Plang" → opens camera (`/camera/plang`)
    - "Selfie dengan PIC" → opens camera (`/camera/selfie`)
    - Each slot: empty state (icon + label + hint), filled state (preview image + retake button + tag with timestamp).
  - Catatan Kunjungan (textarea, optional)
- **Form footer** (sticky bottom): "Simpan Prospek" button (primary).

**Validation:**
- All required fields validated on submit.
- Invalid fields get `.invalid` class (red border + error hint).
- Both photos required; `fFoto` shows error if either missing.
- On submit failure: scroll to first invalid field, show toast.

**Behavior:**
- Photo slot click → navigate to `/camera/:kind`.
- Camera component returns photo data URL; form stores it in a local `photos` map.
- On valid submit: constructs `ProspectPayload`, calls `ProspectStore.add()`, shows `success` modal with recap (store name, PIC, phone, time, "2 foto terlampir"), resets form.
- "Check In Lagi" button in modal: closes modal, resets form, stays on page.
- "Selesai" button: closes modal, navigates to `/home`.

---

### 7.9 Camera Component (`/app/camera/:kind`)

**Purpose:** Capture photos (plang or selfie) with live camera stream.

**UI Elements:**
- Full-screen black background.
- **Video stage**: `<video>` element (object-fit cover), mirrored when front camera.
- **Guide overlay**: dashed rounded rectangle (positioned at 14% / 8% / 22%).
- **Flash overlay**: white full-screen div, animates on capture.
- **Top bar** (gradient overlay):
  - Close button (X) → returns to `/checkin`.
  - Title: "Foto Plang Toko" or "Selfie dengan PIC Toko".
  - Flash toggle button (orange when on).
- **Hint text**: instructions (positioned above bottom bar).
- **Error overlay**: shown if camera unavailable. Contains icon, message, and "Gunakan Foto Contoh" button.
- **Bottom bar** (black):
  - Gallery button (left): opens file picker (`<input type="file" accept="image/*">`).
  - **Shutter button** (center): large white circle with inner disc.
  - Flip camera button (right): toggles front/back.

**Behavior:**
- On init: calls `CameraService.open(facing)` where facing = `'user'` for selfie, `'environment'` for plang.
- Shutter:
  1. Trigger flash animation (0.36s).
  2. Draw video frame to canvas (mirror if front cam).
  3. Stamp photo: bottom bar with date/time WIB, coordinates, address (orange accent strip).
  4. Convert to JPEG data URL.
  5. Navigate back to `/checkin`, pass photo data to form.
- Gallery: file picker → read as data URL → stamp → return to form.
- Sample photo: if camera fails, generate a synthetic placeholder image (gradient + label + stamp) and return.
- Flip: toggle facing, restart stream.
- Torch: if supported by track capabilities, apply `torch` constraint; otherwise show toast about screen-flash fallback.
- On destroy: `CameraService.stop()` to release tracks.

---

### 7.10 Prospect Detail Component (`/app/detail/:id`)

**Purpose:** View full details of a saved prospect.

**UI Elements:**
- App bar: back button (→ `/history`), "Detail Prospek", subtitle (store name).
- **Photo section** ("Dokumentasi Foto"):
  - Two `photo-card` buttons side by side (3:4 aspect ratio):
    - Plang photo → opens `/viewer/:id/plang`
    - Selfie photo → opens `/viewer/:id/selfie`
  - Each card: image + caption bar (label + expand icon).
- **Store data** ("Data Toko"): info card with rows:
  - Nama Toko (store icon)
  - Alamat Toko (pin icon)
  - Nama PIC (user icon)
  - No. Telp PIC (phone icon)
  - Waktu Kunjungan (clock icon)
  - Titik Koordinat (pin icon)
- **Notes** ("Catatan Kunjungan"): dashed-border box with note text or "Tidak ada catatan tambahan."
- **Actions**:
  - "Unduh Kedua Foto" (primary) → downloads both photos sequentially.
  - "Hubungi PIC" (ghost) → `tel:` link with PIC phone number.

**Behavior:**
- On init: `ProspectStore.getById(routeParamId)` → populate all fields.
- If prospect has no stored photos (older records), generate synthetic demo photos via canvas (plang: storefront illustration; selfie: portrait illustration), both stamped with geo/time.

---

### 7.11 Photo Viewer Component (`/app/viewer/:id/:kind`)

**Purpose:** Full-screen photo viewer with download.

**UI Elements:**
- Black full-screen background.
- **Image stage**: centered image, `object-fit: contain`.
- **Top bar**: close button, title, spacer.
- **Bottom bar** (black):
  - **Segment toggle**: "Plang" / "Selfie" — switches between the two photos of the same prospect.
  - **Download button** (primary): "Unduh Foto Ini".
  - **Hint**: "Di ponsel, tekan lama pada foto untuk menyimpannya ke galeri."

**Behavior:**
- `kind` param determines initial photo shown.
- Segment toggle updates image source and `kind`.
- Download: calls `DownloadService.save()` with filename `Prospek_{StoreName}_{plang|selfie}.jpg`.
- Close → returns to `/detail/:id`.

---

### 7.12 Notifications Component (`/app/notifications`)

**Purpose:** View and manage notifications.

**UI Elements:**
- App bar: back button (→ `/home`), "Notifikasi", subtitle (unread count or "Semua sudah dibaca"), "Mark all read" button (double-check icon).
- **List**: grouped by day ("Hari Ini", "Kemarin", "Rabu, 2 September 2026").
  - Each group: label with unread count badge.
  - Each notification (`NotificationItemComponent`):
    - Icon (colored: blue/green/orange/red).
    - Title, body, time.
    - Unread indicator: left accent border + blue dot.
- **Bottom nav**: Home, Notifikasi (active, badge), Profil.

**Behavior:**
- Tap a notification → marks as read, updates badge.
- "Mark all read" button → sets all `unread = false`, toast confirmation.
- Badge count on bottom nav synced with `NotificationStore.unreadCount`.

---

### 7.13 Profile Component (`/app/profile`)

**Purpose:** Agent profile, stats, chart, and logout.

**UI Elements:**
- **Header** (gradient, `#dotwave`):
  - Back button (→ `/home`).
  - Large avatar (64px, orange gradient, initials "RP").
  - Name: "Rizky Pratama".
  - Username: "@rizky.pratama".
  - Badge: "Sales Point Agent · Aktif".
- **Body**:
  - **Info card**: Username, Nama Lengkap, Kode Referral (with copy button), Email.
  - **Chart card** ("Prospek per Bulan"):
    - Header: title + total (677).
    - SVG bar chart (custom component `MonthlyChartComponent`): 9 bars (Jan–Sep), blue for completed months, orange for current month. Y-axis ticks (0, 40, 80, 120). Tooltip on hover/tap showing month + count.
    - Legend: "Bulan selesai" (blue), "Bulan berjalan" (orange).
  - **Stats card**: Prospek bulan ini (38, +12% badge), Kehadiran bulan ini (4 hari, 100%, "Baik" badge).
  - **Logout button** (danger): triggers logout confirmation modal.
  - **Version label**: "Sales Point v1.0.0", build date, copyright.
- **Bottom nav**: Home, Notifikasi (badge), Profil (active).

**Behavior:**
- Copy referral: `ClipboardService.copy('SP-RZK2041')` + toast.
- Chart: rendered as SVG `<path>` bars with rounded tops. Hover/tap shows tooltip positioned above the bar. Tooltip hides on mouseleave.
- Logout: shows `logoutConfirm` modal ("Keluar dari Aplikasi?"). Warning text about clock-out. "Ya, Keluar" → full reset (stop camera, reset form, reset attendance, clear session), navigate to `/splash` → `/login`.

---

## 8. Shared/Dumb Components

| Component | Selector | Purpose |
|---|---|---|
| `AppBarComponent` | `app-bar` | Reusable header: back button, title, subtitle, optional action button. Uses `#dotwave` vector. |
| `BottomNavComponent` | `app-bottom-nav` | 3-tab navigation with active state and notification badge. |
| `BadgeComponent` | `app-badge` | Pill badge with variants: `good`, `warn`, `crit`, `info`. |
| `IconButtonComponent` | `app-icon-btn` | 38px rounded icon button (transparent white bg on dark headers). |
| `FieldComponent` | `app-field` | Form field wrapper: label (with `*` for required), input, error hint. |
| `InputDirective` | `appInput` | Styled input with icon prefix, focus state, error border. |
| `ButtonComponent` | `app-btn` | Button with variants: `primary`, `ghost`, `danger`, `red`, `green`, `sm`. |
| `CardComponent` | `app-card` | White rounded card with border + shadow. |
| `StatCellComponent` | `app-stat-cell` | Stat cell in a 3-column grid: large value + label. |
| `MeterComponent` | `app-meter` | Progress meter: track + fill + caption. |
| `ProspectItemComponent` | `app-prospect-item` | List item for a prospect (thumbnail, title, address, meta, chevron). Click → emit navigate. |
| `DayGroupComponent` | `app-day-group` | Section label with count badge + list of items. |
| `NotificationItemComponent` | `app-notif-item` | Single notification row with icon, text, time, unread dot. |
| `AttendRowComponent` | `app-attend-row` | Single attendance history row (date, duration, times). |
| `PhotoSlotComponent` | `app-photo-slot` | Empty/filled photo slot for check-in form. Emits click to open camera. |
| `PhotoCardComponent` | `app-photo-card` | Detail screen photo card with caption overlay. Emits click to open viewer. |
| `ToastComponent` | `app-toast` | Global toast notification (fixed position, auto-hide). |
| `ModalComponent` | `app-modal` | Overlay modal: icon, title, body, recap, action buttons. |
| `MonthlyChartComponent` | `app-monthly-chart` | SVG bar chart with tooltip. `@Input() data: ChartDatum[]`. |
| `LocChipComponent` | `app-loc-chip` | Location info chip (icon, title, subtitle). |
| `SearchBarComponent` | `app-search-bar` | Search input with icon and clear button. |
| `SpinnerDotsComponent` | `app-spinner-dots` | Three-dot bounce animation for splash. |

---

## 9. Forms

### 9.1 Login Form (Template-driven)

```ts
// LoginComponent
form = {
  username: 'rizky.pratama',  // demo pre-fill
  password: 'salespoint',    // demo pre-fill
  rememberMe: true
};
```

- Validators: `username` required, `password` required.
- Inline validation: `.field.invalid` class toggles error hint visibility.
- Alert banner: shown on auth failure, hidden on success.

### 9.2 Check-In (Prospect) Form (Reactive)

```ts
checkinForm = this.fb.group({
  toko:    ['', Validators.required],
  alamat:  ['', Validators.required],
  pic:     ['', Validators.required],
  telp:    ['', [Validators.required, Validators.minLength(9)]],
  catatan: ['']
});
```

- Phone field: `inputmode="numeric"`, digits stripped on validation.
- Photo validation: handled separately (not in form group) — `photos = { plang: null, selfie: null }`.
- On submit: validate form + photos. If invalid, mark fields invalid, scroll to first error.
- On success: construct payload with all fields + photo data URLs + timestamp, call `ProspectStore.add()`.

---

## 10. UI/UX Behavior

### 10.1 Animations

| Animation | Trigger | Duration | Notes |
|---|---|---|---|
| Screen in | Route activation | 0.26s | `translateY(8px) → 0` + fade |
| Splash rise | Splash mount | 0.7s | `translateY(14px) scale(.97) → 0` + fade |
| Spinner bounce | Splash mount | 1.05s infinite | 3 dots, staggered 0.16s |
| Modal pop | Overlay open | 0.28s | `scale(.9) translateY(10px) → 1` |
| Toast in | Toast show | 0.28s | `translateY(12px) → 0` + fade |
| Camera flash | Shutter click | 0.34s | White overlay opacity 0.95 → 0 |
| Progress fill | Loading steps | 0.35s each | `width` transition |
| Button press | `:active` | 0.12s | `scale(.985)` |

All animations respect `prefers-reduced-motion: reduce` (duration → 0.001ms).

### 10.2 Scroll Behavior

- All scrollable areas use `.scroll` class with `overscroll-behavior: contain`, hidden scrollbars, `-webkit-overflow-scrolling: touch`.
- On navigation, scroll position resets to top.

### 10.3 Safe Areas

- Bottom nav padding: `calc(7px + env(safe-area-inset-bottom))`.
- Sticky form footer: `calc(12px + env(safe-area-inset-bottom))`.
- Splash footer: `calc(20px + env(safe-area-inset-bottom))`.

### 10.4 Focus & Accessibility

- `:focus-visible` outline: 2px solid accent, offset 2px, border-radius 6px.
- All interactive elements have `aria-label` where text is absent.
- Modals: `role="dialog"`, `aria-modal="true"`.
- Chart: `role="img"` with descriptive `aria-label`.
- Camera: `aria-label` on shutter, close, flip, gallery, flash buttons.

### 10.5 Toast

- Positioned: `bottom: calc(84px + env(safe-area-inset-bottom))`, left/right 18px.
- Navy background, white text, orange icon.
- Auto-hide: 2.6s. Replaces any existing toast.

### 10.6 Modals

| Modal ID | Trigger | Content |
|---|---|---|
| `success` | Prospect saved | Green check, "Prospek Berhasil Disimpan", recap (store, PIC, phone, time, photos), "Selesai" + "Check In Lagi" |
| `clockInConfirm` | Clock-in done | Green check, "Clock In Berhasil", recap (time, location), "Lanjut ke Beranda" |
| `logoutConfirm` | Logout button | Warning icon, "Keluar dari Aplikasi?", warning about clock-out, "Ya, Keluar" + "Batal" |

---

## 11. Data Models

```ts
interface User {
  username: string;
  fullName: string;
  email: string;
  referralCode: string;
  role: string;
  avatarInitials: string;
}

interface AttendanceRecord {
  in: string | null;      // "07:48"
  out: string | null;     // "17:10"
  date: string;
  shift: string;
  location: string;
  method: string;
}

interface Prospect {
  id: string;
  n: string;              // store name
  a: string;              // address
  pic: string;
  telp: string;
  t: string;              // time "HH:MM"
  tgl: string;            // date string
  note: string;
  photoPlang?: string;    // data URL
  photoSelfie?: string;   // data URL
  status?: string;        // for older records
  statusClass?: string;
}

interface DayGroup {
  d: string;              // day label
  items: Prospect[];
}

interface NotifItem {
  ic: 'blue' | 'green' | 'orange' | 'red';
  t: string;              // title
  b: string;              // body
  time: string;
  unread: boolean;
  svg: string;            // icon path
}

interface NotifGroup {
  d: string;              // day label