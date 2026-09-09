# Product Requirements Document (PRD)
## Sales Point — Angular Frontend

> **VisioNet Mini ATM · Sales Point** — Field-agent acquisition app for Mini ATM placement at retail stores.  
> **Platform:** Mobile-first PWA (Angular standalone components)  
> **Target devices:** Android / iOS mobile browsers, installable PWA  
> **Scope:** Frontend only (Angular 17+ standalone, signals, zoneless-ready)

---

## 1. Overview & Scope

### 1.1 Purpose
A mobile-first Angular application enabling field sales agents to:
- Authenticate against a backend agent portal.
- Clock in / clock out with GPS + selfie.
- Capture new store prospects (Prospek Toko) with geo-stamped documentation photos.
- Browse prospect history with search.
- View notifications and a personal performance dashboard (monthly chart).

### 1.2 Out of Scope (Backend responsibilities)
- Actual auth token issuance / session validation.
- Persistence of prospects, attendance, notifications.
- Geocoding / reverse-geocoding services.
- File storage for uploaded photos.

The frontend will interact with a REST/JSON API (contract defined in §6 Services).

---

## 2. Technology Stack

| Layer | Choice | Rationale |
|---|---|---|
| Framework | Angular 17+ (standalone components, signals) | Modern, tree-shakeable, signal-based reactivity |
| Routing | `@angular/router` with lazy-loaded routes | Code-split per feature |
| State | **NgRx SignalStore** (component-store pattern) | Lightweight, signal-native; avoids NgRx boilerplate |
| Forms | Reactive Forms + custom validators | Predictable, testable |
| HTTP | `HttpClient` with interceptors | Token refresh, error normalisation |
| Styling | SCSS with CSS custom properties (design tokens) | Exact port of mockup token system |
| Charts | Custom SVG component (no chart lib) | Matches mockup; tiny footprint |
| Camera | `getUserMedia` + `<canvas>` capture | Native browser API |
| Icons | Inline SVG sprite component | Matches mockup stroke style |
| PWA | `@angular/service-worker` | Offline shell, manifest |
| Testing | Jest + Playwright | Unit + E2E |

---

## 3. Architecture & Module Structure

### 3.1 High-Level Architecture

```
src/
├── app/
│   ├── app.config.ts              // ApplicationConfig (providers, router, SW)
│   ├── app.routes.ts              // Top-level route config
│   ├── core/                      // Singleton services, guards, interceptors
│   │   ├── services/
│   │   ├── guards/
│   │   ├── interceptors/
│   │   └── models/                // TypeScript interfaces & DTOs
│   ├── shared/                    // Reusable UI components, directives, pipes
│   │   ├── components/            // Button, Card, Badge, Toast, Modal, etc.
│   │   ├── directives/
│   │   └── pipes/
│   ├── stores/                    // SignalStore definitions (global + feature)
│   │   ├── auth.store.ts
│   │   ├── attendance.store.ts
│   │   ├── prospect.store.ts
│   │   ├── notification.store.ts
│   │   └── ui.store.ts            // Toast, modal, active-tab, screen state
│   └── features/                  // Lazy-loaded feature areas
│       ├── auth/                  // Splash + Login + Loading
│       ├── gate/                  // Clock-in gate
│       ├── home/                  // Dashboard
│       ├── attendance/            // Absen detail
│       ├── prospect/              // Check-in form + Detail + Photo viewer
│       │   ├── check-in/
│       │   ├── detail/
│       │   ├── history/
│       │   └── camera/
│       ├── notification/
│       └── profile/
├── assets/
│   └── scss/
│       ├── _tokens.scss           // CSS custom properties
│       ├── _mixins.scss
│       └── _base.scss
└── styles.scss
```

### 3.2 Feature Modules (Lazy-Loaded Routes)

Each feature is a standalone-component route group with its own route file:

