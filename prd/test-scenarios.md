# Product Requirements Document (PRD) — Test Scenarios
## Sales Point — VisioNet Mini ATM Agent Acquisition App

**Document Scope:** This PRD covers **test scenarios only** — functional test cases, edge cases, acceptance criteria, and validation steps — derived from the provided mockup and application behavior.

**App Overview:** A mobile-first field sales application for VisioNet Mini ATM agents to manage attendance (clock in/out via GPS), register prospect stores with photo documentation, view prospect history, receive notifications, and track performance metrics.

---

## 1. Splash Screen & App Initialization

### 1.1 Functional Test Cases

| TC ID | Scenario | Preconditions | Steps | Expected Result |
|-------|----------|---------------|-------|-----------------|
| SPL-001 | Splash screen displays on app launch | App not running | 1. Open app | Splash screen visible with logo, app name "SALES POINT", tagline, spinner animation, and version "v1.0.0" |
| SPL-002 | Auto-transition from splash to login | App at splash | 1. Wait ~2.4 seconds | Login screen appears automatically |
| SPL-003 | Spinner animation is visible | App at splash | 1. Observe splash screen | Three-dot bounce animation visible below tagline |
| SPL-004 | Brand logo loads correctly | App at splash | 1. Observe splash screen | VisioNet Mini ATM logo image renders (not broken) |

### 1.2 Edge Cases

