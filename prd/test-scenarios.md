# Product Requirements Document (PRD): Test Scenarios
**Project:** Sales Point Application
**Focus:** QA, Test Scenarios, Acceptance Criteria, and Validation Steps

---

## 1. Introduction
Based on the provided UI mockup, the "Sales Point" application is a mobile-first sales force management tool. It includes authentication, attendance tracking via GPS and camera, a performance dashboard, prospect management, and notifications. This document outlines the comprehensive test scenarios, edge cases, and acceptance criteria required to validate the application's functionality and robustness.

---

## 2. Authentication & Onboarding

### 2.1 Login Screen
**Acceptance Criteria:**
- User must be able to log in using valid credentials.
- Invalid credentials must trigger an error message.
- User can toggle password visibility.
- User can select "Remember Me".
- User can view demo login instructions.

**Functional Test Cases:**
| Test ID | Scenario | Pre-conditions | Test Steps | Expected Result |
|---|---|---|---|---|
| AUTH-01 | Valid Login | App is on Login screen | 1. Enter valid username. 2. Enter valid password. 3. Click "Masuk" (Login) | User is navigated to Loading/Home screen |
| AUTH-02 | Invalid Login | App is on Login screen | 1. Enter invalid username. 2. Enter invalid password. 3. Click "Masuk" | Login alert appears with error message |
| AUTH-03 | Empty Fields Validation | App is on Login screen | 1. Leave username/password blank. 2. Click "Masuk" | Input fields show error borders; login blocked |
| AUTH-04 | Password Visibility Toggle | App is on Login screen; password is typed | 1. Click the eye icon in password field | Password text toggles between masked and plain text |
| AUTH-05 | Remember Me | App is on Login screen | 1. Check "Remember Me". 2. Login. 3. Close app. 4. Reopen app | User bypasses login screen and goes directly to loading |

**Edge Cases:**
- Network timeout during login attempt.
- User attempts SQL injection or XSS in input fields.
- Password is copy-pasted with leading/trailing spaces.

**Validation Steps:**
1. Verify UI matches the mockup (purple header, white sheet input area).
2. Verify error states (`is-invalid` class) render with red borders and error text.
3. Verify the `login-alert.show` class is triggered properly on auth failure.

---

## 3. App Initialization & Loading

### 3.1 Splash & Loading States
**Acceptance Criteria:**
- Splash screen displays brand logo and loading spinner.
- Transition from Splash to Loading to Home/Gate is smooth.
- Loading screen displays progress percentage and status text.

**Functional Test Cases:**
| Test ID | Scenario | Pre-conditions | Test Steps | Expected Result |
|---|---|---|---|---|
| LOAD-01 | Splash Screen Visibility | App is launched | Observe initial screen | Splash screen displays for ~2-3 seconds with spinner animation |
| LOAD-02 | Loading Progress | User just authenticated | Observe Loading screen | Progress bar fills 0 to 100% with status text updates |
| LOAD-03 | Clock-in Gate Redirect | User hasn't clocked in today | Wait for loading to complete | App redirects to Clock-in Gate screen |

**Edge Cases:**
- App is force-closed during the loading phase; reopening should resume gracefully.
- API calls during loading fail; appropriate error toast/modal should appear.

---

## 4. Attendance & Clock-in Module

### 4.1 Clock-in Gate & Form
**Acceptance Criteria:**
- User can see current time and location status.
- User can initiate Clock-in/Clock-out.
- User must capture a photo as part of the attendance form.
- User can view attendance history.