| Feature | Route prefix | Components |
|---|---|---|
| Auth | `/auth` | Splash, Login, Loading |
| Gate | `/gate` | ClockInGate |
| Home | `/home` | Home |
| Attendance | `/attendance` | AbsenDetail |
| Prospect / Check-in | `/prospect/new` | CheckInForm |
| Prospect / History | `/prospect/history` | HistoryList |
| Prospect / Detail | `/prospect/:id` | ProspectDetail |
| Prospect / Camera | `/prospect/camera` | CameraCapture |
| Prospect / Viewer | `/prospect/viewer` | PhotoViewer |
| Notification | `/notifications` | NotificationList |
| Profile | `/profile` | Profile |

---

## 4. Routing Configuration

### 4.1 Route Tree

```ts
// app.routes.ts
export const APP_ROUTES: Routes = [
  {
    path: 'auth',
    loadComponent: () => import('./features/auth/auth.routes').then(m => m.AUTH_ROUTES),
  },
  {
    path: 'gate',
    canActivate: [authGuard, clockInGuard],
    loadComponent: () => import('./features/gate/gate.component').then(m => m.ClockInGateComponent),
  },
  {
    path: 'home',
    canActivate: [authGuard],
    loadComponent: () => import('./features/home/home.component').then(m => m.HomeComponent),
  },
  {
    path: 'attendance',
    canActivate: [authGuard],
    loadComponent: () => import('./features/attendance/attendance.component').then(m => m.AttendanceComponent),
  },
  {
    path: 'prospect',
    canActivate: [authGuard, attendanceGuard],
    loadChildren: () => import('./features/prospect/prospect.routes').then(m => m.PROSPECT_ROUTES),
  },
  {
    path: 'notifications',
    canActivate: [authGuard],
    loadComponent: () => import('./features/notification/notification.component').then(m => m.NotificationComponent),
  },
  {
    path: 'profile',
    canActivate: [authGuard],
    loadComponent: () => import('./features/profile/profile.component').then(m => m.ProfileComponent),
  },
  { path: '', redirectTo: 'auth/splash', pathMatch: 'full' },
  { path: '**', redirectTo: 'auth/splash' },
];
```

### 4.2 Prospect Sub-routes

```ts
// prospect.routes.ts
export const PROSPECT_ROUTES: Routes = [
  { path: 'new',     component: CheckInFormComponent },
  { path: 'history', component: HistoryListComponent },
  { path: ':id',     component: ProspectDetailComponent },
  { path: 'camera',  component: CameraCaptureComponent },
  { path: 'viewer',  component: PhotoViewerComponent },
  { path: '',        redirectTo: 'history', pathMatch: 'full' },
];
```

### 4.3 Route Guards

| Guard | Purpose |
|---|---|
| `authGuard` | Redirects unauthenticated users to `/auth/login`. Checks `authStore.isAuthenticated()`. |
| `clockInGuard` | Allows `/gate` only when not yet clocked in; redirects to `/home` if already clocked in. |
| `attendanceGuard` | Blocks `/prospect/new` if not clocked in; redirects to `/gate`. |

### 4.4 Navigation Strategy

- **No `routerLink` for in-app "screens" that share the device-shell.** The mockup uses a custom screen-transition system (absolute-positioned screens with fade/slide). To preserve UX fidelity, the app uses **`provideRouter` with `InMemoryScrolling`** and a **custom `ScreenTransitionService`** that wraps `Router.navigate()` to add the `screen-in` animation class.
- Bottom-nav tabs (`Home`, `Notifikasi`, `Profil`) use standard `routerLink` with `routerLinkActive`.
- Back buttons call `Location.back()` or navigate to a defined parent route.

### 4.5 Route Data & Chrome Toggling

Each route sets `data: { chrome: 'navy' | 'black' }`:
- `black` → camera and photo-viewer screens (status bar background black).
- `navy` → all other screens.

A `ChromeDirective` on the root `<app-root>` host reads `ActivatedRoute` data and toggles a `data-chrome` attribute consumed by SCSS.

---

## 5. State Management Strategy

### 5.1 Approach: NgRx SignalStore (component + global)

Use `@ngrx/signals` `signalStore` for global cross-feature state, and `withComponentStore`-equivalent local stores for screen-scoped state (camera, check-in form).

### 5.2 Global Stores