| TC ID | Scenario | Steps | Expected Result |
|-------|----------|-------|-----------------|
| SPL-E01 | App reopened during splash | 1. Open app 2. Close immediately 3. Reopen | Splash displays fresh, timer restarts |
| SPL-E02 | Device in dark mode | 1. Set device to dark mode 2. Open app | Splash background remains purple (#2a1666) regardless of system theme |
| SPL-E03 | Reduced motion preference | 1. Enable "reduce motion" in OS 2. Open app | Rise animation disabled/instant; splash still transitions to login |

### 1.3 Acceptance Criteria
- ✅ Splash screen must render within 1 second of app launch
- ✅ Auto-transition to login must occur after 2.0–2.8 seconds
- ✅ All visual elements (logo, name, tagline, spinner, version) must be visible

---

## 2. Login

### 2.1 Functional Test Cases

| TC ID | Scenario | Preconditions | Steps | Expected Result |
|-------|----------|---------------|-------|-----------------|
| LOG-001 | Successful login with valid credentials | App at login screen | 1. Enter username `rizky.pratama` 2. Enter password `salespoint` 3. Tap "Masuk" | Loading screen appears with progress bar |
| LOG-002 | Login with empty username | App at login screen | 1. Clear username field 2. Enter password 3. Tap "Masuk" | Username field shows red error border; "Username wajib diisi." hint appears; no navigation |
| LOG-003 | Login with empty password | App at login screen | 1. Enter username 2. Clear password 3. Tap "Masuk" | Password field shows red error border; "Kata sandi wajib diisi." hint appears |
| LOG-004 | Login with both fields empty | App at login screen | 1. Clear both fields 2. Tap "Masuk" | Both fields show red border + error hints; no navigation |
| LOG-005 | Login with incorrect credentials | App at login screen | 1. Enter `wrong.user` 2. Enter `wrongpass` 3. Tap "Masuk" | Red alert banner appears: "Username atau kata sandi salah." |
| LOG-006 | Password visibility toggle | App at login screen | 1. Type password 2. Tap eye icon | Password characters become visible; icon changes to eye-off; tap again to hide |
| LOG-007 | "Ingat saya" checkbox default state | App at login screen | 1. Observe checkbox | Checkbox is checked by default |
| LOG-008 | "Lupa kata sandi?" link | App at login screen | 1. Tap "Lupa kata sandi?" link | Toast appears: "Hubungi supervisor area untuk reset kata sandi." |
| LOG-009 | Demo credentials prefilled | App at login screen | 1. Observe username field | Username field prefilled with `rizky.pratama`; password prefilled with `salespoint` |
| LOG-010 | Login button arrow icon | App at login screen | 1. Observe login button | Button displays "Masuk" text with right-arrow icon |

### 2.2 Edge Cases

| TC ID | Scenario | Steps | Expected Result |
|-------|----------|-------|-----------------|
| LOG-E01 | Username with leading/trailing spaces | 1. Enter ` rizky.pratama ` 2. Enter `salespoint` 3. Submit | Login succeeds (whitespace trimmed) |
| LOG-E02 | Case-sensitive username | 1. Enter `RIZKY.PRATAMA` 2. Enter `salespoint` 3. Submit | Login fails with error banner |
| LOG-E03 | Case-sensitive password | 1. Enter `rizky.pratama` 2. Enter `SALESPOINT` 3. Submit | Login fails with error banner |
| LOG-E04 | Rapid double-tap on login button | 1. Enter valid creds 2. Tap "Masuk" rapidly twice | Only one loading sequence starts; no duplicate transitions |
| LOG-E05 | Form submit via Enter key | 1. Fill valid credentials 2. Press Enter on keyboard | Form submits; loading begins |

### 2.3 Acceptance Criteria
- ✅ Valid credentials (`rizky.pratama` / `salespoint`) must always succeed
- ✅ Invalid credentials must show inline red alert, not a browser dialog
- ✅ Empty field validation must be client-side and field-specific
- ✅ Password toggle must not reset the typed value
- ✅ Error alert must be dismissible on next valid attempt

---

## 3. Loading Screen

### 3.1 Functional Test Cases

| TC ID | Scenario | Preconditions | Steps | Expected Result |
|-------|----------|---------------|-------|-----------------|
| LDR-001 | Progress bar advances through stages | Loading screen active | 1. Observe progress | Progress fills 0% → 18% → 42% → 63% → 84% → 100% |
| LDR-002 | Status text updates per stage | Loading screen active | 1. Observe status text | Text changes: "Menghubungkan ke server…" → "Memverifikasi kredensial agen…" → "Memuat profil & area kerja…" → "Sinkronisasi data absensi…" → "Menarik data prospek terbaru…" → "Siap digunakan" |
| LDR-003 | Percentage counter displays | Loading screen active | 1. Observe percentage | Large % number matches progress bar width |
| LDR-004 | Auto-transition to clock-in gate | Loading screen at 100% | 1. Wait ~420ms after 100% | Clock-in gate screen appears |
| LDR-005 | Footer message visible | Loading screen active | 1. Observe footer | "Mohon tunggu, jangan tutup aplikasi" displayed |

### 3.2 Edge Cases

| TC ID | Scenario | Steps | Expected Result |
|-------|----------|-------|-----------------|
| LDR-E01 | Loading interrupted by app backgrounding | 1. Trigger loading 2. Background app 3. Return | Loading continues or completes; no frozen state |

### 3.3 Acceptance Criteria
- ✅ Progress bar must animate smoothly (not jump) between stages
- ✅ Each stage must take 430–690ms (430 + random 0–260)
- ✅ Total loading time must be ~2.5–3.5 seconds
- ✅ Transition to gate screen must only occur after 100%

---

## 4. Clock-In Gate (Attendance)

### 4.1 Functional Test Cases

| TC ID | Scenario | Preconditions | Steps | Expected Result |
|-------|----------|---------------|-------|-----------------|
| GTE-001 | Gate screen displays user greeting | User logged in, not clocked in | 1. Arrive at gate screen | "Halo, Rizky" and current date displayed |
| GTE-002 | Live clock updates | App at gate screen | 1. Observe clock for 2+ seconds | Clock updates every second (HH:MM:SS format) |
| GTE-003 | "Belum absen" badge shows | User not clocked in | 1. Observe gate screen | Info badge: "Belum absen hari ini" |
| GTE-004 | Location chip displays branch | App at gate screen | 1. Observe location chip | "Kantor Cabang Jakarta Selatan" with address and accuracy "±8 m" |
| GTE-005 | Shift info card shows schedule | App at gate screen | 1. Observe shift card | "Reguler · 08:00 – 17:00 WIB" |
| GTE-006 | Clock In button functions | User not clocked in | 1. Tap "Clock In Sekarang" | Clock-in success modal appears with time and location recap |
| GTE-007 | Modal confirms clock-in | Clock-in modal open | 1. Tap "Lanjut ke Beranda" | Home screen appears; attendance card shows clock-in time |
| GTE-008 | Helper text below button | App at gate screen | 1. Observe below button | "Absen wajib dilakukan sebelum memulai kunjungan prospek." visible |

### 4.2 Edge Cases

| TC ID | Scenario | Steps | Expected Result |
|-------|----------|-------|-----------------|
| GTE-E01 | Clock-in after already clocked in | 1. Clock in 2. Navigate back to gate | Gate screen not shown again (home is entry after clock-in) |
| GTE-E02 | Clock at midnight (00:00) | 1. Set device time to 23:59:50 2. Wait 10s 3. Observe clock | Clock transitions correctly to 00:00:00 |

### 4.3 Acceptance Criteria
- ✅ Clock must update in real-time every second
- ✅ GPS/location info must show branch name, address, and accuracy
- ✅ Clock-in must record current HH:MM and display in success modal
- ✅ After clock-in, home screen attendance card must reflect "Sudah clock in · HH:MM WIB"

---

## 5. Home / Dashboard

### 5.1 Functional Test Cases

| TC ID | Scenario | Preconditions | Steps | Expected Result |
|-------|----------|---------------|-------|-----------------|
| HOM-001 | Home header displays user info | User clocked in, on home | 1. Observe home header | "Welcome back, Rizky Pratama" with logo and profile avatar button |
| HOM-002 | Referral chip displays code | On home screen | 1. Observe referral chip | "Kode Referral: SP-RZK2041" with copy icon |
| HOM-003 | Referral code copy | On home screen | 1. Tap referral chip | Toast: "Kode referral SP-RZK2041 disalin." |
| HOM-004 | Today's prospect count displays | On home screen | 1. Observe stat card | Large number shows count of today's prospects (e.g., "4") |
| HOM-005 | Current date displays | On home screen | 1. Observe stat card subtitle | Date shown: "Jumat, 4 September 2026" |
| HOM-006 | Menu tile: Prospek Toko | On home screen | 1. Tap "Prospek Toko" tile | Check-in form screen appears |
| HOM-007 | Menu tile: History Prospek | On home screen | 1. Tap "History Prospek" tile | History screen appears with prospect list |
| HOM-008 | Attendance card shows clock-in status | User clocked in | 1. Observe attendance card | Shows "Sudah clock in · HH:MM WIB" with green checkmark icon |
| HOM-009 | Clock Out button on home | User clocked in, not out | 1. Tap "Clock Out Sekarang" button | Clock-out recorded; toast: "Clock out tercatat pukul HH:MM WIB." |
| HOM-010 | Attendance card after clock-out | User clocked out | 1. Observe attendance card | Foot/button section hidden; card shows both in and out times |
| HOM-011 | Attendance card navigates to detail | On home screen | 1. Tap attendance card header | Absen/attendance detail screen appears |
| HOM-012 | Bottom navigation: Home active | On home screen | 1. Observe bottom nav | "Home" tab highlighted with accent color and top bar indicator |
| HOM-013 | Bottom navigation: Notifikasi | On home screen | 1. Tap "Notifikasi" tab | Notifications screen appears with unread badge |
| HOM-014 | Bottom navigation: Profil | On home screen | 1. Tap "Profil" tab | Profile screen appears |
| HOM-015 | Profile avatar button | On home screen | 1. Tap profile avatar button (top right) | Profile screen appears |

### 5.2 Edge Cases

| TC ID | Scenario | Steps | Expected Result |
|-------|----------|-------|-----------------|
| HOM-E01 | Clock out before clock in | 1. Fresh login 2. Skip gate (not possible in flow) — verify attendance card when `attend.in` is null | "Clock In Sekarang" button shown (green); clicking triggers clock-in |
| HOM-E02 | Notification badge count | 1. Check home nav badge | Badge shows "3" (unread count); badge hidden if 0 |
| HOM-E03 | Prospect count updates after adding | 1. Note today's count 2. Add new prospect 3. Return to home | Count incremented by 1 |
| HOM-E04 | Check-in subtitle updates | 1. Note "Kunjungan ke-N" 2. Add prospect 3. Return to home, open check-in | Subtitle shows "Kunjungan ke-(N+1)" |

### 5.3 Acceptance Criteria
- ✅ Home must display: welcome message, referral chip, today's prospect count, 2 menu tiles, attendance card, bottom nav
- ✅ Referral chip must copy code to clipboard on tap
- ✅ Clock-out button must only appear when clocked in but not yet clocked out
- ✅ After clock-out, attendance card foot section must be hidden
- ✅ Navigation between all 3 tabs must work bidirectionally

---

## 6. Attendance Detail (Absen)

### 6.1 Functional Test Cases

| TC ID | Scenario | Preconditions | Steps | Expected Result |
|-------|----------|---------------|-------|-----------------|
| ABS-001 | Today's attendance card displays | On absen screen | 1. Observe today's card | Date, shift, clock-in time, clock-out status shown |
| ABS-002 | Clock In time shown | User clocked in | 1. Observe "Clock In" cell | Time displayed (e.g., "07:48") |
| ABS-003 | Clock Out status pending | User not clocked out | 1. Observe "Clock Out" cell | "Belum absen" in muted style |
| ABS-004 | Location metadata shows | On absen screen | 1. Observe metadata section | Location address, duration running, method "GPS + Selfie · ±8 m" |
| ABS-005 | Clock Out button works | User not clocked out | 1. Tap "Clock Out" button | Clock-out recorded; button disabled and shows "Sudah clock out pukul HH:MM WIB" |
| ABS-006 | Back button navigates home | On absen screen | 1. Tap back arrow | Home screen appears |
| ABS-007 | Attendance history list renders | On absen screen | 1. Scroll to "Riwayat Absen" | Previous days' records shown with date, duration, in/out times |
| ABS-008 | "Muat riwayat bulan lalu" | On absen screen | 1. Tap "Muat riwayat bulan lalu" button | 3 older August records appended; button disabled; toast confirmation |
| ABS-009 | History row format | On absen screen | 1. Observe a history row | Shows: date (DD Mon), day abbreviation, duration, in→out times |

### 6.2 Edge Cases

| TC ID | Scenario | Steps | Expected Result |
|-------|----------|-------|-----------------|
| ABS-E01 | Clock-out button after already clocked out | 1. Clock out 2. Tap "Clock Out" again | Button is disabled; no action |
| ABS-E02 | Attendance record with "Izin" status | 1. Observe August 28 row | Shows "Izin sakit" duration and "Izin" badge style (info/blue) |

### 6.3 Acceptance Criteria
- ✅ Today's attendance card must show correct clock-in/out status
- ✅ Clock-out button must be disabled after use
- ✅ History list must show date, day, duration, and in/out times for each entry
- ✅ "Muat riwayat bulan lalu" must append data and disable after all loaded
- ✅ Back button must return to home screen

---

## 7. Prospek Toko (Check-In Form)

### 7.1 Functional Test Cases

| TC ID | Scenario | Preconditions | Steps | Expected Result |
|-------|----------|---------------|-------|-----------------|
| CHK-001 | Form fields render | On check-in form | 1. Observe form | Fields: Nama Toko, Alamat Toko, Nama PIC, No. Telp PIC, Dokumentasi Foto (2 slots), Catatan Kunjungan (optional) |
| CHK-002 | Location chip auto-detects | On check-in form | 1. Observe location chip | Shows coordinates, address, accuracy "±6 m" |
| CHK-003 | Visit counter in header | On check-in form | 1. Observe header subtitle | "Kunjungan ke-N hari ini · HH:MM WIB" |
| CHK-004 | Photo slot: Plang opens camera | On check-in form | 1. Tap "Foto Plang" slot | Camera screen opens; title "Foto Plang Toko" |
| CHK-005 | Photo slot: Selfie opens camera | On check-in form | 1. Tap "Selfie dengan PIC" slot | Camera screen opens; title "Selfie dengan PIC Toko" |
| CHK-006 | Form validation: all empty | On check-in form | 1. Tap "Simpan Prospek" without filling | All required fields show red border + error hints; toast: "Lengkapi data bertanda * sebelum menyimpan." |
| CHK-007 | Form validation: missing name | On check-in form | 1. Fill all except Nama Toko 2. Tap Save | Only Nama Toko field shows error |
| CHK-008 | Form validation: phone < 9 digits | On check-in form | 1. Fill all fields 2. Enter phone "12345" 3. Tap Save | Phone field shows error: "Nomor telepon wajib diisi (minimal 9 digit)." |
| CHK-009 | Form validation: missing photos | On check-in form | 1. Fill all text fields 2. Don't take photos 3. Tap Save | Foto field shows error: "Kedua foto wajib diambil sebelum menyimpan." |
| CHK-010 | Successful save with all data | On check-in form | 1. Fill all required fields 2. Take both photos 3. Add optional note 4. Tap "Simpan Prospek" | Success modal appears with recap (name, PIC, phone, time, "2 foto terlampir") |
| CHK-011 | Success modal: "Selesai" | Success modal open | 1. Tap "Selesai" | Form reset; navigate to home; prospect count incremented |
| CHK-012 | Success modal: "Check In Lagi" | Success modal open | 1. Tap "Check In Lagi" | Form reset; stay on check-in form; scroll to top |
| CHK-013 | Catatan field is optional | On check-in form | 1. Fill required fields + photos, leave catatan empty 2. Save | Save succeeds |
| CHK-014 | Phone field numeric input | On check-in form | 1. Tap phone field 2. Type | Numeric keyboard appears (inputmode="numeric") |
| CHK-015 | Alamat field is textarea | On check-in form | 1. Tap Alamat field | Multi-line text input available |

### 7.2 Edge Cases

| TC ID | Scenario | Steps | Expected Result |
|-------|----------|-------|-----------------|
| CHK-E01 | Special characters in toko name | 1. Enter "Toko Berkah & Jaya!" 2. Fill rest 3. Save | Save succeeds; name stored correctly |
| CHK-E02 | Very long address text | 1. Enter 500+ char address 2. Fill rest 3. Save | Save succeeds (textarea expands) |
| CHK-E03 | Phone with spaces and dashes | 1. Enter "0812-3456-7890" 2. Save | Validation strips non-digits; accepts if ≥9 digits |
| CHK-E04 | Form fields cleared after save + "Selesai" | 1. Save a prospect 2. Tap "Selesai" 3. Reopen form | All fields empty; photo slots reset to placeholder |
| CHK-E05 | Retake photo | 1. Take plang photo 2. Tap retake icon on slot 3. Take new photo | New photo replaces old; timestamp updates |
| CHK-E06 | Only one photo taken | 1. Take plang only 2. Fill text fields 3. Save | Save fails; foto field shows error for missing selfie |

### 7.3 Acceptance Criteria
- ✅ All required fields (toko, alamat, pic, telp, 2 photos) must be validated before save
- ✅ Phone validation must strip non-digit characters and require ≥9 digits
- ✅ Photo slots must show thumbnail, timestamp tag, and retake button when filled
- ✅ Success modal must show recap of saved data
- ✅ "Selesai" must reset form and increment home prospect count
- ✅ "Check In Lagi" must reset form and stay on check-in screen
- ✅ Optional catatan field must not block save when empty

---

## 8. Camera

### 8.1 Functional Test Cases

| TC ID | Scenario | Preconditions | Steps | Expected Result |
|-------|----------|---------------|-------|-----------------|
| CAM-001 | Camera opens for plang | On check-in form | 1. Tap plang photo slot | Camera screen with live video feed; title "Foto Plang Toko" |
| CAM-002 | Camera opens for selfie | On check-in form | 1. Tap selfie photo slot | Camera screen; front camera preferred; title "Selfie dengan PIC Toko" |
| CAM-003 | Capture photo | Camera screen active | 1. Tap shutter button | Flash animation; photo captured; returns to check-in form; slot shows thumbnail |
| CAM-004 | Close camera without capture | Camera screen active | 1. Tap X button | Camera stops; returns to check-in form; no photo saved |
| CAM-005 | Flip camera | Camera screen active | 1. Tap flip button | Switches between front/back camera; toast confirms |
| CAM-006 | Gallery option | Camera screen active | 1. Tap gallery button | File picker opens for image selection |
| CAM-007 | Flash/torch toggle | Camera screen active | 1. Tap flash button | Button toggles "on" state (orange); torch activates if supported |
| CAM-008 | Guide frame visible | Camera screen active | 1. Observe camera view | Dashed border guide frame visible |
| CAM-009 | Hint text displays | Camera screen active | 1. Observe hint | Context-appropriate instructions shown (plang vs selfie) |
| CAM-010 | Photo stamp overlay | Photo captured | 1. Capture photo 2. View in detail | Photo has bottom overlay: date/time + coordinates/address with orange accent bar |
| CAM-011 | Selfie mirror effect | Selfie camera active | 1. Observe video | Video mirrored horizontally (front camera) |
| CAM-012 | "Gunakan Foto Contoh" fallback | Camera permission denied | 1. Observe error overlay 2. Tap "Gunakan Foto Contoh" | Sample photo generated; returns to form with slot filled |

### 8.2 Edge Cases

| TC ID | Scenario | Steps | Expected Result |
|-------|----------|-------|-----------------|
| CAM-E01 | Camera permission denied | 1. Open camera 2. Deny permission | Error overlay: "Kamera tidak dapat diakses" with message + "Gunakan Foto Contoh" button |
| CAM-E02 | No camera device | 1. Open camera on device without camera | Error: "Tidak ada kamera yang terdeteksi pada perangkat ini." |
| CAM-E03 | Shutter before camera ready | 1. Open camera 2. Immediately tap shutter | Toast: "Kamera belum siap, coba sesaat lagi." |
| CAM-E04 | Sample photo for plang | 1. Deny camera 2. Tap "Gunakan Foto Contoh" | Orange gradient image with "PLANG TOKO" text and stamp |
| CAM-E05 | Sample photo for selfie | 1. Open selfie 2. Deny camera 3. Tap "Gunakan Foto Contoh" | Blue gradient image with "SELFIE" text and stamp |
| CAM-E06 | Camera stream stopped on exit | 1. Open camera 2. Close without capture | `camStream` tracks stopped; no lingering camera indicator |
| CAM-E07 | Torch on unsupported device | 1. Tap flash button on device without torch | Toast: "Kilat layar aktif (perangkat tanpa lampu kilat)." |

### 8.3 Acceptance Criteria
- ✅ Camera must request appropriate facing mode (user for selfie, environment for plang)
- ✅ Captured photo must include timestamp and GPS coordinate stamp overlay
- ✅ Camera stream must be fully stopped when leaving camera screen
- ✅ Error fallback must provide "Gunakan Foto Contoh" option for demo continuity
- ✅ Front camera (selfie) must mirror the live preview
- ✅ Photo stamp must include: date/time (DD/MM/YYYY HH:MM WIB) and coordinates + address

---

## 9. History Prospek

### 9.1 Functional Test Cases

| TC ID | Scenario | Preconditions | Steps | Expected Result |
|-------|----------|---------------|-------|-----------------|
| HIS-001 | History list renders grouped by day | On history screen | 1. Observe list | Day groups with labels ("Hari Ini — Jumat, 4 September 2026", etc.) and item counts |
| HIS-002 | Each prospect item displays info | On history screen | 1. Observe a prospect item | Shows: thumbnail/initials, store name, address, time, PIC name |
| HIS-003 | Tap prospect opens detail | On history screen | 1. Tap any prospect item | Detail prospek screen appears with store data and photos |
| HIS-004 | Search by store name | On history screen | 1. Type store name in search | List filters to matching items only |
| HIS-005 | Search by address | On history screen | 1. Type part of address | List filters to matching items |
| HIS-006 | Search by PIC name | On history screen | 1. Type PIC name | List filters to matching items |
| HIS-007 | Search: no results | On history screen | 1. Type non-matching query | Empty state: "Tidak ada prospek yang cocok" with guidance |
| HIS-008 | Clear search button | Search has text | 1. Tap X clear button | Search cleared; full list restored; input focused |
| HIS-009 | Search clear button visibility | On history screen | 1. Type in search → X appears 2. Clear text → X disappears | Clear button toggles with query presence |
| HIS-010 | "Muat data sebelumnya" | On history screen | 1. Tap "Muat data sebelumnya" | Previous day's data loaded and appended; toast confirms |
| HIS-011 | Load older data exhausted | All older data loaded | 1. Tap load button when all loaded | Button disabled; text: "Semua data 7 hari terakhir sudah dimuat" |
| HIS-012 | Subtitle shows count | On history screen | 1. Observe header subtitle | "4 prospek hari ini" (or search result count) |
| HIS-013 | Search result count | On history screen | 1. Type query | Subtitle: "N hasil untuk 'query'" |
| HIS-014 | Back button | On history screen | 1. Tap back arrow | Home screen appears |

### 9.2 Edge Cases

| TC ID | Scenario | Steps | Expected Result |
|-------|----------|-------|-----------------|
| HIS-E01 | Search with special characters | 1. Type "!@#$%" in search | No results; empty state shown |
| HIS-E02 | Search case insensitivity | 1. Type "TOKO" (uppercase) | Matches "Toko" entries |
| HIS-E03 | Search after loading older data | 1. Load older data 2. Search | Search includes newly loaded older data |
| HIS-E04 | Day group with 0 matching items | 1. Search for term only in today | Only matching day groups shown; empty groups hidden |
| HIS-E05 | Load older while search active | 1. Search 2. Load older data | New data loads; search filter reapplied |

### 9.3 Acceptance Criteria
- ✅ History must be grouped by day with labels and item counts
- ✅ Search must filter by store name, address, and PIC name (case-insensitive)
- ✅ Empty state must display when no results match
- ✅ Clear search button must toggle visibility based on query presence
- ✅ "Muat data sebelumnya" must append data and disable when exhausted
- ✅ Each prospect item must navigate to its detail screen on tap

---

## 10. Detail Prospek

### 10.1 Functional Test Cases

| TC ID | Scenario | Preconditions | Steps | Expected Result |
|-------|----------|---------------|-------|-----------------|
| DTL-001 | Detail header shows store name | On detail screen | 1. Observe header | Store name in subtitle |
| DTL-002 | Two photos displayed | On detail screen | 1. Observe "Dokumentasi Foto" section | Plang photo and selfie photo in 2-column grid |
| DTL-003 | Tap photo opens viewer | On detail screen | 1. Tap plang photo | Full-screen photo viewer opens |
| DTL-004 | Data Toko section | On detail screen | 1. Observe "Data Toko" card | Shows: Nama Toko, Alamat Toko, Nama PIC, No. Telp PIC, Waktu Kunjungan, Titik Koordinat |
| DTL-005 | Catatan Kunjungan section | On detail screen | 1. Observe catatan box | Visit notes displayed (or "Tidak ada catatan tambahan" if empty) |
| DTL-006 | "Unduh Kedua Foto" button | On detail screen | 1. Tap "Unduh Kedua Foto" | Both photos downloaded sequentially |
| DTL-007 | "Hubungi PIC" button | On detail screen | 1. Tap "Hubungi PIC" | Initiates tel: link; toast: "Menghubungi [PIC name] · [phone]" |
| DTL-008 | Back button | On detail screen | 1. Tap back arrow | History screen appears |

### 10.2 Edge Cases

| TC ID | Scenario | Steps | Expected Result |
|-------|----------|-------|-----------------|
| DTL-E01 | Prospect with no catatan | 1. Open a prospect with empty note field | Catatan box: "Tidak ada catatan tambahan pada kunjungan ini." |
| DTL-E02 | Generated photo for older prospects | 1. Open older prospect detail | Photos generated on-the-fly with store name, coordinates, and stamp |
| DTL-E03 | Phone number missing | 1. Open prospect with no telp | Phone defaults to "0812 3456 7890" |

### 10.3 Acceptance Criteria
- ✅ Detail must show all stored data: name, address, PIC, phone, time, coordinates, notes, 2 photos
- ✅ Photos must be tappable to open full-screen viewer
- ✅ "Unduh Kedua Foto" must download both images with sequential delays
- ✅ "Hubungi PIC" must trigger phone dialer (tel: link)
- ✅ Back button must return to history screen

---

## 11. Photo Viewer

### 11.1 Functional Test Cases

| TC ID | Scenario | Preconditions | Steps | Expected Result |
|-------|----------|---------------|-------|-----------------|
| PVW-001 | Viewer displays photo full screen | On detail screen | 1. Tap a photo | Full-screen black viewer with image (object-fit: contain) |
| PVW-002 | Viewer title shows photo type | Photo viewer open | 1. Observe top bar | "Foto Plang Toko" or "Selfie dengan PIC Toko" |
| PVW-003 | Segmented toggle: Plang | Photo viewer open | 1. Tap "Plang" segment | Plang photo displayed; segment highlighted |
| PVW-004 | Segmented toggle: Selfie | Photo viewer open | 1. Tap "Selfie" segment | Selfie photo displayed; segment highlighted |
| PVW-005 | "Unduh Foto Ini" button | Photo viewer open | 1. Tap "Unduh Foto Ini" | Current photo downloaded with filename `Prospek_[StoreName]_[plang|selfie].jpg` |
| PVW-006 | Close viewer | Photo viewer open | 1. Tap X button | Returns to detail screen |
| PVW-007 | Hint text about long-press | Photo viewer open | 1. Observe bottom hint | "Di ponsel, tekan lama pada foto untuk menyimpannya ke galeri." |

### 11.2 Edge Cases

| TC ID | Scenario | Steps | Expected Result |
|-------|----------|-------|-----------------|
| PVW-E01 | Toggle segments rapidly | 1. Open viewer 2. Rapidly tap Plang/Selfie segments | Final segment's photo displays; no broken state |
| PVW-E02 | Download file naming convention | 1. Open viewer for "Toko Berkah Jaya" plang 2. Download | Filename: `Prospek_Toko_Berkah_Jaya_plang.jpg` |

### 11.3 Acceptance Criteria
- ✅ Viewer must display photo in full-screen, contain-fit mode
- ✅ Segmented control must switch between plang and selfie photos
- ✅ Download must produce file with naming convention `Prospek_[slug]_[type].jpg`
- ✅ Close button must return to detail screen

---

## 12. Notifications

### 12.1 Functional Test Cases

| TC ID | Scenario | Preconditions | Steps | Expected Result |
|-------|----------|---------------|-------|-----------------|
| NTF-001 | Notifications grouped by date | On notif screen | 1. Observe list | Groups: "Hari Ini", "Kemarin", "Rabu, 2 September 2026" with unread counts |
| NTF-002 | Unread badge on items | On