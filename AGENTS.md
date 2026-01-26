# AGENDS.md — Prayer Lock (Islamic Focus)
A blueprint + step-by-step AI build prompts for an Expo (iOS + Android) app that pauses selected apps during Islamic prayer times, works offline, tracks daily prayers, and uses a calm premium UI (noise background + emerald accents). This document is written to be consumed by an AI coding agent.

---

## 0) Product rules (non-negotiable)
- **Android-first for blocking.** Android supports overlay/block via system permissions; iOS does **not** allow true app blocking/overlays. iOS runs in **Soft Mode** (prayer times, reminders, tracking, Focus guidance).
- **No login required.** All data local-first (device storage). Optional subscriptions (RevenueCat) later.
- **No judgement language.** Tracking is “confirmed / bypass / unknown”, not “faith scoring”.
- **Offline-first.** Prayer times computed locally from coordinates and method (no paid API).
- **Progressive permissions.** Explain → user action → request permission → show immediate value.

---

## 1) Tech stack (best practice)
- **Expo (latest stable SDK)** + **Expo Router**
- **TypeScript strict** (no `any`, no `unknown`)
- **NativeWind (Tailwind CSS v4)** for styling
- **State**: Zustand + **zenty** wrapper for simpler store updates
- **Validation**: Zod
- **Persistence**: AsyncStorage (settings + tracking); SecureStore only if later needed for keys
- **Location**: `expo-location`
- **Notifications**: `expo-notifications` (reminders + “did you pray?” prompt)
- **Prayer times**: local computation using a TS-compatible Adhan/prayer-time library (no paid API), plus method presets (MWL, Umm al-Qura, ISNA, Egyptian)
- **Android blocker**: custom native module + Accessibility/Usage Access + Overlay Activity (requires `expo prebuild`)

---

## 2) App structure (Expo Router)

app/
(onboarding)/
index.tsx
step-2.tsx
step-3.tsx
(setup)/
location.tsx
method.tsx
apps.tsx
duration.tsx
mute.tsx
enable.tsx
(tabs)/
_layout.tsx
home.tsx
settings.tsx
overlay/
index.tsx # React UI for overlay (Android only, launched by native Activity)

src/
core/
constants/
theme/
ui/
storage/
permissions/
prayer/
tracking/
schedule/
platform/
android/
ios/
state/
store.ts
selectors.ts
features/
onboarding/
setup/
home/
settings/
overlay/
assets/
noise.png
fonts/

---

## 3) Core data models (Zod)
### 3.1 Settings schema
- `hasCompletedOnboarding: boolean`
- `theme: "system" | "light" | "dark"`
- `enabled: boolean` (global prayer lock enabled)
- `location: { lat: number; lon: number; timezone: string; label?: string } | null`
- `calcMethod: "MWL" | "UmmAlQura" | "ISNA" | "Egyptian"`
- `madhab: "shafi" | "hanafi"`
- `adjustmentsMinutes: { fajr: number; dhuhr: number; asr: number; maghrib: number; isha: number }`
- `blockedAppsAndroid: string[]` (package names)
- `blockDurationMinutes: number` (presets + custom, min 1 max 120)
- `muteDuringPrayer: boolean`
- `muteDurationMinutes: number` (default = blockDurationMinutes)
- `softModeIOS: boolean` (always true on iOS)
- `versionAcknowledged: string` (for migrations)

### 3.2 Daily tracking schema
Track each prayer of each day:
- Day key: `YYYY-MM-DD` in user’s timezone
- Prayers: `fajr, dhuhr, asr, maghrib, isha`
- Each prayer status:
  - `"confirmed"` = user tapped **Continue after prayer**
  - `"bypassed"` = user used **Emergency bypass**
  - `"unknown"` = time passed with no action
- Timestamps for audit:
  - `confirmedAt?: number`
  - `bypassedAt?: number`
  - `promptedAt?: number` (notification “did you pray?”)

---

