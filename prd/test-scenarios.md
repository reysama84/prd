# Product Requirements Document (PRD): Test Scenarios
**Project:** Sales Point Mobile Application
**Focus:** Quality Assurance & Test Strategy
**Source Material:** UI Mockup (HTML/CSS)

> **Note:** As the provided user requirements text was corrupted/unreadable, the following test scenarios, acceptance criteria, and validation steps are derived exclusively from the structural and functional elements present in the provided UI mockup.

---

## 1. Authentication & Access Control

### Acceptance Criteria
- The system must allow access only to authenticated users.
- The login form must validate inputs before submission.
- The UI must provide feedback during authentication and on error.

### Functional Test Cases
| Test Case ID | Description | Expected Result |
|---|---|---|
| AUTH-01 | Enter valid credentials and tap "Login". | User is authenticated; app transitions to Loading screen. |
| AUTH-02 | Enter invalid credentials and tap "Login". | Submission fails; the `.login-alert` error message becomes visible. |
| AUTH-03 | Tap the password visibility toggle (`.pw-toggle`). | Password text switches between masked (`•••`) and plain text. |
| AUTH-04 | Tap "Remember Me" checkbox, login, close app, reopen. | User is bypassed past the login screen on next launch. |
| AUTH-05 | Focus on input fields. | Input border changes to accent color with a soft box-shadow. |

### Edge Cases
- **Empty Submission:** Attempt to login with empty username/password fields. Verify submission is blocked or validation error appears.
- **Rapid Tapping:** Tap the "Login" button rapidly. Verify no multiple API calls are fired or duplicate navigation occurs.
- **Whitespace Inputs:** Enter only spaces in username/password. Verify it is treated as invalid.

### Validation Steps
1. Navigate to `#s-login` screen.
2. Inspect DOM for `.login-alert` visibility state on error.
3. Verify focus states (`:focus-visible`) are applied for keyboard navigation.

---

## 2. App Initialization & Navigation

### Acceptance Criteria
- The app must display a splash screen upon initialization.
- A loading screen with a progress indicator must show while fetching initial data.
- Bottom navigation must successfully switch between primary contexts (Home, Prospects, Notifications).

### Functional Test Cases
| Test Case ID | Description | Expected Result |
|---|---|---|
| INIT-01 | Launch the application. | `#s-splash` screen displays brand logo and spinner dots animate. |
| INIT-02 | Observe transition post-splash. | App transitions to `#s-loading` screen; progress bar fills smoothly. |
| INIT-03 | Tap "Home" nav button. | `#s-home` becomes active; Home nav button gets `.active` class (accent color + top indicator). |
| INIT-04 | Tap "Notifications" nav button. | `#s-notif` becomes active; notification badge dot disappears if items are read. |

### Edge Cases
- **Slow Network:** Simulate 3G network. Verify the loading bar pauses appropriately and does not freeze the UI.
- **Animation Preferences:** Enable "Reduce Motion" in OS settings. Verify screen transition animations (`screen-in`) are disabled or shortened.

### Validation Steps
1. Verify CSS animation `@keyframes bounce` (splash) and `@keyframes screen-in` execute.
2. Check `active` class toggling logic on `.navbtn` elements.

---

## 3. Attendance Management & Camera Integration

### Acceptance Criteria
- User must be able to view current time, location, and clock-in status.
- User must capture a photo using the device camera to complete check-in.
- System must handle camera permission denials gracefully.

### Functional Test Cases
| Test Case ID | Description | Expected Result |
|---|---|---|
| ATT-01 | Navigate to Clock-in Gate (`#s-gate`). | Current time updates dynamically; location chip displays detected GPS location. |
| ATT-02 | Tap Clock-in button. | Navigates to Check-in Form (`#s-checkin`). |
| ATT-03 | Tap photo slot in form. | Camera screen (`#s-camera`) opens; live camera feed displays in `.cam-stage`. |
| ATT-04 | Tap shutter button. | Screen flash effect (`.cam-flash.fire`) triggers; photo is captured and previewed in slot. |
| ATT-05 | Tap "Retake" icon on captured photo. | Camera screen reopens; previous photo cleared. |
| ATT-06 | Submit Check-in form. | Success modal (`.modal`) displays recap info; user returns to Home. |

### Edge Cases
- **Camera Denied:** User denies camera permissions. Verify `.cam-error.show` displays with a friendly message and retry option.
- **GPS Disabled:** Location services off. Verify location chip shows "Unable to fetch location" or blocks check-in.
- **Background Interruption:** User switches apps while camera is open. Verify camera stream is safely released and resumed on return.
- **No Front Camera:** Test on device without front camera. Verify default camera selection doesn't crash.

### Validation Steps
1. Inspect `#s-camera` DOM to ensure `#camVideo` srcObject is bound correctly.
2. Verify `.cam-guide` aspect ratio aligns with video stream.
3. Verify `.cam-flash` animation duration is exactly `0.34s`.

---

## 4. Home Dashboard

### Acceptance Criteria
- Dashboard must display user profile, reference code, sales stats, and quick actions.
- Reference code must be copyable to clipboard.
- Quick action tiles must navigate to correct modules.