#### 5.2.1 `AuthStore`

```ts
export const AuthStore = signalStore(
  { providedIn: 'root' },
  withState<AuthState>({
    status: 'idle',           // 'idle' | 'authenticating' | 'authenticated' | 'error'
    user: null,                // AgentProfile | null
    token: null,               // string | null
    rememberMe: true,
    error: null,               // string | null
  }),
  withComputed(({ user }) => ({
    initials: computed(() => user()?.fullName
      ? user()!.fullName.split(' ').slice(0, 2).map(w => w[0]).join('').toUpperCase()
      : ''),
    referralCode: computed(() => user()?.referralCode ?? null),
  })),
  withMethods((store, authApi = inject(AuthApiService)) => ({
    async login(credentials: LoginCredentials) { /* ... */ },
    logout() { /* ... */ },
  })),
  withHooks({
    onInit(store) { /* hydrate from localStorage if rememberMe */ },
  }),
);
```

#### 5.2.2 `AttendanceStore`

| Signal | Type | Description |
|---|---|---|
| `today` | `AttendanceRecord \| null` | Today's clock-in/out record |
| `history` | `AttendanceRecord[]` | Past 5 days |
| `olderHistory` | `AttendanceRecord[]` | Loaded on demand |
| `status` | `'idle' \| 'loading' \| 'loaded' \| 'error'` | |
| `hasOlder` | `boolean` | More history available? |

Methods: `clockIn()`, `clockOut()`, `loadHistory()`, `loadOlder()`.

Computed: `isClockedIn`, `isClockedOut`, `durationLabel`.

#### 5.2.3 `ProspectStore`

| Signal | Type |
|---|---|
| `today` | `Prospect[]` |
| `groups` | `ProspectDayGroup[]` (today + loaded older days) |
| `olderRemaining` | `number` |
| `detail` | `Prospect \| null` |
| `searchQuery` | `string` |
| `photos` | `{ plang: string \| null; selfie: string \| null }` (active check-in) |
| `status` | `'idle' \| 'loading' \| 'saving' \| 'saved' \| 'error'` |

Methods: `save(prospect)`, `loadOlder()`, `setSearch(q)`, `selectDetail(id)`, `resetPhotos()`.

#### 5.2.4 `NotificationStore`

| Signal | Type |
|---|---|
| `groups` | `NotificationDayGroup[]` |
| `unreadCount` | `number` |

Methods: `markRead(id)`, `markAllRead()`.

#### 5.2.5 `UiStore`

Transient UI state shared across components:

| Signal | Type | Description |
|---|---|---|
| `toast` | `{ message: string; visible: boolean } \| null` | Current toast |
| `activeModal` | `'success' \| 'clock-in' \| 'logout' \| null` | Currently open modal |
| `deviceChrome` | `'navy' \| 'black'` | Status bar / device-shell colour |

Methods: `showToast(msg, ms?)`, `openModal(id)`, `closeModal()`.

### 5.3 Local (Component-Scoped) Stores

- **`CameraStore`** (provided in `CameraCaptureComponent`): `stream`, `facingMode`, `torch`, `error`, `kind`.
- **`CheckInFormStore`** (provided in `CheckInFormComponent`): wraps `FormGroup`, photo slots, validation flags.

---

## 6. Core Models & Interfaces

