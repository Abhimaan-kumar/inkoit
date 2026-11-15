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