**Functional Test Cases:**
| Test ID | Scenario | Pre-conditions | Test Steps | Expected Result |
|---|---|---|---|---|
| ATT-01 | Location Fetch | App is on Clock-in Gate | Observe location chip | App displays current GPS location accurately |
| ATT-02 | Clock-in Action | User is clocked out | 1. Click Clock-in button. 2. Complete photo capture. 3. Submit form | Success modal appears; Home screen Clock-in tile turns Green/Active |
| ATT-03 | Clock-out Action | User is clocked in | 1. Click Clock-out button. 2. Complete photo capture. 3. Submit form | Success modal appears; Home screen Clock-out tile turns Red/Active |
| ATT-04 | View Attendance Detail | User has attendance records | 1. Go to Home. 2. Click Attendance card | Navigates to Attendance Detail showing dates, durations, and times |

**Edge Cases:**
- **GPS Disabled:** User tries to clock in. App should show a warning or prompt to enable GPS.
- **GPS Permission Denied:** App should show an explanation and request permission again.
- **Clock-out without Clock-in:** System should prevent this and show an error.
- **Duplicate Clock-in:** Attempting to clock in twice should be blocked.

**Validation Steps:**
1. Verify CSS classes `green`, `red`, `done` on `.clock-tile` change based on attendance state.
2. Verify the GPS `loc-chip` displays the correct address or coordinates.
3. Ensure the success modal contains recap data (time, location, photo).

---

## 5. Camera & Photo Capture

### 5.1 In-App Camera
**Acceptance Criteria:**
- User can access device camera from the Check-in form.
- User can take a photo and review it.
- User can retake the photo.
- User can cancel camera operation.

**Functional Test Cases:**
| Test ID | Scenario | Pre-conditions | Test Steps | Expected Result |
|---|---|---|---|---|
| CAM-01 | Open Camera | User is on Check-in Form | Click photo slot | Camera opens with guide overlay (face guide) |
| CAM-02 | Capture Photo | Camera is open | Click shutter button | Flash animation plays; photo is captured and displayed in slot |
| CAM-03 | Retake Photo | Photo is already captured | Click "Retake" button on photo slot | Camera opens again; previous photo is cleared |
| CAM-04 | Cancel Camera | Camera is open | Click back/close button | Returns to Check-in form; photo slot remains empty |

**Edge Cases:**
- **Camera Permission Denied:** App shows `cam-error` overlay prompting user to enable permissions in settings.
- **Front/Back Toggle:** User toggles camera (if button exists in UI); stream should mirror/unmirror appropriately (`#camVideo.mirror`).
- **Storage Full:** Camera fails to save; error toast displayed.

**Validation Steps:**
1. Verify the mirror effect (CSS `transform: scaleX(-1)`) applies correctly when switching cameras.
2. Verify the `cam-flash.fire` animation triggers on shutter click.
3. Check that the captured image fits the `aspect-ratio: 3/4` in the preview slot.

---

## 6. Home Dashboard

### 6.1 Dashboard Display & Navigation
**Acceptance Criteria:**
- Dashboard displays user info, reference code, and stats (e.g., target vs realization).
- Quick menu tiles navigate to respective modules.
- Upcoming/Recent prospects list is displayed.

**Functional Test Cases:**
| Test ID | Scenario | Pre-conditions | Test Steps | Expected Result |
|---|---|---|---|---|
| HOME-01 | Stats Rendering | User has sales data | View Home screen | Stat cards show correct numbers, colored correctly (accent/orange) |
| HOME-02 | Copy Reference Code | User is on Home screen | Click "Copy" button next to Ref Code | Code is copied to clipboard; toast "Copied!" appears |
| HOME-03 | Navigate to Prospects | User is on Home screen | Click "Lihat Semua" (See All) on prospects list | Navigates to full Prospects List screen |
| HOME-04 | View Prospect Detail | User is on Home screen | Click a prospect item | Navigates to Prospect Detail screen |

**Edge Cases:**
- Stats data is null/undefined; UI should gracefully fallback to '0' or 'N/A'.
- User name is extremely long; CSS `truncate` or wrapping should prevent UI breakage.

**Validation Steps:**
1. Verify progress meter (`meter-fill` width) calculates correctly against `stat-cell` values.
2. Verify the copy button icon changes or toast appears successfully.

