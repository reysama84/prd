# Product Requirements Document — Test Scenarios

**Product:** Sales Point (VisioNet Mini ATM — Field Agent App)
**Scope of this document:** Functional test cases, edge cases, acceptance criteria, and validation steps.
**Authoritative spec source:** The provided interactive mockup (HTML/CSS/JS prototype). The user-requirement text was unreadable; therefore all test scenarios below are derived strictly from the behavior defined in the mockup. Any behavior not present in the mockup is out of scope.

---

## 1. Test Scope & Strategy

### 1.1 In Scope
| Area | Coverage |
|---|---|
| Screen navigation & routing | 13 screens + modal/toast overlays |
| Authentication | Login, logout, session reset |
| Attendance | Clock-in gate, clock-in, clock-out, history, duration |
| Prospect acquisition | Form validation, camera capture, photo stamping, save |
| Prospect history | Search, grouping, load-more, detail view |
| Photo management | Capture, gallery upload, sample fallback, viewer, download |
| Notifications | Grouping, read/unread, mark-all-read |
| Profile | Info, referral copy, monthly chart, stats |
| Shared components | Toast, modal, clipboard, tel: link, status bar |
| Device integration | Camera, torch, GPS coordinates, clipboard, downloads |
| Responsive / a11y | Reduced motion, focus-visible, viewport, safe-area |

### 1.2 Out of Scope
- Backend API contracts (no network layer present)
- Real biometric auth, real GPS hardware behavior (mock uses static coords)
- Push notifications delivery
- Offline persistence (state is in-memory only)
- Localization beyond the Indonesian strings present
- Performance/load testing beyond perceptible UI responsiveness

### 1.3 Test Environment
| Item | Value |
|---|---|
| Device class | Mobile-first, viewport ≤ 412px; desktop framed device shell ≥ 560×700 |
| Browsers | Latest Chrome, Safari (iOS), Firefox, Edge |
| Permissions needed | Camera, Clipboard (write), Downloads |
| Demo credentials | `rizky.pratama` / `salespoint` |
| OS-level settings | Camera permission toggle, screen-rotation lock, dark/light mode |

---

## 2. Global Acceptance Criteria (Apply to Every Screen)

| AC-ID | Criterion |
|---|---|
| G-AC-01 | Only one `.screen` has the `active` class at any time. |
| G-AC-02 | The bottom navigation bar appears only on Home, Notifications, Profile; active tab matches current screen. |
| G-AC-03 | `data-chrome` on `#device` equals `"black"` only when Camera or Photo Viewer is active; otherwise `"navy"`. |
| G-AC-04 | Every interactive element has a visible `:focus-visible` outline (2px accent). |
| G-AC-05 | When `prefers-reduced-motion: reduce` is set, all CSS animations and transitions are effectively instant. |
| G-AC-06 | Toast auto-dismisses after ~2.6 s; concurrent toasts replace the previous text. |
| G-AC-07 | Overlay backdrop click closes only the Logout overlay (Success / Clock-in / sample confirm are not backdrop-dismissible). |
| G-AC-08 | All required-field errors are cleared as soon as the field becomes valid (no stale red border). |
| G-AC-09 | No horizontal scrollbar appears inside any `.scroll` container. |
| G-AC-10 | Status bar time updates at least every 20 s and matches device clock. |

---

## 3. Functional Test Cases by Feature

### 3.1 Splash Screen (`#s-splash`)

| TC-ID | Scenario | Preconditions | Steps | Expected Result | Priority |
|---|---|---|---|---|---|
| SPL-01 | Auto-advance to login | App just loaded | Wait 2.4 s on splash | `#s-login` becomes active | P0 |
| SPL-02 | Spinner visible during splash | App just loaded | Observe splash | Three-dot spinner animates; brand logo, divider, app name, tagline, version foot visible | P1 |
| SPL-03 | Reduced-motion splash | OS reduced-motion on | Load app | Splash visible (animation duration ~0 ms); auto-advance still occurs at 2.4 s | P2 |
| SPL-04 | No interaction leak | Splash visible | Tap anywhere on splash | No navigation occurs; no buttons present | P2 |

