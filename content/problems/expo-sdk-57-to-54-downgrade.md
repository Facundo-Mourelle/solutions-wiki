---
title: "Expo SDK 57 to 54 Downgrade"
date: 2026-08-20T14:30:00-03:00
categories: ["Development"]
tags: ["expo", "react-native", "mobile", "dependency-management"]
difficulty: "beginner"
status: "solved"
---

# Expo SDK 57 to 54 Downgrade

## TL;DR

Downgraded an Expo project from SDK 57 to SDK 54 because the target Android device couldn't run Expo Go 57. Fixed config plugin errors and version mismatches.

## Problem Description

A mobile app was scaffolded with Expo SDK 57, but the testing Android phone couldn't run that version of Expo Go. Needed to downgrade to SDK 54 for device compatibility.

## Root Cause

Expo SDK 57 requires newer Android API levels. Older devices can't run Expo Go 57.

## Solution

### 1. Update package.json

Changed dependency versions to SDK 54-compatible:

```json
{
  "expo": "~54.0.0",
  "expo-file-system": "~19.0.24",
  "expo-sharing": "~14.0.8",
  "expo-sqlite": "~16.0.10",
  "expo-status-bar": "~3.0.9",
  "react": "19.1.0",
  "react-native": "0.81.5"
}
```

### 2. Fix config plugin error

Removed `expo-sharing` from plugins in `app.json` (not used yet):

```json
"plugins": [
  "expo-sqlite"
]
```

### 3. Run expo install --fix

Auto-corrected all version mismatches:

```bash
npx expo install --fix
```

### 4. Reinstall dependencies

```bash
rm -f package-lock.json && npm install
```

## Verification

- `npx expo-doctor` → 18/18 checks passed
- `tsc --noEmit` → no errors
- `npx jest` → 46/46 tests passed
- `npx expo export --platform android` → bundled successfully (595 modules)

## Key Learnings

- Use `npx expo install <package>` instead of `npm install` to get SDK-compatible versions
- `npx expo install --fix` auto-corrects version mismatches
- Config plugins in `app.json` must match installed SDK versions
- Expo Go version must match or be older than the SDK version on the device

## Files Modified

- `mobile/package.json` — dependency versions
- `mobile/app.json` — removed expo-sharing plugin