```ts
// core/models/agent.model.ts
export interface AgentProfile {
  id: string;
  username: string;
  fullName: string;
  email: string;
  role: 'agent';
  referralCode: string;        // e.g. "SP-RZK2041"
  area: string;                 // e.g. "Jakarta Selatan"
  branch: string;               // e.g. "Kantor Cabang Jakarta Selatan"
  status: 'active' | 'suspended';
  avatarUrl?: string;
}

// core/models/attendance.model.ts
export interface AttendanceRecord {
  id: string;
  date: string;                 // ISO date (YYYY-MM-DD)
  clockIn?: string;             // "HH:mm"
  clockOut?: string;            // "HH:mm"
  location: GeoLocation;
  method: 'gps_selfie' | 'gps';
  shift: { name: string; start: string; end: string };
  status: 'on_time' | 'late' | 'permission';
}

export interface GeoLocation {
  lat: number;
  lng: number;
  address: string;
  accuracyMeters: number;
  branch?: string;
}

// core/models/prospect.model.ts
export type ProspectStatus = 'verified' | 'pending' | 'rejected';

export interface Prospect {
  id: string;
  storeName: string;
  address: string;
  picName: string;
  picPhone: string;
  visitTime: string;            // "HH:mm"
  visitDate: string;            // "4 September 2026"
  status?: ProspectStatus;
  note?: string;
  photoPlang?: string;          // data URL or CDN URL
  photoSelfie?: string;
  location?: { lat: number; lng: number };
  createdAt: string;            // ISO datetime
}

export interface ProspectDayGroup {
  dateLabel: string;            // "Hari Ini — Jumat, 4 September 2026"
  items: Prospect[];
}

// core/models/notification.model.ts
export type NotificationKind = 'success' | 'reminder' | 'info' | 'warning';

export interface AppNotification {
  id: string;
  kind: NotificationKind;
  title: string;
  body: string;
  timestamp: string;            // ISO
  read: boolean;
}

export interface NotificationDayGroup {
  dateLabel: string;
  items: AppNotification[];
}

// core/models/chart.model.ts
export interface MonthlyDataPoint {
  month: string;                // "Jan"
  value: number;
  isCurrent?: boolean;
}
```

---

## 7. Services Layer

### 7.1 API Services (HTTP)

All API services inject `HttpClient` and return observables converted to signals via `toSignal` where appropriate.

#### `AuthApiService`

| Method | Endpoint | Body / Params | Returns |
|---|---|---|---|
| `login(creds)` | `POST /api/auth/login` | `{ username, password, rememberMe }` | `{ token, user: AgentProfile }` |
| `logout()` | `POST /api/auth/logout` | — | `void` |
| `refreshProfile()` | `GET /api/auth/me` | — | `AgentProfile` |

#### `AttendanceApiService`

| Method | Endpoint | Returns |
|---|---|---|
| `getToday()` | `GET /api/attendance/today` | `AttendanceRecord \| null` |
| `clockIn(payload)` | `POST /api/attendance/clock-in` | `AttendanceRecord` |
| `clockOut()` | `POST /api/attendance/clock-out` | `AttendanceRecord` |
| `getHistory(monthOffset)` | `GET /api/attendance/history?offset={n}` | `AttendanceRecord[]` |

#### `ProspectApiService`

| Method | Endpoint | Returns |
|---|---|---|
| `getToday()` | `GET /api/prospects/today` | `Prospect[]` |
| `getOlder(cursor)` | `GET /api/prospects?cursor={id}&limit=20` | `{ items: Prospect[]; nextCursor: string \| null }` |
| `getById(id)` | `GET /api/prospects/{id}` | `Prospect` |
| `create(payload)` | `POST /api/prospects` (multipart) | `Prospect` |
| `search(q)` | `GET /api/prospects/search?q={q}` | `Prospect[]` |

#### `NotificationApiService`

| Method | Endpoint | Returns |
|---|---|---|
| `getAll()` | `GET /api/notifications` | `NotificationDayGroup[]` |
| `markRead(id)` | `PATCH /api/notifications/{id}/read` | `void` |
| `markAllRead()` | `POST /api/notifications/read-all` | `void` |

#### `ChartApiService`

| Method | Endpoint | Returns |
|---|---|---|
| `getMonthly()` | `GET /api/stats/monthly` | `MonthlyDataPoint[]` |

### 7.2 Infrastructure Services

#### `HttpInterceptor` (`authInterceptor`)

```ts
export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const auth = inject(AuthStore);
  const token = auth.token();
  const cloned = token
    ? req.clone({ setHeaders: { Authorization: `Bearer ${token}` } })
    : req;
  return next(cloned).pipe(
    catchError((err: HttpErrorResponse) => {
      if (err.status === 401) {
        auth.logout();
        inject(Router).navigate(['/auth/login']);
      }
      return throwError(() => err);
    }),
  );
};
```