### 3.2 Login (`#s-login`)

| TC-ID | Scenario | Preconditions | Steps | Expected Result | Priority |
|---|---|---|---|---|---|
| LGN-01 | Successful login | On login screen | Enter `rizky.pratama` / `salespoint` → tap **Masuk** | Loading screen appears | P0 |
| LGN-02 | Empty username | Login screen | Leave username empty, fill password → submit | `#fUser` invalid; hint "Username wajib diisi." visible; no navigation | P0 |
| LGN-03 | Empty password | Login screen | Fill username, leave password empty → submit | `#fPass` invalid; hint "Kata sandi wajib diisi." visible | P0 |
| LGN-04 | Both fields empty | Login screen | Submit with both empty | Both fields flagged invalid; alert hidden | P0 |
| LGN-05 | Wrong credentials | Login screen | Enter `wrong.user` / `wrongpass` → submit | `#loginAlert` shows "Username atau kata sandi salah…"; no navigation | P0 |
| LGN-06 | Password visibility toggle | Login screen | Tap eye icon | `type` flips between `password`/`text`; icon SVG swaps; aria-label updates | P1 |
| LGN-07 | Forgot password link | Login screen | Tap "Lupa kata sandi?" | Toast "Hubungi supervisor area untuk reset kata sandi." Default link navigation prevented | P1 |
| LGN-08 | Remember-me default state | Login screen load | Observe checkbox | Checkbox pre-checked | P2 |
| LGN-09 | Demo note visible | Login screen | Scroll to bottom | Demo hint with credentials and version visible | P2 |
| LGN-10 | Re-login after logout resets state | Post-logout | Land on splash → login | Login fields contain pre-filled demo values; previous alert hidden | P1 |

### 3.3 Loading (`#s-loading`)

| TC-ID | Scenario | Preconditions | Steps | Expected Result | Priority |
|---|---|---|---|---|---|
| LD-01 | Progress increments through 5 steps | After successful login | Observe progress | Bar fills 0 → 18 → 42 → 63 → 84 → 100 %; status text matches each step | P0 |
| LD-02 | Transition to clock-in gate | Loading at 100 % | Wait ~420 ms | `#s-gate` becomes active | P0 |
| LD-03 | Percentage label sync | Loading in progress | Compare `#loadPct` text vs bar width | Numeric % matches bar width % at each step | P1 |
| LD-04 | Foot note visible | Loading visible | Observe bottom | "Mohon tunggu, jangan tutup aplikasi" visible | P2 |

### 3.4 Clock-In Gate (`#s-gate`)

| TC-ID | Scenario | Preconditions | Steps | Expected Result | Priority |
|---|---|---|---|---|---|
| GATE-01 | Live clock ticking | On gate | Wait ≥ 2 s | `#gateClock` updates `HH:MM:SS` every second | P0 |
| GATE-02 | Ticking stops off-screen | Navigate away then back | Compare | Updates only while gate is active (no leaked intervals causing global updates) | P1 |
| GATE-03 | Greeting & date shown | On gate | Observe | "Halo, Rizky" + current date "Jumat, 4 September 2026" | P1 |
| GATE-04 | Location chip content | On gate | Observe | Branch name, address, accuracy "±8 m" visible | P1 |
| GATE-05 | Shift info card | On gate | Observe | "Reguler · 08:00 – 17:00 WIB" visible | P1 |
| GATE-06 | Clock-in triggers modal | On gate, `attend.in` is null | Tap **Clock In Sekarang** | `attend.in` set to current HH:MM; modal `#ovClockIn` opens with recap; gate button still present | P0 |
| GATE-07 | Modal continue navigates home | Clock-in modal open | Tap **Lanjut ke Beranda** | Modal closes; `#s-home` active | P0 |
| GATE-08 | Status badge pre-clock-in | `attend.in` is null | Render gate | Badge "Belum absen hari ini" | P2 |