## 4) UI design system (noise + emerald)
### 4.1 Colors (recommended)
- Background: **charcoal** `#0B0F14`
- Card: `#111827` (very dark slate)
- Text primary: `#F5F7FA`
- Text secondary: `#9CA3AF`
- Accent emerald: `#1F8A5B` (muted)
- Danger (only for bypass marker): muted red `#B4534C` (never neon)

### 4.2 Typography
- Headline: serif (e.g. DM Serif / Playfair)
- Body: clean sans

### 4.3 Components
- `NoiseBackground`
- `PrimaryButton` (emerald accent)
- `GhostButton`
- `Card`
- `Toggle`
- `SegmentedControl`
- `ListRow`
- `DotPager`

---

## 5) Prayer times: offline + accurate
### 5.1 Source of truth
- Use **location coordinates** + calculation method + madhab + adjustments.
- Compute:
  - today’s times
  - next day’s times
- Recompute:
  - on app launch
  - at local midnight (schedule a background refresh best-effort)
  - when location changes significantly

### 5.2 “Prayer window”
Define an “active prayer window” based on:
- prayer time start (e.g., Asr start)
- lock duration minutes (user setting)
During this window:
- Android: blocking overlay may trigger if a blocked app opens
- iOS: show notification + in-app overlay screen only

---

## 6) Android blocking (best-practice implementation)
**Expo alone cannot reliably block other apps.** Use `expo prebuild` and ship a custom Android module.

### 6.1 Permissions (Android)
- Location permission (Expo)
- **Usage Access** (to know foreground app reliably)
- **Accessibility Service** (to react fast + show overlay trigger)
- Optional: Notification permission (for prompts)

### 6.2 Blocking mechanism
- A foreground detector:
  - Prefer AccessibilityService events for app changes
  - Fallback to UsageStats polling (low frequency) if needed
- When conditions match:
  - current time is within active prayer window
  - global enabled = true
  - foreground package is in blocked list
  → Launch `PrayerOverlayActivity` (full-screen, non-dismissable except allowed actions)

### 6.3 Safety
- Always include **Emergency bypass**.
- Bypass never disables prayer times; it only suspends lock temporarily.
- Keep battery usage low.

---

## 7) iOS behavior (Soft Mode)
iOS cannot block/overlay other apps:
- Show prayer times + tracking + reminders
- Offer Focus guidance:
  - “Create a Prayer Focus to silence apps during Salah”
- Allow “mute toggle” to only schedule reminders (no system mute)

---

## 8) Tracking logic (daily 5 prayers)
### 8.1 What counts as “confirmed”
- If user presses **Continue after prayer** during that prayer window → status = confirmed (green).
### 8.2 What counts as “bypassed”
- If user presses **Emergency bypass** → status = bypassed (muted red).
### 8.3 Unknown
- If prayer window ends and status still unset → unknown (no bg).
### 8.4 Prompt before next prayer
- Before next prayer time, send a notification:
  - “Did you pray [Previous Prayer]?”
  - Buttons:
    - “Yes” → mark confirmed
    - “No / Skipped” → mark bypassed (or keep unknown if preferred)
- Only prompt once per prayer.

---

## 9) Screens: what to show inside the app
### 9.1 Onboarding (3 screens)
- Short text, noise background, emerald accent word(s), dots and arrow button.
- No permissions asked here.

### 9.2 Setup flow
1) Location
2) Method & madhab
3) Duration (presets + custom)
4) Mute toggle and mute duration link to block duration
5) Apps to block (Android only)
6) Enable (Android permissions screen + deep links)

### 9.3 Home
- Next prayer + countdown
- Today’s list (5 times)
- If day is finished, show next day
- Tracking row per prayer:
  - green background if confirmed
  - muted red if bypassed
  - none if unknown or upcoming
- Global toggle: enabled on/off
- Mute toggle quick status

### 9.4 Settings
- Theme: system/light/dark
- Calculation method + madhab + adjustments
- Location refresh
- Blocked apps (Android)
- Duration settings
- Mute settings
- Notifications toggle
- About:
  - Version
  - Privacy
  - Terms
  - Imprint