#### `LoadingSequenceService`

Replicates the splash → login → loading → gate sequence. Steps:
1. `authenticating` (18%) — verify credentials
2. `loading-profile` (42%) — load agent profile + work area
3. `syncing-attendance` (63%) — pull today's attendance
4. `pulling-prospects` (84%) — pull today's prospects
5. `ready` (100%) — navigate to `/gate` or `/home`

Each step emits `{ progress: number; status: string }` via a signal; UI binds to it. Randomised 430–690 ms delay per step (matching mockup feel).

#### `GeolocationService`

```ts
@Injectable({ providedIn: 'root' })
export class GeolocationService {
  getCurrentPosition(highAccuracy = true): Promise<GeoLocation> { /* wrapper around navigator.geolocation.getCurrentPosition */ }
  watchPosition(cb: (loc: GeoLocation) => void): () => void { /* ... */ }
}
```

Used by: Gate, Check-in form, Clock-in/out.

#### `CameraService`

Encapsulates `getUserMedia`, torch capability detection, facing-mode switching, and canvas capture with geo/time stamping. See §10 Camera Feature.

#### `ClipboardService`

Wraps `navigator.clipboard.writeText` with `document.execCommand('copy')` fallback (for older WebViews). Used for referral code copy.

#### `DownloadService`

Wraps `window.claude.use('downloads')` (PWA download API) when available, falls back to `<a download>` anchor. Used for prospect photo downloads.

#### `ToastService` (thin wrapper around `UiStore`)

```ts
@Injectable({ providedIn: 'root' })
export class ToastService {
  private ui = inject(UiStore);
  show(message: string, durationMs = 2600): void {
    this.ui.showToast({ message, visible: true });
    setTimeout(() => this.ui.hideToast(), durationMs);
  }
}
```

#### `ScreenTransitionService`

Wraps `Router.navigate()` to apply the mockup's `screen-in` animation:

```ts
@Injectable({ providedIn: 'root' })
export class ScreenTransitionService {
  navigate(commands: any[], extras?: NavigationExtras): Promise<boolean> {
    // Optionally pre-add 'leaving' class to current view
    return this.router.navigate(commands, extras);
  }
}
```

The `AppComponent` template listens to `NavigationEnd` and toggles `.active` on the routed `<router-outlet>` container to trigger the `screen-in` keyframe.

---

## 8. Feature Modules & Components

### 8.1 Auth Feature

#### 8.1.1 `SplashComponent`
- **Route:** `/auth/splash`
- **Duration:** ~2.4 s, then auto-redirect to `/auth/login`.
- **Elements:** Brand logo (WebP asset), divider, app name, tagline, three-dot spinner animation, version footer.
- **No user interaction.** Reduced-motion: skip spinner animation.

#### 8.1.2 `LoginComponent`
- **Route:** `/auth/login`
- **Form:** Reactive `FormGroup` with `username` (required), `password` (required), `rememberMe` (checkbox, default true).
- **Behaviour:**
  - Password visibility toggle button (eye icon swap).
  - Inline validation error messages (`hint-err`) shown when field touched & invalid.
  - Login alert banner (`login-alert`) shown on auth failure; message from API or default "Username atau kata sandi salah."
  - On success → call `LoadingSequenceService.start()` and navigate to `/auth/loading`.
  - "Lupa kata sandi?" link → toast "Hubungi supervisor area untuk reset kata sandi."
  - Demo note footer with credentials hint.
- **Validation:** `username: [required, minLength(3)]`, `password: [required]`.

#### 8.1.3 `LoadingComponent`
- **Route:** `/auth/loading`
- **Elements:** Brand logo, percentage text (`load-pct`), progress bar (`load-bar`), status text (`load-status`), footer warning.
- **Behaviour:** Binds to `LoadingSequenceService` signals. On `ready` (100%), wait 420 ms, then navigate to `/gate` (or `/home` if already clocked in).
- **Reduced motion:** Progress bar transition duration reduced to ~1 ms.

---

### 8.2 Gate Feature