### 3.5 Home (`#s-home`)

| TC-ID | Scenario | Preconditions | Steps | Expected Result | Priority |
|---|---|---|---|---|---|
| HM-01 | Prospek count reflects today | `prospekToday` has 4 items | Observe stat card | "4" + "Prospek hari ini" + today's date | P0 |
| HM-02 | Navigate to Prospek Toko | On home | Tap **Prospek Toko** tile | `#s-checkin` active | P0 |
| HM-03 | Navigate to History Prospek | On home | Tap **History Prospek** tile | `#s-history` active | P0 |
| HM-04 | Attendance card — clocked in | `attend.in` set, `attend.out` null | Observe card | Green check icon; text "Sudah clock in · HH:MM WIB"; button red labeled **Clock Out Sekarang** | P0 |
| HM-05 | Attendance card — clocked out | `attend.in` and `attend.out` set | Observe card | Footer hidden; text shows both times | P0 |
| HM-06 | Attendance card — not clocked in | `attend.in` null | Observe card | Pending icon; text "Belum clock in hari ini"; green **Clock In Sekarang** button | P0 |
| HM-07 | Tap attendance card head navigates | On home | Tap card head | `#s-absen` active | P1 |
| HM-08 | Quick clock-out from home | `attend.in` set | Tap **Clock Out Sekarang** on home card | `attend.out` set to HH:MM; toast "Clock out tercatat…"; card updates to hide footer | P0 |
| HM-09 | Referral chip copy | On home | Tap referral chip | Toast "Kode referral SP-RZK2041 disalin." displayed | P1 |
| HM-10 | Avatar button navigates to profile | On home | Tap avatar circle | `#s-profile` active | P1 |
| HM-11 | Bottom nav active state | On home | Observe | "Home" highlighted with underline pip; others inactive | P1 |
| HM-12 | Subtitle updates after new prospect | Save a new prospect, return home | Observe | Stat count incremented; check-in subtitle shows incremented visit index | P0 |

### 3.6 Attendance Detail (`#s-absen`)

| TC-ID | Scenario | Preconditions | Steps | Expected Result | Priority |
|---|---|---|---|---|---|
| ABS-01 | Today's cells reflect state | `attend.in` set, `attend.out` null | Open detail | Clock-in time shown; clock-out "Belum absen" pending style | P0 |
| ABS-02 | Duration label updates | On detail | Observe | "5 jam 34 menit berjalan" (or current text) visible | P1 |
| ABS-03 | Method label | On detail | Observe | "GPS + Selfie · akurasi ±8 m" | P1 |
| ABS-04 | History list renders | On detail | Observe | 5 history rows (Sep 03, 02, 01; Ags 29, 28) visible with date, duration, in→out | P0 |
| ABS-05 | Load older attendance | On detail | Tap **Muat riwayat bulan lalu** | 3 more rows appended (Ags 27/26/25); button disabled and label changes to "Riwayat Agustus 2026 sudah dimuat"; toast shown | P0 |
| ABS-06 | Load older disabled after exhausted | After ABS-05 | Tap button again | Button disabled; no-op | P1 |
| ABS-07 | Clock-out from detail | `attend.in` set, `attend.out` null | Tap **Clock Out** | `attend.out` set; button disabled with text "Sudah clock out pukul HH:MM WIB"; toast shown | P0 |
| ABS-08 | Back button returns home | On detail | Tap top-left back | `#s-home` active | P1 |
| ABS-09 | Izin row rendering | On detail | Observe Ags 28 row | "Izin sakit" duration, "Izin" status, em-dashes for in/out | P2 |

### 3.7 History Prospek (`#s-history`)

