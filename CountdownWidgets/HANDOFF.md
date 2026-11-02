# Countdown Widgets: Handoff

Status doc for any AI or developer who continues this project. Read it fully before writing code.

## 1. What this is
An Android app (Capacitor hybrid) plus three native home-screen widgets (Text, Photo, Countdown). The WebView app is the widget editor. The widgets are native Kotlin and render themselves to Bitmaps so the fonts, colors and 7-segment digits match the editor. The editor is an adapted copy of the demo app, `index_-_2026-11-02T234300_603.html` (about 81 KB, a single file with inline CSS and JS).

- Package/appId: `com.countdown.widgets`
- Platform: Android only (minSdk 26, target/compile 34)
- Stack: Capacitor 6, Kotlin, WorkManager, AppWidgetProvider with RemoteViews

## 2. Decisions already made
1. Hybrid approach: keep the existing HTML as the editor UI, write widgets natively.
2. Widgets are drawn to a Bitmap by `WidgetRenderer.kt` and shown in an ImageView inside RemoteViews.
3. Android cannot redraw widgets every second, so countdowns tick once per minute. Days/hours/minutes are accurate and seconds are approximate. Do not promise live seconds.
4. Photo auto-change cannot be seconds-level. Use AlarmManager inexact alarms or WorkManager (15 min minimum) and clamp the UI to those limits. The Zoom transition is dropped on the widget; the editor preview may keep it.
5. Frosted blur is not possible on a widget, so it is replaced by a translucent fill. The editor should show a note.
6. Long-press menu: the launcher owns long-press on widgets. The menu items (shuffle, pause, remove photo, full screen) move into the app editor. The "Actions" tab is reduced to tap behaviour only.
7. Tap zones on the Photo widget (the user's default): left third = previous photo, right third = next photo, center = open settings of that widget. Implemented with three transparent overlay views in `widget_photo.xml`, each with its own PendingIntent (`WidgetActions.kt`). The "Left / Center / Right" option stays the default (`tap: 'zones'`).
8. The photo widget has no border or outline. The border option was removed from the photo style tab (`s.border = 'none'` in the demo). Keep it that way natively. Only a shadow toggle remains.
9. Each widget instance has separate settings, keyed by `appWidgetId`. A widget library in the app can hold designs that are not placed yet.
10. The picker must show a preview thumbnail per widget: a text sample, a photo sample and a countdown sample. Use `android:previewLayout` (API 31+) and `android:previewImage` (fallback) in each `*_widget_info.xml`.
11. The photo widget has the sidebar "New widget" buttons in the editor (Text, Countdown, Photos). Keep them.

## 3. Data model (must match the JS in www/index.html)
Text: id, type 'text', text, font, size, weight, italic, color, align, spacing, bgOn, bg, bgOpacity, blur, radius, pad, border, borderColor.
Count: id, type 'count', label, showTitle, target ('YYYY-MM-DDTHH:mm'), units{years,months,weeks,days,hours,minutes,seconds}, font ('seg7' or a font name), size, weight, italic, color, showLabels, seps, bgOn, bg, bgOpacity, blur, radius, pad, border, borderColor.
Photo: id, type 'photo', label, photos[{url,name}], idx, last, paused, auto, interval, shuffle, loop, trans, width, height, fit, radius, opacity, pad, bg, border, borderColor, shadow, brightness, gray, tap ('zones' default), lp, lpTime, plus menu flags mEdit/mNext/mPrev/mShuffle/mPause/mView/mRemove.

In the web demo photos are `URL.createObjectURL` blobs. On Android they must be copied into app storage (`PhotoStorage.kt`, files dir `photos/<widgetId>/`) and referenced by file path. Never keep blob URLs.

Countdown math is in the demo's JS `breakdown()` and `addM()` (month-aware, clamps the day to month length). Port it exactly to `CountdownMath.kt`. The 7-segment polygons are in the JS `SEG` and `DIG` tables. Port them to `SevenSegment.kt` as Paths on a 60x100 grid.

## 4. Planned bridge API (www/bridge.js <-> WidgetBridgePlugin.kt)
- `getWidgets()` returns all widget configs
- `saveWidget({config})` persists it and refreshes the placed widget(s)
- `deleteWidget({id})` removes the config and its photos
- `savePhoto({widgetId, base64, name})` returns `{path}`
- `pinWidget({type})` requests pinning a widget (`AppWidgetManager.requestPinAppWidget`)
- `getLaunchContext()` returns `{appWidgetId, mode}` so the editor opens the right widget when launched from the widget config screen or a center tap

The JS must fall back to the in-memory demo state when `window.Capacitor` is absent so the page can still be opened in a desktop browser.

## 5. File plan and progress
Legend: [ ] todo, [x] done, [B] binary, which a text-only tool cannot create.

- [x] HANDOFF.md
- [x] package.json
- [x] capacitor.config.json
- [x] .gitignore
- [ ] www/index.html (adapt the demo: add bridge calls, remove Google Fonts links, load fonts from bundled files via @font-face, keep all features)
- [ ] www/bridge.js
- [ ] android build files (build.gradle, settings.gradle, gradle.properties, capacitor.settings.gradle, wrapper properties, app/build.gradle, proguard)
- [ ] AndroidManifest.xml (3 receivers, config activity, viewer activity, boot receiver, FileProvider)
- [ ] Kotlin: MainActivity, WidgetBridgePlugin, data/*, render/*, widgets/*, config/*, viewer/*, work/*
- [ ] res layouts (3 widgets + 3 previews), xml widget infos + file_paths, values, launcher icon
- [B] res/drawable/preview_*.png: screenshot or render sample widgets and drop them in
- [B] res/font/*.ttf: download from Google Fonts (Instrument Sans, Fraunces, Playfair Display, Lora, Caveat, Bebas Neue, Space Mono, Orbitron, VT323, Share Tech Mono, Audiowide, Oxanium, Geist Mono). Resource names must be lowercase with underscores, as in the file plan.

## 6. Known risks
- RemoteViews has a bitmap size limit (about 15 MB of memory per update). Downscale to the widget size in dp times density and never push full-resolution photos.
- Large widget Bitmaps: render at the real widget size from `AppWidgetManager.getAppWidgetOptions`.
- Config activity must `setResult(RESULT_OK)` with `EXTRA_APPWIDGET_ID`, or the widget will not be placed.
- Capacitor plugin registration happens in `MainActivity` via `registerPlugin(WidgetBridgePlugin::class.java)` before `super.onCreate`.
- Reschedule updates in `BootReceiver` after a reboot.

## 7. Next step
Write `www/index.html` (adapted demo), then `www/bridge.js`, then the Gradle files, manifest and Kotlin in the order of the file plan. Keep all existing features and the Cream/Dark themes.