#### `ClockInGateComponent`
- **Route:** `/gate`
- **Guard:** `authGuard` (must be authenticated), `clockInGuard` (must NOT yet be clocked in).
- **Elements:**
  - Header (vector background, brand logo XS), greeting ("Halo, {firstName}"), today's date.
  - Live clock card: HH:MM:SS (updates every second via `interval(1000)` → signal).
  - Badge "Belum absen hari ini".
  - Location chip: branch name, address, accuracy.
  - Shift info card.
  - Primary button "Clock In Sekarang".
- **Behaviour:**
  - On mount, fetch `AttendanceStore.today()`. If already clocked in → redirect `/home`.
  - Button click → `AttendanceStore.clockIn()`:
    1. Acquire `GeolocationService.getCurrentPosition()`.
    2. POST to API.
    3. On success → open `ovClockIn` success modal with recap.
    4. Modal "Lanjut ke Beranda" → navigate `/home`.

---

### 8.3 Home Feature

#### `HomeComponent`
- **Route:** `/home`
- **Layout:** Scrollable body + sticky bottom nav.
- **Header (gradient + vector bg):**
  - Brand logo XS (left-centre), avatar button (right) → navigates `/profile`.
  - Welcome block: "Welcome back," + agent full name.
  - Referral chip button → `ClipboardService.copy(referralCode)`.
- **Body:**
  - **Stat-solo card:** Big number = `prospectStore.today().length`, label "Prospek hari ini", today's date.
  - **Menu grid (2 columns):**
    - "Prospek Toko" tile → `/prospect/new`.
    - "History Prospek" tile → `/prospect/history`.
  - **Attendance card:**
    - Status row (icon + label): "Sudah clock in · 07:48 WIB" or "Belum clock in hari ini".
    - Action button: "Clock In Sekarang" (green, if not clocked in) OR "Clock Out Sekarang" (red, if clocked in but not out) OR hidden (if both done).
    - Card body click → `/attendance`.
- **Bottom nav:** Home (active), Notifikasi (with unread badge), Profil.

---

### 8.4 Attendance Feature

#### `AttendanceComponent`
- **Route:** `/attendance`
- **AppBar:** Back button (→ `/home`), title "Clock In", subtitle "Absensi kehadiran harian".
- **Body:**
  - Section "Absen Hari Ini":
    - Card with date, shift, today's clock-in/out times (two cells in `tt-grid`).
    - Meta rows: location, duration (live updating if clocked in & not out), method.
    - Clock Out button (red, disabled after clock out).
  - Section "Riwayat Absen":
    - List of `attend-row` items (date, duration, in→out times).
    - "Muat riwayat bulan lalu" load-more button.
- **Behaviour:** Duration label computed reactively from `clockIn` time and `now` when not yet clocked out.

---

### 8.5 Prospect Feature

#### 8.5.1 `CheckInFormComponent`
- **Route:** `/prospect/new`
- **Guard:** `attendanceGuard` (must be clocked in).
- **AppBar:** Back (→ `/home`), title "Prospek Toko", subtitle "Kunjungan ke-{n} hari ini · {HH:mm} WIB".
- **Location chip** (auto-detected on init).
- **Form (Reactive `FormGroup`):**

| Field | Control | Validators |
|---|---|---|
| Nama Toko | `storeName` | `required` |
| Alamat Toko | `address` | `required` |
| Nama PIC | `picName` | `required` |
| No. Telp PIC | `picPhone` | `required`, `minLength(9)`, phone pattern |
| Catatan Kunjungan | `note` | (optional) |

- **Photo grid (2 slots):**
  - `slotPlang` ("Foto Plang") and `slotSelfie` ("Selfie dengan PIC").
  - Each slot is a button → navigates to `/prospect/camera` with state `{ kind: 'plang' \| 'selfie' }`.
  - On return from camera, slot shows captured image, retake button, time tag.
  - Both photos required to save (custom group validator).