| TC-ID | Scenario | Preconditions | Steps | Expected Result | Priority |
|---|---|---|---|---|---|
| HST-01 | Today's group renders | On history | Observe | "Hari Ini — Jumat, 4 September 2026" group with count 4 and 4 prospect rows | P0 |
| HST-02 | Search filters by name | On history | Type "berkah" | Only matching rows remain in today's group; subtitle "1 hasil untuk 'berkah'" | P0 |
| HST-03 | Search filters by address | On history | Type "fatmawati" | Matching row visible; others hidden | P0 |
| HST-04 | Search filters by PIC | On history | Type "sri wahyuni" | Matching row visible | P0 |
| HST-05 | Clear-search button appears | On history with query | Observe | `#histClear` becomes visible (`.has-q` on search bar) | P1 |
| HST-06 | Clear-search resets | On history with query | Tap clear button | Input emptied; full list restored; focus returns to input | P1 |
| HST-07 | Empty-search state | On history | Type "zzz-nonexistent" | Empty-state block with search icon and "Tidak ada prospek yang cocok" message | P0 |
| HST-08 | Load older data | On history | Tap **Muat data sebelumnya** | New day group appended (Kamis 3 Sep, 6 items); toast shown | P0 |
| HST-09 | Load older exhausted | After all groups loaded | Tap button again | Button disabled; label "Semua data 7 hari terakhir sudah dimuat"; opacity reduced | P1 |
| HST-10 | Tap row opens detail | On history | Tap a prospect row | `#s-detail` active; subtitle = toko name | P0 |
| HST-11 | Subtitle count on first load | On history (no search) | Observe | Subtitle "4 prospek hari ini" | P1 |
| HST-12 | Search persists across load-more | Search "toko", then load older | Observe | Only matching items from new groups render | P2 |

### 3.8 Check-In Form (`#s-checkin`)

| TC-ID | Scenario | Preconditions | Steps | Expected Result | Priority |
|---|---|---|---|---|---|
| CK-01 | Subtitle shows next visit index | `prospekToday.length` = 4 | Open form | Subtitle "Kunjungan ke-5 hari ini · HH:MM WIB" | P1 |
| CK-02 | All required empty on save | Fresh form | Tap **Simpan Prospek** | All five fields flagged invalid (Toko, Alamat, Pic, Telp, Foto); toast "Lengkapi data bertanda *…"; scroll to first invalid | P0 |
| CK-03 | Phone < 9 digits rejected | Fill toko/alamat/pic, telp = "0812" | Submit | `#fTelp` invalid; hint visible | P0 |
| CK-04 | Phone exactly 9 digits accepted | Fill all, telp = "081234567" | Submit | Phone field considered valid | P1 |
| CK-05 | Photos missing → blocked | Fill text fields, no photos | Submit | `#fFoto` invalid; hint "Kedua foto wajib diambil…" | P0 |
| CK-06 | Only one photo → blocked | Take only plang | Submit | `#fFoto` invalid | P0 |
| CK-07 | Successful save | Fill all validly, both photos | Submit | Modal `#ovSuccess` opens with recap of toko, PIC, telp, time, "2 foto terlampir"; new item pushed to `prospekToday` | P0 |
| CK-08 | Recap content correctness | After save | Observe modal recap | All entered values reflected accurately | P1 |
| CK-09 | "Selesai" closes modal and resets | Success modal open | Tap **Selesai** | Modal closes; form reset; navigates to Home; stat count incremented | P0 |
| CK-10 | "Check In Lagi" resets form | Success modal open | Tap **Check In Lagi** | Modal closes; form fields cleared; photos cleared; scroll to top; stays on `#s-checkin` | P0 |
| CK-11 | Photo slot filled state | Take a photo | Observe slot | `.filled` class added; `<img>` present; `.ph-tag` shows label + HH:MM | P1 |
| CK-12 | Retake photo | Slot already filled | Tap slot | Camera opens; new photo replaces old | P1 |
| CK-13 | Catatan optional | Leave catatan empty | Save | Save still succeeds | P1 |
| CK-14 | Location chip shown | Open form | Observe | Coordinates, street, accuracy "±6 m" visible | P2 |
| CK-15 | Back returns home | On form | Tap back | `#s-home` active | P1 |
| CK-16 | Invalid error cleared on input | Field flagged invalid | Type valid value | Red border removed; hint hidden | P1 |

