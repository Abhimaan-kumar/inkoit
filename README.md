# Inkoit — Incognito Print

> Secure printing at local print shops. Your file is deleted within 60 seconds of handover.

---

## Quick start

```bash
# 1. Install Flutter deps
flutter pub get

# 2. Connect Firebase (after creating the project)
flutterfire configure

# 3. Run against emulators
firebase emulators:start
flutter run -d chrome          # web / customer
flutter run -d windows         # Windows / vendor
flutter run                    # Android
```

---

## Project structure

```
lib/
  main.dart               Entry point — Firebase init + DI bootstrap
  app.dart                Root widget, GoRouter, BlocProvider<AuthCubit>
  core/
    constants.dart         App-wide numeric/string constants
    result.dart            Result<T> sealed class (Success / Failure)
    di/injector.dart       get_it manual registration
    theme/app_theme.dart   Material 3, deep ink-blue seed
    utils/
      page_range.dart      Parse "1-3,7" page strings
      price.dart           Estimate print cost
      code_generator.dart  Customer codes (6-digit) and shop codes
    widgets/shared_widgets.dart  PrimaryButton, EmptyState, ErrorCard
  features/
    auth/                  Anonymous, Google, email, role chooser
    shop/                  Shop creation, QR, landing page
    customer_print/        Scan, upload, prepare, customise, job status
    vendor_queue/          Live queue, Windows printing, handover
    analytics/             Daily totals, charts, expenses
    saved_shops/           Saved / recent shops for logged-in customers
    owner/                 Super-admin dashboard (Phase 5)
backend/
  functions/src/index.ts  Cloud Functions (TypeScript, 2nd gen)
  converter/server.js     LibreOffice Office→PDF Cloud Run service
firestore.rules            Security rules
storage.rules
firestore.indexes.json
firebase.json              Emulator ports
```

---

## Firebase setup (manual steps required)

1. Create a Firebase project at <https://console.firebase.google.com>.
2. **Switch to the Blaze plan** (Cloud Functions and Cloud Tasks require it). Set a budget alert.
3. Enable these Authentication providers: **Anonymous**, **Google**, **Email/Password**.
4. Run `flutterfire configure` to generate `lib/firebase_options.dart`, then un-comment the two lines in `lib/main.dart`.
5. Deploy security rules and indexes:
   ```bash
   firebase deploy --only firestore:rules,firestore:indexes,storage
   ```
6. Deploy Cloud Functions:
   ```bash
   cd backend/functions && npm ci && npm run build
   firebase deploy --only functions
   ```
7. Deploy the Office converter to Cloud Run:
   ```bash
   cd backend/converter
   gcloud run deploy inkoit-converter \
     --source . \
     --region asia-south1 \
     --no-allow-unauthenticated \
     --set-env-vars GCS_BUCKET=<your-bucket>
   ```
   Set the resulting URL as the `CONVERTER_URL` environment variable in Cloud Functions.
8. Deploy the Flutter web build to Firebase Hosting:
   ```bash
   flutter build web --release
   firebase deploy --only hosting
   ```
9. Configure Android **App Links** (optional for QR deep-linking):
   - Add `inkoit.app` to `AndroidManifest.xml` intent-filter.
   - Upload the `assetlinks.json` to `https://inkoit.app/.well-known/`.
10. Place **SumatraPDF.exe** in `assets/bin/SumatraPDF.exe`.
    > **Note:** SumatraPDF is licensed under GPLv3. Check <https://www.sumatrapdfreader.org/manual.html> before distributing.

---

## Emulator wiring

`firebase.json` configures ports:

| Service   | Port |
|-----------|------|
| Auth      | 9099 |
| Firestore | 8080 |
| Storage   | 9199 |
| Functions | 5001 |
| Hosting   | 5000 |
| UI        | 4000 |

The Flutter app connects to emulators when built in **debug mode** — wire up emulator hosts in `main.dart` after `Firebase.initializeApp()`.

---

## Owner superuser account

`abhimanmandal@gmail.com` has a two-factor login:

1. Normal email+password sign-in.
2. A second prompt appears asking for the **current time in HHMM format** (24-hour). A wrong code triggers immediate sign-out.

The owner can see all vendor accounts, disable/enable shops, and view cross-shop analytics.

---

## Office conversion

The `backend/converter` Cloud Run service uses **LibreOffice headless** to convert Office files (`.doc`, `.docx`, `.xls`, `.xlsx`, `.ppt`, `.pptx`, `.odt`) to PDF. Output may look slightly different from Microsoft Office rendering. The service is private (no public invocations) and called only via service-to-service auth from the `convertToPdf` Cloud Function.

---

## Windows printing

The Windows vendor app bundles **SumatraPDF.exe** in `assets/bin/`. It never opens a viewer — it prints silently with `-silent -print-to <printer>` and deletes the temp file in a `finally` block even on error.

---

## Build phases

| Phase | Contents | Status |
|-------|----------|--------|
| 1 | Setup — packages, DI, router, theme, auth, role chooser | ✅ Complete |
| 2 | Auth & shop — shop creation, QR, lookup by code | ✅ Complete |
| 3 | Customer flow — upload, prepare, preview, submit, job status | ✅ Complete |
| 4 | Vendor flow — live queue, Windows print, handover, Cloud Functions | ✅ Complete |
| 5 | Analytics, expenses, saved shops, owner dashboard | ✅ Complete |
| 6 | Office conversion — converter service + callable + app wiring | ✅ Complete |
| 7 | Polish — tests, README with Decisions, emulator setup, deploy commands | ✅ Complete |

---

## Decisions

| # | Decision |
|---|----------|
| 1 | `AuthCubit` is a lazy singleton (not a factory) so the router's `refreshListenable` can hold a stable reference to the same instance. |
| 2 | The owner's time-PIN is validated in-client (HHmm format). This is intentionally simple — the second factor deters casual access, not dedicated attackers. |
| 3 | `GoRouter` is isolated in `core/router/app_router.dart` and created with `buildAppRouter(authCubit)` so routes are cleanly organized and decoupled from widget state. |
| 4 | Storage rules deny `read` for customers under `jobs/` (vendor-only) but allow customers to `delete` their own file while status is `queued` — matching the cancel-job flow. |
| 5 | `selectedPages` is stored as a string (`"all"`, `"1-3,7"`) so the vendor's Windows print command can pass it directly to SumatraPDF's `-print-settings`. |
| 6 | Analytics writes are Cloud-Function-only (`allow write: if false` in Firestore rules) to prevent client tampering with income totals. |
| 7 | `uuid` package is used for client-side job ID generation so the Storage upload and Firestore write share the same ID without a server round-trip. |
| 8 | If Firestore job document write fails, the uploaded Storage PDF is automatically rolled back (deleted) to prevent orphaned files in Firebase Storage. |
| 9 | `Shop` entity includes `isDisabled` boolean. The app owner can toggle this to stop print requests without deleting the shop record, allowing easy re-enabling later. |
| 10 | Printing on Windows uses conditional imports (`print_service_provider.dart`), ensuring web builds never reference `dart:io` or fail on compilation. |
| 11 | For office document conversion on Web/Mobile, files are uploaded to `tmp/` and converted via a private Cloud Run micro-service using headless LibreOffice with service-to-service authentication. |