- **Footer (sticky):** "Simpan Prospek" button.
- **On submit:**
  1. Validate all fields + photos.
  2. If invalid → set `invalid` class on failing fields, scroll first invalid into view, toast "Lengkapi data bertanda * sebelum menyimpan."
  3. If valid → `ProspectStore.save()`:
     - Build multipart form-data (photos as `Blob`).
     - POST to API.
     - On success → open `ovSuccess` modal with recap (nama toko, PIC, telp, waktu, "2 foto terlampir").
     - Modal buttons: "Selesai" (→ reset form + navigate `/home`) or "Check In Lagi" (→ reset form, stay).

#### 8.5.2 `HistoryListComponent`
- **Route:** `/prospect/history`
- **AppBar:** Back (→ `/home`), title "History Prospek", subtitle "{n} prospek hari ini" (or search result count).
- **Search bar:**
  - `<input type="search">` with magnifier icon.
  - Clear button (X) appears when query non-empty.
  - Filters across all loaded groups (today + older) by store name, address, PIC name.
  - Debounced 200 ms via `toSignal(form.valueChanges)`.
- **List:**
  - Day groups with label + count badge.
  - Each item (`pitem`): thumbnail (initials or photo), store name, address, time + PIC meta, chevron.
  - Click → `/prospect/{id}`.
- **Load more:** "Muat data sebelumnya" button → `ProspectStore.loadOlder()`. Disables when exhausted.

#### 8.5.3 `ProspectDetailComponent`
- **Route:** `/prospect/:id`
- **AppBar:** Back (→ `/prospect/history`), title "Detail Prospek", subtitle = store name.
- **Sections:**
  - "Dokumentasi Foto" — 2 photo cards (plang, selfie). Click → `/prospect/viewer?kind={kind}`.
  - "Data Toko" — info-card rows: nama toko, alamat, PIC, telp, waktu kunjungan, koordinat.
  - "Catatan Kunjungan" — note box.
  - Actions: "Unduh Kedua Foto" (primary), "Hubungi PIC" (ghost → `tel:` link).