### 3.9 Camera (`#s-camera`)

| TC-ID | Scenario | Preconditions | Steps | Expected Result | Priority |
|---|---|---|---|---|---|
| CAM-01 | Open camera for plang | On form | Tap plang slot | `#s-camera` active; title "Foto Plang Toko"; hint about positioning papan nama | P0 |
| CAM-02 | Open camera for selfie | On form | Tap selfie slot | Title "Selfie dengan PIC Toko"; video has `.mirror` class (front cam) | P0 |
| CAM-03 | Capture photo | Camera streaming | Tap shutter | Flash animation; canvas draws video frame; stamp applied; returns to form with photo set | P0 |
| CAM-04 | Photo stamp content | After capture | Inspect generated image | Bottom overlay with orange accent bar; date/time "DD/MM/YYYY HH:MM WIB"; coords "−6.26412, 106.79931 · Jl. Bangka Raya, Jakarta Selatan" | P0 |
| CAM-05 | Front-camera mirror on capture | Selfie mode | Capture | Output image is non-mirrored (mirror transform only on preview) | P1 |
| CAM-06 | Flip camera | Camera open | Tap flip button | `camFacing` toggles; toast "Kamera depan/belakang aktif."; stream restarts | P1 |
| CAM-07 | Torch toggle on supported device | Device supports torch | Tap flash button | Button gets `.on`; torch applied via `applyConstraints`; toast "Lampu kilat menyala/dimatikan." | P1 |
| CAM-08 | Torch toggle on unsupported device | Device lacks torch | Tap flash button | Button toggles `.on`; toast "Kilat layar aktif (perangkat tanpa lampu kilat)." | P2 |
| CAM-09 | Camera permission denied | Permission blocked | Open camera | `#camError` shown with message about permission; "Gunakan Foto Contoh" button available | P0 |
| CAM-10 | No camera device | `NotFoundError` | Open camera | Error message "Tidak ada kamera yang terdeteksi…"; sample button available | P0 |
| CAM-11 | Browser without getUserMedia | Old browser | Open camera | camFail with "tidak mendukung akses kamera langsung" | P1 |
| CAM-12 | Use sample photo | Error visible | Tap **Gunakan Foto Contoh** | Sample gradient image generated with kind label and stamp; slot filled; returns to form | P0 |
| CAM-13 | Capture while error visible | Error visible | Tap shutter | Triggers `makeSample()` (no crash) | P2 |
| CAM-14 | Capture before stream ready | Camera just opened, no `videoWidth` yet | Tap shutter quickly | Toast "Kamera belum siap, coba sesaat lagi." | P1 |
| CAM-15 | Gallery upload | On camera screen | Tap gallery button | File picker opens (image/*); selected image accepted as photo; slot filled | P1 |
| CAM-16 | Close camera without capture | Camera open | Tap X | Stream stopped (`camStream` null, srcObject null); returns to `#s-checkin`; no photo set | P0 |
| CAM-17 | Stream stopped on navigation away | Camera open | Navigate via back/other | `stopStream()` invoked; tracks stopped | P0 |
| CAM-18 | Chrome attribute switches to black | Camera active | Observe device | `data-chrome="black"` | P1 |
| CAM-19 | Toast confirmation after accept | Capture completes | Observe | Toast "Foto plang tersimpan." or "Foto selfie tersimpan." | P2 |

### 3.10 Detail Prospek (`#s-detail`)

| TC-ID | Scenario | Preconditions | Steps | Expected Result | Priority |
|---|---|---|---|---|---|
| DT-01 | Subtitle = toko name | Open any detail | Observe | `#detailSub` shows toko name | P1 |
| DT-02 | Both photos render | Open detail | Observe | Plang and selfie images displayed in photo cards | P0 |
| DT-03 | Photo generated for existing prospect | Open older prospect without stored photos | Observe | `makePhoto()` generates plang and selfie on demand | P1 |
| DT-04 | Plang photo visual content | Inspect plang image | Observe | Sky gradient, store front, papan nama with toko name, awning, door, etalase, timestamp | P2 |
| DT-05 | Selfie photo visual content | Inspect selfie image | Observe | Two figures, gradient bg, "Sales Point · {PIC}" label | P2 |
| DT-06 | Data rows completeness | Open detail | Observe | Rows: Nama Toko, Alamat, PIC, No. Telp, Waktu Kunjungan, Titik Koordinat | P0 |
| DT-07 | Coordinate uniqueness per toko | Open two different details | Compare | Coordinates differ (deterministic hash of name) | P2 |
| DT-08 | Note display | Open detail | Observe | Note text shown; fallback "Tidak ada catatan tambahan…" when empty | P1 |
| DT-09 | Download both photos | On detail | Tap **Unduh Kedua Foto** | Two downloads triggered sequentially (plang then selfie) with delay; toasts appear | P0 |
| DT-10 | Download single from viewer | On photo viewer | Tap **Unduh Foto Ini** | One download of current segment | P0 |
| DT-11 | Call PIC | On detail | Tap **Hubungi PIC** | `tel:` link invoked; toast "Menghubungi {PIC} · {telp}" | P1 |
| DT-12 | Tap photo opens viewer | On detail | Tap a photo card | `#s-photo` active; image loaded | P0 |
| DT-13 | Back returns to history | On detail | Tap back | `#s-history` active | P1 |
| DT-14 | File name format | Download a photo | Inspect file name | `Prospek_{TokoName}_{plang|selfie}.jpg` (non-alphanumerics collapsed to underscores) | P1 |

### 3.11 Photo Viewer (`#s-photo`)

| TC-ID | Scenario | Preconditions | Steps | Expected Result | Priority |
|---|---|---|---|---|---|
| PV-01 | Plang segment shown by default | Open viewer | Observe | `#segPlang` has `.on`; title "Foto Plang Toko" | P1 |
| PV-02 | Switch to selfie | On viewer | Tap selfie segment | Image swaps; `#segSelfie` `.on`; title updates | P0 |
| PV-03 | Switch back to plang | Selfie segment active | Tap plang segment | Image swaps back | P0 |
| PV-04 | Close returns to detail | On viewer | Tap X | `#s-detail` active | P0 |
| PV-05 | Download current | On viewer | Tap **Unduh Foto Ini** | Current segment downloaded | P0 |
| PV-06 | Hint about long-press | On viewer | Observe bottom | "Di ponsel, tekan lama pada foto untuk menyimpannya ke galeri." visible | P2 |
| PV-07 | Chrome attribute black | On viewer | Observe device | `data-chrome="black"` | P1 |

### 3.12 Notifications (`#s-notif`)

| TC-ID | Scenario | Preconditions | Steps | Expected Result | Priority |
|---|---|---|---|---|---|
| NTF-01 | Groups render | Open notifications | Observe | 3 groups: "Hari Ini", "Kemarin", "Rabu, 2 September 2026" with unread counts | P0 |
| NTF-02 | Initial unread count | First load | Observe badges | 3 unread across all nav badges; subtitle "3 belum dibaca" | P0 |
| NTF-03 | Tap unread marks read | Open notif | Tap an unread item | It loses unread styling; badge counts decrement | P0 |
| NTF-04 | Mark all read | Open notif | Tap top-right check-double icon | All items lose unread; badges hidden; subtitle "Semua sudah dibaca"; toast shown | P0 |
| NTF-05 | Read item no-op | Already-read item | Tap again | No state change | P1 |
| NTF-06 | Unread visual cues | Open notif | Observe unread items | Left border accent + blue dot | P1 |
| NTF-07 | Bottom nav badge sync | Mark all read | Observe all 3 nav badges | Badges hide (display:none) | P0 |
| NTF-08 | Group label count | Open notif | Observe | "Hari Ini" shows "3 baru"; "Kemarin" no badge | P2 |
| NTF-09 | Back returns home | On notif | Tap back | `#s-home` active | P1 |
| NTF-10 | Bottom nav active state | On notif | Observe | "Notifikasi" highlighted | P1 |

### 3.13 Profile (`#s-profile`)

| TC-ID | Scenario | Preconditions | Steps | Expected Result | Priority |
|---|---|---|---|---|---|
| PRF-01 | Avatar initials | Open profile | Observe | "RP" displayed in large avatar | P2 |
| PRF-02 | Info rows complete | Open profile | Observe | Username, Nama Lengkap, Kode Referral, Email visible | P0 |
| PRF-03 | Referral copy button | Open profile | Tap copy icon next to code | Toast "Kode referral SP-RZK2041 disalin." | P0 |
| PRF-04 | Chart renders 9 bars | Open profile | Observe chart | Bars Jan–Sep; Aug tallest (104) with value label; Sep is orange (current) with label | P0 |
| PRF-05 | Chart tooltip on hover | Desktop | Hover a bar | Tooltip with month/year + count + "berjalan" indicator for Sep | P1 |
| PRF-06 | Chart tooltip on tap (mobile) | Mobile | Tap a bar | Tooltip shows; hides on leave | P1 |
| PRF-07 | Chart legend | Open profile | Observe | "Bulan selesai" blue, "Bulan berjalan" orange | P2 |
| PRF-08 | Total prospek label | Open profile | Observe | "677 total prospek" | P1 |
| PRF-09 | Monthly prospek stat | Open profile | Observe | "38 toko" with "+12%" green badge | P2 |
| PRF-10 | Attendance stat | Open profile | Observe | "4 hari · 100%" with "Baik" badge | P2 |
| PRF-11 | Logout opens confirm modal | Open profile | Tap **Keluar** | `#ovLogout` modal opens with warning about pending clock-out | P0 |
| PRF-12 | Logout cancel | Logout modal open | Tap **Batal** | Modal closes; remains on profile | P0 |
| PRF-13 | Logout confirm | Logout modal open | Tap **Ya, Keluar** | Modal closes; camera stopped; form reset; `attend` reset; password reset to "salespoint"; splash → login flow | P0 |
| PRF-14 | Version label | Open profile | Observe bottom | "Sales Point v1.0.0 / Build 2026.09.04 · VisioNet Mini ATM / © 2026 PT Visionet Data Internasional" | P2 |
| PRF-15 | Back returns home | On profile | Tap top-left back | `#s-home` active | P1 |
| PRF-16 | Bottom nav active | On profile | Observe | "Profil" highlighted | P1 |

### 3.14 Shared Components

| TC-ID | Scenario | Preconditions | Steps | Expected Result | Priority |
|---|---|---|---|---|---|
| SH-01 | Toast text replaces | Trigger two toasts quickly | Observe | Second toast text replaces first; same toast element re-shown | P1 |
| SH-02 | Toast auto-dismiss | Trigger toast | Wait 2.6 s | Toast hidden | P1 |
| SH-03 | Clipboard fallback | `navigator.clipboard` unavailable | Trigger referral copy | Fallback textarea + execCommand path; toast still shown | P1 |
| SH-04 | Clipboard ultimate fallback | All clipboard methods fail | Trigger copy | Toast "Kode referral Anda: SP-RZK2041" (no crash) | P2 |
| SH-05 | Downloads API — declined | `window.claude.use('downloads')` rejects with `declined` | Download photo | No toast shown; silent | P1 |
| SH-06 | Downloads API — rate limited | Rejects with `rate_limited` | Download photo | Toast "Tunggu sebentar, lalu coba unduh lagi." | P1 |
| SH-07 | Downloads API — other error | Rejects other code | Download photo | Toast "Unduhan tidak tersedia. Tekan lama pada foto…" | P1 |
| SH-08 | Anchor fallback download | No downloads API | Download photo | Anchor with `download` attribute clicked; toast "Foto diunduh: {name}" | P1 |
| SH-09 | Anchor fallback blocked | Anchor click throws | Download photo | Toast "Unduhan diblokir browser…" | P2 |
| SH-10 | Tel link construction | Tap **Hubungi PIC** | Inspect | `window.location.href` set to `tel:{cleaned-number}` | P1 |
| SH-11 | Backdrop closes logout only | Success modal open | Tap backdrop | Modal stays open | P1 |
| SH-12 | Backdrop closes logout | Logout modal open | Tap backdrop | Modal closes | P1 |
| SH-13 | Status bar time accuracy | Any screen | Compare `#sbTime` with device clock | Matches HH:MM | P2 |
| SH-14 | Status bar updates every 20 s | Any screen | Wait | Updates without page interaction | P2 |
| SH-15 | Reduced motion disables animations | OS reduced-motion | Observe any screen | Animations effectively instant | P1 |
| SH-16 | Focus-visible outline | Keyboard tab | Tab through elements | 2px accent outline visible on focused element | P1 |
| SH-17 | Safe-area insets respected | Notched device | Observe bottom bars | Bottom padding includes `env(safe-area-inset-bottom)` | P2 |

---

## 4. Edge Cases & Negative Tests

| EC-ID | Scenario | Steps | Expected Behavior |
|---|---|---|---|
| EC-01 | Login with whitespace-only username | Type spaces in username, submit | Trimmed to empty; flagged invalid |
| EC-02 | Login with trailing spaces in username | "rizky.pratama " | Trimmed; accepted if valid after trim |
| EC-03 | Phone with non-digits | Type "08ab12cd34" | `.replace(/\D/g,"")` strips to "081234"; if < 9 digits, rejected |
| EC-04 | Phone with separators | "0812-3456-7890" | Sanitized to "081234567890"; accepted if ≥ 9 |
| EC-05 | Very long toko name | Enter 500 chars | Field accepts; no length cap shown — verify no layout breakage |
| EC-06 | Special characters in toko name | Enter "Toko <script>" | Escaped in detail view; no HTML injection |
| EC-07 | Multiline alamat with newlines | Type multiline | Rendered with line breaks in detail card |
| EC-08 | Emoji in catatan | Type emoji | Rendered correctly in detail note |
| EC-09 | Save prospect with both photos identical file | Reuse same file | Both slots filled; save proceeds |
| EC-10 | Rapid double-tap on shutter | Tap shutter twice fast | Only one photo accepted (second tap may show "Kamera belum siap") |
| EC-11 | Rapid double-submit on login | Tap Masuk twice fast | No duplicate loading screens |
| EC-12 | Rapid double-tap on Save | Tap Simpan twice fast | Only one item appended; only one modal |
| EC-13 | Logout before clock-in | `attend.in` null, open logout modal | Modal warning shows "Anda belum melakukan clock out" (text generic; behavior correct) |
| EC-14 | Logout after clock-out | `attend.out` set | Logout still shows same modal (text not conditional — acceptable per mockup) |
| EC-15 | Search with leading/trailing spaces | Type "  toko  " | Trimmed; "toko" used as filter |
| EC-16 | Search case-insensitivity | Type "TOKO" vs "toko" | Same results |
| EC-17 | Load older when search active | Search "toko", load older | Only matching items from older groups appear |
| EC-18 | Camera stream lost mid-session | Stop tracks externally | Shutter returns "Kamera belum siap" toast |
| EC-19 | Photo slot tapped after logout/reset | Form reset clears photos | Slot returns to empty state |
| EC-20 | Notifications all marked read then re-opened | Mark all read, leave, return | All remain read; badge stays hidden |
| EC-21 | Navigation away from gate pauses clock | Leave gate, return after 30 s | Gate clock resumes; no backlog of updates |
| EC-22 | Detail opened for prospect without note | Prospect with empty note | Fallback "Tidak ada catatan tambahan…" shown |
| EC-23 | Detail opened for older prospect without telp | Older item | Deterministic fake telp generated |
| EC-24 | Switch device orientation mid-camera | Rotate device | Stream continues; mirror transform preserved for selfie |
| EC-25 | Backgrounded app during loading | Switch tabs during loading | On return, transition may have completed