---

## 7. Prospects Module

### 7.1 Prospect List & Detail
**Acceptance Criteria:**
- User can search for prospects.
- User can view prospect details including attached photos.
- User can view photos in full screen.

**Functional Test Cases:**
| Test ID | Scenario | Pre-conditions | Test Steps | Expected Result |
|---|---|---|---|---|
| PROS-01 | Search Prospects | User is on Prospects list | 1. Click search bar. 2. Type query | List filters dynamically to match query; clear (X) button appears |
| PROS-02 | Clear Search | User has typed a query | Click (X) button in search bar | Search clears; full list restores |
| PROS-03 | View Prospect Detail | User is on Prospects list | Click a prospect item | Detail screen opens showing info rows, notes, and photos |
| PROS-04 | Full Screen Photo Viewer | User is in Prospect Detail | Click on a photo | Full screen black viewer opens (`#s-photo`) |

**Edge Cases:**
- **Empty Search Results:** Search yields no matches; display `empty` state icon and text.
- **Broken Image URLs:** Prospect photo fails to load; display placeholder or broken image icon.

**Validation Steps:**
1. Verify the search bar shows `.has-q` class when text is entered to reveal the clear button.
2. Verify the full-screen viewer (`viewer-stage`) properly uses `object-fit: contain` so the image isn't cropped.
3. Check the `seg` (segmented control) in photo viewer switches views if applicable.

---

## 8. Notifications Module

### 8.1 Notification List
**Acceptance Criteria:**
- User can view all notifications.
- Unread notifications are visually distinguished.
- User can navigate back from notifications.

**Functional Test Cases:**
| Test ID | Scenario | Pre-conditions | Test Steps | Expected Result |
|---|---|---|---|---|
| NOTIF-01 | View Notifications | User clicks Notification icon | Navigate to Notifications screen | List of notifications displays with correct icons and timestamps |
| NOTIF-02 | Unread Indicators | User has unread notifications | Observe list | Unread items have left blue border (`.notif.unread`) |
| NOTIF-03 | Read Notification | User clicks an unread notification | Click notification item | Item loses unread styling (border removed) |

**Edge Cases:**
- No notifications exist; App should display an empty state message.
- Extremely long notification text; CSS ellipsis (`text-overflow: ellipsis`) or multi-line wrapping should apply correctly.

---

## 9. Global UI/UX Elements

### 9.1 Modals, Toasts, and Navigation
**Acceptance Criteria:**
- Success/Error modals display correctly with animation.
- Toasts appear and disappear automatically.
- Bottom navigation switches between Home, Prospects, and Notifications seamlessly.

**Functional Test Cases:**
| Test ID | Scenario | Pre-conditions | Test Steps | Expected Result |
|---|---|---|---|---|
| UI-01 | Modal Display | Action triggers a success | e.g., Submit attendance | Modal pops up with scale animation (`pop` keyframe) |
| UI-02 | Toast Display | Action triggers a toast | e.g., Copy text | Toast slides in from bottom (`toastin` keyframe) |
| UI-03 | Bottom Nav Switching | User is on Home | Click "Prospects" tab | Screen transitions; Prospects tab highlights active |
| UI-04 | Network Indicator | App loses connectivity | Trigger network disconnect | App displays appropriate offline behavior/toast |

**Edge Cases:**
- Rapidly switching tabs to test memory leaks or overlapping screens.
- Reduced motion accessibility: User has "Reduce Motion" enabled on OS; animations should be disabled (`@media (prefers-reduced-motion: reduce)`).

**Validation Steps:**
1. Inspect `active` class toggling on `.screen` elements during navigation.
2. Verify the `prefers-reduced-motion` media query nullifies transition durations.
3. Verify safe-area insets (`env(safe-area-inset-bottom)`) are respected on iOS devices so buttons aren't hidden by the home indicator.