- **Behaviour:**
  - On init → `ProspectStore.selectDetail(id)` fetches from store or API.
  - If photos not yet loaded (store cache miss), component generates placeholder via `PhotoPlaceholderService` (matches mockup's canvas-generated demo photos).

#### 8.5.4 `CameraCaptureComponent`
- **Route:** `/prospect/camera`
- **State:** `{ kind: 'plang' | 'selfie' }` passed via router state or query param.
- **Elements:**
  - `<video>` (autoplay, muted, playsinline) with mirror transform when front camera.
  - Dashed guide overlay.
  - Top bar: close (X), title, flash toggle.
  - Hint text (changes by kind).
  - Bottom bar: gallery button (left), shutter (centre), flip button (right).
  - Error overlay (when `getUserMedia` fails): message + "Gunakan Foto Contoh" button.
- **Behaviour:**
  - On init → request `getUserMedia({ video: { facingMode } })`.
  - Flip button → toggle `user`/`environment`, restart stream.
  - Flash toggle → `applyConstraints({ advanced: [{ torch: true }] })` if supported, else toast "Kilat layar aktif".
  - Shutter:
    1. Trigger `cam-flash` white-flash animation.
    2. Draw `<video>` frame to `<canvas>`.
    3. Apply mirror if front camera.
    4. Stamp photo (geo + timestamp bar at bottom).
    5. Convert to JPEG data URL.
    6. Store in `ProspectStore.photos[kind]`.
    7. Stop stream, navigate back to `/prospect/new`.
  - Gallery button → hidden `<input type="file" accept="image/*">`; on file select, read as data URL, stamp, accept.
  - "Gunakan Foto Contoh" → generate placeholder via canvas (gradient + text), stamp, accept.
  - Error states: `NotFoundError` (no camera), permission denied → show `camError` overlay with appropriate message.
- **Lifecycle:** `OnDestroy` stops all tracks to release camera.

#### 8.5.5 `PhotoViewerComponent`
- **Route:** `/prospect/viewer?kind={plang|selfie}&id={prospectId}`
- **Elements:**
  - Full-screen image (`object-fit: contain`).
  - Top bar: close (X), title, spacer.
  - Bottom bar: segmented control (Plang / Selfie), download button, hint text.
- **Behaviour:**
  - Segment switch swaps displayed photo (from `ProspectStore.detail`).
  - Download → `DownloadService.save(blob, filename)`.
  - Close → back to `/prospect/{id}`.

---

### 8.6 Notification Feature

#### `NotificationComponent`
- **Route:** `/notifications`
- **AppBar:** Back (→ `/home`), title "Notifikasi", subtitle "{n} belum dibaca" or "Semua sudah dibaca", "mark all read" icon button.
- **List:**
  - Day groups with label + unread count badge.
  - Each notification: coloured icon (green/orange/blue/red), title, body, timestamp, unread dot.
  - Click → mark as read (removes unread styling + dot, updates badge).
- **Bottom nav** with active state on Notifikasi.

---

### 8.7 Profile Feature

#### `ProfileComponent`
- **Route:** `/profile`
- **Header (gradient + vector):** Back button, large avatar (initials), full name, @username, status badge.
- **Body:**
  - **Info card:** username, full name, referral code (with copy button), email.
  - **Chart card:** "Prospek per Bulan" — custom SVG bar chart (Jan–Sep), total prospek count, legend (completed vs current month). Hoverable bars with tooltip.
  - **Stats card:** prospek bulan ini (+12% badge), kehadiran bulan ini (good badge).
  - **Logout button** (danger style) → opens `ovLogout` modal → "Ya, Keluar" confirms.
  - Version label footer.

---

## 9. Shared / Common Components

| Component | Selector | Props / Inputs | Used in |
|---|---|---|---|
| `ButtonComponent` | `app-button` | `variant: 'primary' \| 'ghost' \| 'danger' \| 'red' \| 'green'`, `disabled`, `icon` (SVG path) | Everywhere |
| `CardComponent` | `app-card` | — | Lists, info panels |
| `BadgeComponent` | `app-badge` | `variant: 'good' \| 'warn' \| 'crit' \| 'info'`, `pip` | Attendance, notifications, profile |
| `IconBtnComponent` | `app-icon-btn` | `icon`, `ariaLabel` | AppBars, modals |
| `InputComponent` | `app-input` | `formControl`, `label`, `icon`, `placeholder`, `required`, `errorText` | Forms |
| `TextareaComponent` | `app-textarea` | `formControl`, `label`, `rows` | Forms |
| `FieldComponent` | `app-field` | `label`, `required`, `invalid` (wraps input/textarea) | Forms |
| `ToastComponent` | `app-toast` | Binds to `UiStore.toast` | Root |
| `ModalComponent` | `app-modal` | `open`, `variant: 'success' \| 'warn'`, `title`, `body`, slots for actions | Success, clock-in, logout |
| `OverlayComponent` | `app-overlay` | `open` | Modal host |
| `PhotoSlotComponent` | `app-photo-slot` | `kind`, `filled`, `image`, `time`, `label` | Check-in form |
| `PhotoCardComponent` | `app-photo-card` | `src`, `caption` | Detail |
| `ProspectItemComponent` | `app-prospect-item` | `prospect`, `(click)` | History lists |
| `NotificationItemComponent` | `app-notif-item` | `notification`, `(read)` | Notification list |
| `AttendRowComponent` | `app-attend-row` | `record` | Attendance history |
| `DayGroupComponent` | `app-day-group` | `label`, `count` | History, notifications |
| `BottomNavComponent` | `app-bottom-nav` | `active: 'home' \| 'notif' \| 'profile'`, `unread` | Home, notifications, profile |
| `AppBarComponent` | `app-appbar` | `title`, `subtitle`, `backRoute?`, `actions?` | All secondary screens |
| `ReferralChipComponent` | `app-referral-chip` | `code`, `(copy)` | Home, profile |
| `MonthlyChartComponent` | `app-monthly-chart` | `data: MonthlyDataPoint[]` | Profile |
| `SvgBgDirective` | `[appSvgBg]` | `variant: 'dotwave' \| 'ringwave'` | Headers, splash |
| `ChromeDirective` | `[appChrome]` | reads route data | Root host