### Functional Test Cases
| Test Case ID | Description | Expected Result |
|---|---|---|
| HOME-01 | View Home Screen (`#s-home`). | User avatar, name, reference code chip, and stat cards render correctly. |
| HOME-02 | Tap the Copy button (`.copy-btn`) next to reference code. | Code is copied to clipboard; toast notification appears briefly. |
| HOME-03 | Tap "Clock In/Out" tile (`.clock-tile`). | Navigates to Attendance module. |
| HOME-04 | Tap a Menu Tile (`.menu-tile`). | Navigates to corresponding feature screen. |
| HOME-05 | Tap "See All" on prospects strip. | Navigates to Prospects list screen. |

### Edge Cases
- **Zero Stats:** New user with 0 sales. Verify stat cards display "0" without layout breaking.
- **Long Names:** User with very long name. Verify text truncates (`text-overflow: ellipsis`) rather than overflowing the header.
- **Clipboard API Block:** Browser/OS blocks clipboard access. Verify fallback toast or error message.

### Validation Steps
1. Verify `navigator.clipboard.writeText` is called on copy button click.
2. Inspect layout flex properties on `.stat-grid` to ensure responsiveness on narrow screens.
3. Check `.menu-tile:active` scale transform (`.98`) is applied.

---

## 5. Prospect Management & Photo Viewer

### Acceptance Criteria
- User must be able to view a list of prospects grouped by date.
- List must be searchable.
- User must be able to view prospect details and zoom/view photos in fullscreen.

### Functional Test Cases
| Test Case ID | Description | Expected Result |
|---|---|---|
| PROS-01 | View Prospect List screen. | Prospects load grouped under `.daygroup` headers (e.g., "Today", "Yesterday"). |
| PROS-02 | Type text into search bar (`.searchbar input`). | List filters dynamically; clear button (`.clr`) appears. |
| PROS-03 | Tap clear button (`.clr`). | Search input clears; full list restores. |
| PROS-04 | Tap a prospect item (`.pitem`). | Navigates to Prospect Detail screen (`#s-detail`). |
| PROS-05 | Tap a photo in prospect detail (`.photo-card`). | Opens fullscreen Photo Viewer (`#s-photo`). |
| PROS-06 | Tap back button in Photo Viewer. | Returns to Prospect Detail screen. |

### Edge Cases
- **Empty Search Results:** Search for gibberish. Verify `.empty` state component displays "No prospects found".
- **Missing Photos:** Prospect with no photos. Verify a placeholder icon/text is shown instead of broken image links.
- **Extremely Long Notes:** Prospect note exceeds screen height. Verify `.note-box` scrolls internally or text truncates.

### Validation Steps
1. Verify search input event listener filters `.pitem` elements efficiently.
2. Check `.photo-card` aspect ratio (`3/4`) maintains integrity across different screen sizes.
3. Verify `.daygroup-label` right border line flexes correctly.

---

## 6. Notifications

### Acceptance Criteria
- User must be able to view a list of system notifications.
- Unread notifications must be visually distinct.
- User must be able to mark notifications as read.

### Functional Test Cases
| Test Case ID | Description | Expected Result |
|---|---|---|
| NOTIF-01 | View Notifications screen (`#s-notif`). | Notifications list renders. |
| NOTIF-02 | Observe unread items. | `.notif.unread` elements have a left accent border (`.notif.unread border-left`). |
| NOTIF-03 | Tap an unread notification. | Item navigates to target content; visual style updates to "read" state. |
| NOTIF-04 | View bottom nav badge. | Red badge dot on Notifications nav button reflects unread count. |

### Edge Cases
- **No Notifications:** User with zero notifications. Verify `.empty` state is displayed.
- **100+ Notifications:** High volume. Verify list scrolls smoothly without lag; pagination or "Load More" appears if applicable.

### Validation Steps
1. Inspect DOM for `.notif.unread` class manipulation on tap.
2. Verify nav badge (`.navbtn .dot`) number updates dynamically.

---

## 7. UI/UX & Cross-Cutting Concerns

### Acceptance Criteria
- App must be responsive and fit within the device viewport without horizontal scrolling.
- Interactive elements must have clear active/hover states.
- Accessibility features (focus states, reduced motion) must function.

### Functional Test Cases
| Test Case ID | Description | Expected Result |
|---|---|---|
| UI-01 | Resize browser window to mobile width (< 560px). | `.device` fills viewport; status bar hidden. |
| UI-02 | Resize browser window to desktop width (> 560px). | `.device` is centered with rounded corners and device shell shadow; status bar visible. |
| UI-03 | Tab through form inputs. | `:focus-visible` outline appears clearly on focused element. |
| UI-04 | Tap and hold a button (e.g., `.btn`). | Button scales down slightly (transform: scale(.985)) providing tactile feedback. |
| UI-05 | Trigger overlay modal (`.overlay`). | Background blurs (`.backdrop-filter`); modal pops in with scale animation. |

### Edge Cases
- **Landscape Orientation:** Rotate device to landscape. Verify layout doesn't break or stretch awkwardly (specifically forms and camera views).
- **High Contrast Mode:** Enable OS high contrast. Verify text remains legible against navy/white backgrounds.
- **Font Scaling:** Increase system font size to 150%. Verify text containers expand without text clipping.

### Validation Steps
1. Validate CSS media queries `@media (min-width:560px)` apply correct device shell styling.
2. Verify `prefers-reduced-motion: reduce` media query disables all transition durations.
3. Check `env(safe-area-inset-bottom)` padding is respected on notched devices.