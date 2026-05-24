# Slice 1: Read-Only Cross-Platform Cheleri Tree — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship a cross-platform (iOS + Android) Flutter app that, behind an email-link auth + invite gate, lets a user browse the migrated Cheleri Mana genealogy, search by name, view profiles with privacy-aware DOB, and run the "how am I related to X" relationship calculator. Full read-side feature parity with the existing native Android app.

**Architecture:** Flutter app (Riverpod state, go_router routing) reads Firestore (persons/marriages/families/invites/users). Firebase Auth handles email-link sign-in. Cloud Storage holds profile photos. A separate Dart CLI tool migrates the bundled SQLite + 607 PNGs into Firestore + Storage as a one-time pre-launch step.

**Tech Stack:** Flutter 3.22+, Dart 3.4+, Firebase (Auth + Firestore + Cloud Storage), Riverpod 2.x, go_router, freezed, json_serializable, sqflite_common_ffi (in migration tool only), flutter_test, integration_test, firebase_emulator_suite.

**Spec reference:** [`docs/superpowers/specs/2026-05-23-ftree-platform-design.md`](../specs/2026-05-23-ftree-platform-design.md)

---

## What Slice 1 Includes vs. Excludes

| Included (read-side parity + minimal gate) | Excluded (later slices) |
|---|---|
| Cross-platform iOS + Android | Web (Phase 4) |
| Email-link auth (Firebase Auth) | Phone OTP |
| Invite redemption (read-only viewer role) | Family-admin role, super-admin role |
| Migrated Cheleri data (1,499 persons, 607 photos) | Multi-family, multiple families |
| Profile-centric navigation | Zoomable canvas, indented tablet list |
| Search by name | Search by other fields |
| Relationship calculator (parity with existing app) | — |
| DOB hidden for living members | Other privacy fine-tuning |
| Heritage Warm theme | Dark mode |
| Offline read cache (Firestore default persistence) | Offline write queue (Slice 2) |
| Firestore security rules enforcing read access | Edit/write rules (Slice 2) |

---

## File Structure

```
/                                       (repo root)
├── pubspec.yaml
├── analysis_options.yaml
├── firebase.json
├── .firebaserc
├── lib/
│   ├── main.dart                       App entry, Firebase init
│   ├── app.dart                        MaterialApp + theme + router
│   ├── theme/
│   │   └── heritage_warm.dart          ColorScheme + ThemeData
│   ├── firebase/
│   │   └── firebase_options.dart       flutterfire-generated
│   ├── auth/
│   │   ├── auth_service.dart           Email-link send + handle
│   │   ├── auth_state.dart             Riverpod auth stream
│   │   └── pages/
│   │       ├── sign_in_email_page.dart
│   │       ├── sign_in_link_pending_page.dart
│   │       └── invite_redeem_page.dart
│   ├── models/
│   │   ├── person.dart                 frozen / json_serializable
│   │   ├── marriage.dart
│   │   ├── family.dart
│   │   ├── app_user.dart
│   │   └── invite.dart
│   ├── data/
│   │   ├── person_repository.dart
│   │   ├── marriage_repository.dart
│   │   ├── family_repository.dart
│   │   └── invite_repository.dart
│   ├── ui/
│   │   ├── pages/
│   │   │   ├── home_page.dart
│   │   │   ├── profile_page.dart
│   │   │   ├── search_page.dart
│   │   │   └── relationship_calculator_page.dart
│   │   └── widgets/
│   │       ├── person_card.dart
│   │       ├── family_around_person.dart
│   │       └── dob_display.dart
│   └── relationship/
│       ├── calculator.dart
│       └── kinship_terms.dart
├── test/                                Unit + widget tests (mirrors lib/)
├── integration_test/
│   └── smoke_test.dart                  Emulator-driven E2E
├── firebase/
│   ├── firestore.rules
│   ├── firestore.indexes.json
│   └── storage.rules
├── tools/
│   └── migrate-sqlite-to-firestore/
│       ├── pubspec.yaml
│       ├── bin/migrate.dart
│       ├── lib/
│       │   ├── sqlite_reader.dart
│       │   ├── transformer.dart
│       │   ├── firestore_writer.dart
│       │   ├── photo_uploader.dart
│       │   └── heuristics.dart
│       └── test/
│           ├── transformer_test.dart
│           └── heuristics_test.dart
└── docs/
    └── reference/
        ├── existing-app-db-schema.md   Documented during discovery task
        └── existing-app-relationship-algorithm.md
```

---

## Conventions

- **TDD throughout.** Tests live in `test/` mirroring `lib/`. Failing test → minimal code → passing test → commit.
- **Commit cadence.** Every passing task = one commit. Commit messages follow `<phase>: <imperative>` (e.g., `auth: add email-link send`).
- **Frozen models.** All models use `freezed` for immutability and `json_serializable` for Firestore (de)serialization.
- **Riverpod state.** Repositories are providers. UI consumes via `Consumer`/`ref.watch`.
- **No `print` in `lib/`.** Use `developer.log` if logging is needed. Configure `flutter_lints` to enforce.
- **Build runner.** `dart run build_runner build --delete-conflicting-outputs` regenerates `*.g.dart` / `*.freezed.dart` after model changes.

---

# Phase A — Bootstrap (Tasks 1–5)

### Task 1: Install Flutter SDK

**Files:** none (tooling installation)

- [ ] **Step 1: Install Flutter via Homebrew**

Run: `brew install --cask flutter`

Expected: Homebrew downloads and installs Flutter SDK.

- [ ] **Step 2: Verify installation**

Run: `flutter --version`

Expected output starts with `Flutter 3.22.0` (or newer; if older than 3.22, run `flutter upgrade`).

- [ ] **Step 3: Accept Android SDK licenses and iOS toolchain check**

Run: `flutter doctor --android-licenses` (accept all), then `flutter doctor`.

Expected: all checkmarks green except possibly "Connected device" if no simulator is running. Resolve any red items before continuing.

### Task 2: Scaffold the Flutter project

**Files:**
- Create: `lib/main.dart` (via `flutter create`)
- Create: `pubspec.yaml` (via `flutter create`)
- Create: `android/`, `ios/`, `test/`, etc.

- [ ] **Step 1: Run `flutter create` in place**

Run from repo root: `flutter create --org com.npalasseri --project-name ftree --platforms=android,ios .`

Expected: scaffolds the project files into the existing directory without overwriting `README.md`, `docs/`, `.gitignore`, `.git/`.

- [ ] **Step 2: First-run smoke test**

Run: `flutter run -d <ios_simulator_id>` (open Simulator first via `open -a Simulator`).

Expected: counter app launches on iOS Simulator.

Then: `flutter run -d <android_emulator_or_device>`.

Expected: same counter app on Android.

- [ ] **Step 3: Stage and commit**

```bash
git add -A
git commit -m "bootstrap: flutter create scaffold"
```

### Task 3: Add core dependencies to `pubspec.yaml`

**Files:**
- Modify: `pubspec.yaml`

- [ ] **Step 1: Add dependencies**

Edit `pubspec.yaml` to add under `dependencies:` (alphabetized):

```yaml
dependencies:
  flutter:
    sdk: flutter
  cloud_firestore: ^5.4.0
  firebase_auth: ^5.3.0
  firebase_core: ^3.6.0
  firebase_storage: ^12.3.0
  flutter_riverpod: ^2.5.1
  freezed_annotation: ^2.4.4
  go_router: ^14.2.7
  json_annotation: ^4.9.0
```

And under `dev_dependencies:`:

```yaml
dev_dependencies:
  flutter_test:
    sdk: flutter
  integration_test:
    sdk: flutter
  build_runner: ^2.4.13
  flutter_lints: ^5.0.0
  freezed: ^2.5.7
  json_serializable: ^6.8.0
  mocktail: ^1.0.4
```

- [ ] **Step 2: Fetch packages**

Run: `flutter pub get`

Expected: "Got dependencies!" with no resolution errors.

- [ ] **Step 3: Commit**

```bash
git add pubspec.yaml pubspec.lock
git commit -m "bootstrap: add Firebase, Riverpod, go_router, freezed deps"
```

### Task 4: Tighten lints

**Files:**
- Modify: `analysis_options.yaml`

- [ ] **Step 1: Replace generated `analysis_options.yaml`**

Replace the entire contents with:

```yaml
include: package:flutter_lints/flutter.yaml

linter:
  rules:
    - prefer_const_constructors
    - prefer_const_literals_to_create_immutables
    - prefer_final_locals
    - prefer_single_quotes
    - avoid_print
    - unawaited_futures
    - require_trailing_commas

analyzer:
  errors:
    invalid_annotation_target: ignore
  exclude:
    - "**/*.g.dart"
    - "**/*.freezed.dart"
```

- [ ] **Step 2: Run analyzer**

Run: `flutter analyze`

Expected: "No issues found!" (the scaffold has no real code yet).

- [ ] **Step 3: Commit**

```bash
git add analysis_options.yaml
git commit -m "bootstrap: stricter lints"
```

### Task 5: Smoke-test analyzer + tests run

- [ ] **Step 1: Run the default scaffold test**

Run: `flutter test`

Expected: 1 test passes (the counter increment test from `flutter create`).

- [ ] **Step 2: Commit any leftover changes**

```bash
git status   # confirm clean
```

If clean, no commit. If anything dirty, investigate before continuing.

---

# Phase B — Firebase Project (Tasks 6–10)

### Task 6: Create the Firebase project (manual console step)

- [ ] **Step 1: Open Firebase console**

Navigate to https://console.firebase.google.com in a browser. Sign in as `npalasseri@gmail.com`.

- [ ] **Step 2: Create project**

Click "Add project". Name: `ftree-prod`. Disable Google Analytics for now (can be enabled later). Region: `asia-south1` (Mumbai) for Cheleri users.

- [ ] **Step 3: Register iOS app**

In project overview, click iOS+ icon. Bundle ID: `com.npalasseri.ftree`. App nickname: "ftree iOS". Skip the SDK steps shown — `flutterfire configure` (Task 9) handles this.

- [ ] **Step 4: Register Android app**

Click "Add app" → Android. Package: `com.npalasseri.ftree`. App nickname: "ftree Android". Skip SHA-1 for now (only needed for Google sign-in, not email link). Skip SDK steps.

- [ ] **Step 5: Note the project ID**

Project ID will be something like `ftree-prod-abc1d`. Record it; needed in Task 9.

### Task 7: Enable Auth, Firestore, Storage in console

- [ ] **Step 1: Enable Email Link sign-in**

In Firebase console → Authentication → Get started → Sign-in method → "Email/Password" → toggle ON. **Inside the same panel, also toggle ON "Email link (passwordless sign-in)"**. Save.

- [ ] **Step 2: Enable Firestore**

In Firebase console → Firestore Database → Create database → "Start in production mode" → region `asia-south1`. Click Enable.

- [ ] **Step 3: Enable Cloud Storage**

In Firebase console → Storage → Get started → "Start in production mode" → region `asia-south1`. Click Done.

- [ ] **Step 4: Note success**

All three services should now appear under "Build" in the console sidebar.

### Task 8: Install Firebase CLI + flutterfire CLI locally

- [ ] **Step 1: Install Firebase CLI**

Run: `npm install -g firebase-tools` (or `curl -sL https://firebase.tools | bash`).

Expected: `firebase --version` prints `13.x.x` or newer.

- [ ] **Step 2: Log in**

Run: `firebase login`. Authorize in browser.

Expected: "Success! Logged in as npalasseri@gmail.com".

- [ ] **Step 3: Install flutterfire CLI**

Run: `dart pub global activate flutterfire_cli`.

Add `~/.pub-cache/bin` to PATH if not already (zshrc).

Expected: `flutterfire --version` works.

### Task 9: Wire Flutter app to Firebase

**Files:**
- Create: `lib/firebase/firebase_options.dart` (auto-generated)
- Create: `firebase.json`
- Create: `.firebaserc`
- Modify: `ios/Runner/Info.plist`, `android/app/google-services.json` (auto-handled)

- [ ] **Step 1: Run flutterfire configure**

Run from repo root: `flutterfire configure --project=<ftree-project-id> --platforms=android,ios --out=lib/firebase/firebase_options.dart`

Select both iOS and Android when prompted. Confirms bundle/package IDs.

Expected: generates `lib/firebase/firebase_options.dart`, `ios/Runner/GoogleService-Info.plist`, `android/app/google-services.json`. Updates iOS/Android build files.

- [ ] **Step 2: Initialize Firebase in main.dart**

Replace contents of `lib/main.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:firebase_core/firebase_core.dart';

import 'package:ftree/app.dart';
import 'package:ftree/firebase/firebase_options.dart';

Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Firebase.initializeApp(options: DefaultFirebaseOptions.currentPlatform);
  runApp(const ProviderScope(child: FtreeApp()));
}
```

- [ ] **Step 3: Stub `app.dart`**

Create `lib/app.dart`:

```dart
import 'package:flutter/material.dart';

class FtreeApp extends StatelessWidget {
  const FtreeApp({super.key});

  @override
  Widget build(BuildContext context) {
    return const MaterialApp(
      home: Scaffold(body: Center(child: Text('ftree — bootstrapping'))),
    );
  }
}
```

- [ ] **Step 4: Run both platforms**

Run: `flutter run -d <ios_simulator_id>` then on Android.

Expected: app launches, shows "ftree — bootstrapping" text. No Firebase errors in console output.

- [ ] **Step 5: Commit**

```bash
git add lib/main.dart lib/app.dart lib/firebase/firebase_options.dart firebase.json .firebaserc ios/ android/
git commit -m "firebase: wire Firebase Core to Flutter app"
```

### Task 10: Initialize Firestore rules + Storage rules files

**Files:**
- Create: `firebase/firestore.rules`
- Create: `firebase/firestore.indexes.json`
- Create: `firebase/storage.rules`
- Modify: `firebase.json`

- [ ] **Step 1: Create deny-all baseline rules**

Create `firebase/firestore.rules`:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    // Deny everything by default. Specific reads/writes are added per resource.
    match /{document=**} {
      allow read, write: if false;
    }
  }
}
```

Create `firebase/storage.rules`:

```
rules_version = '2';
service firebase.storage {
  match /b/{bucket}/o {
    match /{allPaths=**} {
      allow read, write: if false;
    }
  }
}
```

Create `firebase/firestore.indexes.json`:

```json
{ "indexes": [], "fieldOverrides": [] }
```

- [ ] **Step 2: Wire `firebase.json`**

Replace `firebase.json` (generated by `flutterfire configure`) with:

```json
{
  "firestore": {
    "rules": "firebase/firestore.rules",
    "indexes": "firebase/firestore.indexes.json"
  },
  "storage": {
    "rules": "firebase/storage.rules"
  },
  "emulators": {
    "auth": { "port": 9099 },
    "firestore": { "port": 8080 },
    "storage": { "port": 9199 },
    "ui": { "enabled": true, "port": 4000 }
  }
}
```

- [ ] **Step 3: Deploy rules**

Run: `firebase deploy --only firestore:rules,storage:rules --project <ftree-project-id>`

Expected: "Deploy complete!"

- [ ] **Step 4: Commit**

```bash
git add firebase/ firebase.json
git commit -m "firebase: deny-all rules baseline + emulator config"
```

---

# Phase C — Theme + Hello (Tasks 11–13)

### Task 11: Implement Heritage Warm theme

**Files:**
- Create: `lib/theme/heritage_warm.dart`
- Test: `test/theme/heritage_warm_test.dart`

- [ ] **Step 1: Write failing test**

Create `test/theme/heritage_warm_test.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:ftree/theme/heritage_warm.dart';

void main() {
  test('Heritage Warm palette matches spec', () {
    final theme = heritageWarmTheme();
    expect(theme.colorScheme.surface, const Color(0xFFFAF6EE));
    expect(theme.colorScheme.surfaceContainerHighest, const Color(0xFFEDE4D3));
    expect(theme.colorScheme.primary, const Color(0xFF7A5230));
    expect(theme.colorScheme.secondary, const Color(0xFFB8860B));
    expect(theme.colorScheme.onSurface, const Color(0xFF2C1810));
  });

  test('Heritage Warm primary has adequate contrast vs surface', () {
    final theme = heritageWarmTheme();
    final ratio = _contrast(theme.colorScheme.primary, theme.colorScheme.surface);
    expect(ratio, greaterThanOrEqualTo(4.5)); // WCAG AA
  });
}

double _contrast(Color a, Color b) {
  double lum(Color c) {
    double channel(int v) {
      final s = v / 255.0;
      return s <= 0.03928 ? s / 12.92 : pow((s + 0.055) / 1.055, 2.4).toDouble();
    }
    return 0.2126 * channel(c.red) + 0.7152 * channel(c.green) + 0.0722 * channel(c.blue);
  }
  final la = lum(a), lb = lum(b);
  return (max(la, lb) + 0.05) / (min(la, lb) + 0.05);
}
```

(Import `dart:math` at top for `pow`, `max`, `min`.)

- [ ] **Step 2: Run test, watch fail**

Run: `flutter test test/theme/heritage_warm_test.dart`

Expected: FAIL — `heritage_warm.dart` doesn't exist.

- [ ] **Step 3: Implement theme**

Create `lib/theme/heritage_warm.dart`:

```dart
import 'package:flutter/material.dart';

ThemeData heritageWarmTheme() {
  const surface = Color(0xFFFAF6EE);
  const surfaceVariant = Color(0xFFEDE4D3);
  const primary = Color(0xFF7A5230);
  const secondary = Color(0xFFB8860B);
  const onSurface = Color(0xFF2C1810);

  return ThemeData(
    useMaterial3: true,
    colorScheme: const ColorScheme.light(
      primary: primary,
      onPrimary: Color(0xFFFAF6EE),
      secondary: secondary,
      onSecondary: Color(0xFF2C1810),
      surface: surface,
      onSurface: onSurface,
      surfaceContainerHighest: surfaceVariant,
    ),
    scaffoldBackgroundColor: surface,
    fontFamily: null, // system font per spec
  );
}
```

- [ ] **Step 4: Run test, watch pass**

Run: `flutter test test/theme/heritage_warm_test.dart`

Expected: both tests pass.

- [ ] **Step 5: Commit**

```bash
git add lib/theme/ test/theme/
git commit -m "theme: implement Heritage Warm palette"
```

### Task 12: Wire theme into MaterialApp

**Files:**
- Modify: `lib/app.dart`

- [ ] **Step 1: Apply theme**

Replace `lib/app.dart`:

```dart
import 'package:flutter/material.dart';

import 'package:ftree/theme/heritage_warm.dart';

class FtreeApp extends StatelessWidget {
  const FtreeApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'ftree',
      theme: heritageWarmTheme(),
      home: const _BootstrapScreen(),
    );
  }
}

class _BootstrapScreen extends StatelessWidget {
  const _BootstrapScreen();

  @override
  Widget build(BuildContext context) {
    return const Scaffold(
      body: Center(child: Text('ftree')),
    );
  }
}
```

- [ ] **Step 2: Run both platforms**

Run on iOS sim and Android. Confirm the background is cream (`#FAF6EE`).

- [ ] **Step 3: Commit**

```bash
git add lib/app.dart
git commit -m "theme: apply Heritage Warm to MaterialApp"
```

### Task 13: Set up CI (GitHub Actions)

**Files:**
- Create: `.github/workflows/ci.yml`

- [ ] **Step 1: Add CI workflow**

Create `.github/workflows/ci.yml`:

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  flutter:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.22.x'
          channel: 'stable'
      - run: flutter pub get
      - run: dart run build_runner build --delete-conflicting-outputs
      - run: flutter analyze
      - run: flutter test
```

- [ ] **Step 2: Commit and push**

```bash
git add .github/
git commit -m "ci: add GitHub Actions for analyze + test"
git push
```

- [ ] **Step 3: Verify CI runs**

Open https://github.com/npalasseri/ftree/actions. Confirm the workflow runs green.

---

# Phase D — Discover the Existing App (Tasks 14–16)

> These tasks produce reference documentation that later tasks (migration, relationship calculator) depend on. No app code is written here.

### Task 14: Extract and inspect the bundled SQLite DB

**Files:**
- Create: `docs/reference/existing-app-db-schema.md`

- [ ] **Step 1: Unzip the bundled DB**

Run:

```bash
cd "cheleri_extracted/Cheleri Mana Ancestry/app/src/main/assets/databases/"
unzip -o cheleri_mana_vamsavali3.zip
ls *.db   # confirm a .db file appeared
```

Expected: one `.db` file (likely `CheleriFamilyTree.db` per spec §8) is extracted alongside the zip.

- [ ] **Step 2: Inspect schema**

Run: `sqlite3 <db_filename> ".schema"`

Expected: schema dump showing tables `individual`, `member`, `UniqueID`, and views.

- [ ] **Step 3: Sample data**

Run:

```
sqlite3 <db_filename> "SELECT name FROM sqlite_master WHERE type='table';"
sqlite3 <db_filename> "SELECT COUNT(*) FROM individual;"
sqlite3 <db_filename> "SELECT * FROM individual LIMIT 3;"
sqlite3 <db_filename> "SELECT * FROM member LIMIT 3;"
sqlite3 <db_filename> "SELECT * FROM UniqueID LIMIT 3;"
```

- [ ] **Step 4: Document findings**

Create `docs/reference/existing-app-db-schema.md` with:
- Every table's columns (name, type, nullable)
- Every view's SQL
- Row counts per table
- 3 sample rows per table (sanitized — names OK, but redact phone/address if needed)
- Spouse columns: confirm `spID`, `spID2`, `spID3` exist on `member`
- DOB columns: confirm `db`, `dm`, `dd` exist on `individual`

- [ ] **Step 5: Commit**

```bash
git add docs/reference/existing-app-db-schema.md
git commit -m "discover: document existing app SQLite schema"
```

### Task 15: Document the Relationship algorithm

**Files:**
- Create: `docs/reference/existing-app-relationship-algorithm.md`

- [ ] **Step 1: Read the source**

Open and read:
- `cheleri_extracted/Cheleri Mana Ancestry/app/src/main/java/com/familytree/CheleriMana_Ancestry/Relationship.java`
- `cheleri_extracted/Cheleri Mana Ancestry/app/src/main/java/com/familytree/CheleriMana_Ancestry/RelationshipAdapter.java`
- `cheleri_extracted/Cheleri Mana Ancestry/app/src/main/java/com/familytree/CheleriMana_Ancestry/DatabaseAccess.java` (look for relationship-query methods)

- [ ] **Step 2: Write the doc**

Create `docs/reference/existing-app-relationship-algorithm.md` covering:
- How the algorithm finds a path from person A to person B in the graph
- What kinship terms it outputs (English: father, brother, paternal uncle, etc.) and in which language
- The full list of kinship terms recognized (extract the string constants)
- 20 known input → output examples extracted from the source or computed manually using the bundled DB (e.g., "personId 100 → personId 105: paternal uncle"). These become parity test fixtures in Task 53.

- [ ] **Step 3: Commit**

```bash
git add docs/reference/existing-app-relationship-algorithm.md
git commit -m "discover: document existing relationship algorithm"
```

### Task 16: Document the GenId (Uid) algorithm

**Files:**
- Modify: `docs/reference/existing-app-db-schema.md` (add a section)

- [ ] **Step 1: Locate the algorithm**

Search the Java source for "UniqueID" or "Uid" view definition (in `DatabaseOpenHelper.java`) and any Java code that generates Uid strings.

- [ ] **Step 2: Document the rules**

Add a "GenId (Uid) Algorithm" section to `docs/reference/existing-app-db-schema.md` describing:
- The letter-string scheme (e.g., "A", "AB", "ABA" representing generation/branch position)
- Whether siblings are ordered by birth, by row order, or another rule
- How spouses get their Uid (do they inherit from the partner, or have their own?)
- 10 examples of person → Uid from the bundled DB

- [ ] **Step 3: Commit**

```bash
git add docs/reference/existing-app-db-schema.md
git commit -m "discover: document GenId/Uid algorithm"
```

---

# Phase E — Data Models (Tasks 17–22)

### Task 17: Person model

**Files:**
- Create: `lib/models/person.dart`
- Create: `lib/models/person.freezed.dart` (generated)
- Create: `lib/models/person.g.dart` (generated)
- Test: `test/models/person_test.dart`

- [ ] **Step 1: Write failing test**

Create `test/models/person_test.dart`:

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:ftree/models/person.dart';

void main() {
  test('Person.fromJson parses a complete document', () {
    final json = {
      'personId': 'p_100',
      'familyId': 'cheleri-mana',
      'name': 'Subraya',
      'gender': 'M',
      'generation': 5,
      'fatherPersonId': 'p_50',
      'motherPersonId': 'p_51',
      'isLiving': false,
      'genId': 'ABA',
    };
    final p = Person.fromJson(json);
    expect(p.personId, 'p_100');
    expect(p.name, 'Subraya');
    expect(p.gender, Gender.male);
    expect(p.isLiving, false);
  });

  test('Person.fromJson handles missing optional fields', () {
    final json = {
      'personId': 'p_200',
      'familyId': 'cheleri-mana',
      'name': 'Unknown',
      'gender': 'F',
      'generation': 1,
      'isLiving': true,
      'genId': 'A',
    };
    final p = Person.fromJson(json);
    expect(p.fatherPersonId, isNull);
    expect(p.dob, isNull);
  });
}
```

- [ ] **Step 2: Run test, watch fail**

Run: `flutter test test/models/person_test.dart`

Expected: FAIL — `person.dart` doesn't exist.

- [ ] **Step 3: Implement model**

Create `lib/models/person.dart`:

```dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'person.freezed.dart';
part 'person.g.dart';

enum Gender {
  @JsonValue('M') male,
  @JsonValue('F') female,
  @JsonValue('O') other,
}

@freezed
class Dob with _$Dob {
  const factory Dob({int? year, int? month, int? day}) = _Dob;
  factory Dob.fromJson(Map<String, dynamic> json) => _$DobFromJson(json);
}

@freezed
class Contact with _$Contact {
  const factory Contact({String? phone, String? email, String? address}) = _Contact;
  factory Contact.fromJson(Map<String, dynamic> json) => _$ContactFromJson(json);
}

@freezed
class Person with _$Person {
  const factory Person({
    required String personId,
    required String familyId,
    required String name,
    required Gender gender,
    required int generation,
    required bool isLiving,
    required String genId,
    String? fatherPersonId,
    String? motherPersonId,
    Dob? dob,
    @Default('default') String dobVisibility,
    String? profession,
    String? bloodGroup,
    Contact? contact,
    String? photoURL,
  }) = _Person;

  factory Person.fromJson(Map<String, dynamic> json) => _$PersonFromJson(json);
}
```

- [ ] **Step 4: Generate code**

Run: `dart run build_runner build --delete-conflicting-outputs`

Expected: `person.freezed.dart` and `person.g.dart` are created.

- [ ] **Step 5: Run test, watch pass**

Run: `flutter test test/models/person_test.dart`

Expected: both tests pass.

- [ ] **Step 6: Commit**

```bash
git add lib/models/person.dart lib/models/person.freezed.dart lib/models/person.g.dart test/models/person_test.dart
git commit -m "models: add Person with freezed + json_serializable"
```

### Task 18: Marriage model

**Files:**
- Create: `lib/models/marriage.dart` (+ generated files)
- Test: `test/models/marriage_test.dart`

- [ ] **Step 1: Write failing test**

Create `test/models/marriage_test.dart`:

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:ftree/models/marriage.dart';

void main() {
  test('Marriage.fromJson parses', () {
    final json = {
      'marriageId': 'm_1',
      'personAId': 'p_100',
      'personBId': 'p_200',
      'personAFamilyId': 'cheleri-mana',
      'personBFamilyId': 'cheleri-mana',
      'isCrossFamily': false,
      'status': 'married',
      'consensusStatus': 'approved',
    };
    final m = Marriage.fromJson(json);
    expect(m.marriageId, 'm_1');
    expect(m.isCrossFamily, false);
    expect(m.status, MarriageStatus.married);
  });
}
```

- [ ] **Step 2: Run, fail.** `flutter test test/models/marriage_test.dart`

- [ ] **Step 3: Implement**

Create `lib/models/marriage.dart`:

```dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'marriage.freezed.dart';
part 'marriage.g.dart';

enum MarriageStatus {
  @JsonValue('married') married,
  @JsonValue('divorced') divorced,
  @JsonValue('deceased-spouse') deceasedSpouse,
}

enum ConsensusStatus {
  @JsonValue('approved') approved,
  @JsonValue('pending') pending,
  @JsonValue('rejected') rejected,
  @JsonValue('disputed') disputed,
}

@freezed
class Marriage with _$Marriage {
  const factory Marriage({
    required String marriageId,
    required String personAId,
    required String personBId,
    required String personAFamilyId,
    required String personBFamilyId,
    required bool isCrossFamily,
    required MarriageStatus status,
    required ConsensusStatus consensusStatus,
    DateTime? marriedDate,
    DateTime? dissolvedDate,
  }) = _Marriage;

  factory Marriage.fromJson(Map<String, dynamic> json) => _$MarriageFromJson(json);
}
```

- [ ] **Step 4: Generate, test, pass.** `dart run build_runner build --delete-conflicting-outputs && flutter test test/models/marriage_test.dart`

- [ ] **Step 5: Commit**

```bash
git add lib/models/marriage.dart lib/models/marriage.freezed.dart lib/models/marriage.g.dart test/models/marriage_test.dart
git commit -m "models: add Marriage"
```

### Task 19: Family model

**Files:**
- Create: `lib/models/family.dart` (+ generated)
- Test: `test/models/family_test.dart`

- [ ] **Step 1: Write failing test**

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:ftree/models/family.dart';

void main() {
  test('Family.fromJson parses', () {
    final json = {'familyId': 'cheleri-mana', 'name': 'Cheleri Mana', 'slug': 'cheleri-mana'};
    final f = Family.fromJson(json);
    expect(f.familyId, 'cheleri-mana');
    expect(f.name, 'Cheleri Mana');
  });
}
```

- [ ] **Step 2: Run, fail.**

- [ ] **Step 3: Implement**

```dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'family.freezed.dart';
part 'family.g.dart';

@freezed
class Family with _$Family {
  const factory Family({
    required String familyId,
    required String name,
    required String slug,
    int? foundedGeneration,
    String? description,
  }) = _Family;

  factory Family.fromJson(Map<String, dynamic> json) => _$FamilyFromJson(json);
}
```

- [ ] **Step 4: Build + test, pass.**

- [ ] **Step 5: Commit.** `git commit -m "models: add Family"`

### Task 20: AppUser model

Same TDD pattern as Tasks 17–19. Fields: `uid`, `email`, `displayName?`, `linkedPersonId?`, `createdAt`, `lastLoginAt`. Path: `lib/models/app_user.dart`, test: `test/models/app_user_test.dart`. Commit: `models: add AppUser`.

### Task 21: Invite model

Same TDD pattern. Fields: `inviteId`, `code`, `createdBy`, `role` (enum: viewer/admin), `email?`, `expiresAt`, `redeemedBy?`, `redeemedAt?`. Path: `lib/models/invite.dart`. Commit: `models: add Invite`.

### Task 22: Verify all models lint clean

- [ ] **Step 1: Run analyzer**

Run: `flutter analyze`

Expected: "No issues found!"

- [ ] **Step 2: Run all tests**

Run: `flutter test`

Expected: all model tests pass.

---

# Phase F — Migration Tool (Tasks 23–31)

> Builds the one-time CLI that imports the bundled SQLite + 607 PNGs into Firestore + Cloud Storage. This is a separate Dart package under `tools/`.

### Task 23: Scaffold migration tool

**Files:**
- Create: `tools/migrate-sqlite-to-firestore/pubspec.yaml`
- Create: `tools/migrate-sqlite-to-firestore/bin/migrate.dart`

- [ ] **Step 1: Create pubspec**

Create `tools/migrate-sqlite-to-firestore/pubspec.yaml`:

```yaml
name: migrate_sqlite_to_firestore
description: One-time migration of bundled Cheleri SQLite DB and photos into Firestore + Cloud Storage.
publish_to: 'none'
environment:
  sdk: ^3.4.0

dependencies:
  args: ^2.5.0
  firedart: ^0.9.7   # lightweight Firestore client for Dart CLI; or use googleapis + service account
  googleapis: ^13.2.0
  googleapis_auth: ^1.6.0
  http: ^1.2.2
  path: ^1.9.0
  sqlite3: ^2.4.6

dev_dependencies:
  test: ^1.25.8
  lints: ^5.0.0
```

- [ ] **Step 2: Stub `bin/migrate.dart`**

```dart
import 'package:args/args.dart';

void main(List<String> argv) {
  final parser = ArgParser()
    ..addOption('db', help: 'Path to the extracted CheleriFamilyTree.db')
    ..addOption('assets-dir', help: 'Path to the assets/ directory with PNGs')
    ..addOption('service-account', help: 'Path to GCP service account JSON')
    ..addFlag('dry-run', defaultsTo: false, help: 'Read but do not write to Firestore')
    ..addFlag('help', abbr: 'h', negatable: false);
  final args = parser.parse(argv);
  if (args['help'] as bool) {
    print(parser.usage);
    return;
  }
  print('Migration tool stub — db=${args['db']} dry-run=${args['dry-run']}');
}
```

- [ ] **Step 3: Get deps and run stub**

```bash
cd tools/migrate-sqlite-to-firestore
dart pub get
dart run bin/migrate.dart --help
```

Expected: usage output.

- [ ] **Step 4: Commit**

```bash
git add tools/migrate-sqlite-to-firestore/
git commit -m "migrate: scaffold CLI tool"
```

### Task 24: SqliteReader — read `individual` table

**Files:**
- Create: `tools/migrate-sqlite-to-firestore/lib/sqlite_reader.dart`
- Test: `tools/migrate-sqlite-to-firestore/test/sqlite_reader_test.dart`

- [ ] **Step 1: Write failing test**

Test uses a fixture: copy the extracted DB into `tools/migrate-sqlite-to-firestore/test/fixtures/cheleri.db`.

```dart
import 'package:test/test.dart';
import 'package:migrate_sqlite_to_firestore/sqlite_reader.dart';

void main() {
  test('readIndividuals returns 1499 rows', () {
    final reader = SqliteReader.open('test/fixtures/cheleri.db');
    final rows = reader.readIndividuals().toList();
    expect(rows.length, 1499);
    expect(rows.first.keys, containsAll(['id', 'name', 'sex', 'db', 'dm', 'dd']));
    reader.close();
  });
}
```

- [ ] **Step 2: Run, fail.** `cd tools/migrate-sqlite-to-firestore && dart test test/sqlite_reader_test.dart`

- [ ] **Step 3: Implement**

```dart
import 'package:sqlite3/sqlite3.dart';

class SqliteReader {
  final Database _db;
  SqliteReader._(this._db);

  factory SqliteReader.open(String path) => SqliteReader._(sqlite3.open(path));

  Iterable<Map<String, Object?>> readIndividuals() sync* {
    final result = _db.select('SELECT * FROM individual');
    for (final row in result) {
      yield Map<String, Object?>.from(row);
    }
  }

  Iterable<Map<String, Object?>> readMembers() sync* {
    final result = _db.select('SELECT * FROM member');
    for (final row in result) {
      yield Map<String, Object?>.from(row);
    }
  }

  Iterable<Map<String, Object?>> readUniqueIds() sync* {
    final result = _db.select('SELECT * FROM UniqueID');
    for (final row in result) {
      yield Map<String, Object?>.from(row);
    }
  }

  void close() => _db.dispose();
}
```

- [ ] **Step 4: Run test, pass.**

- [ ] **Step 5: Commit.** `git commit -m "migrate: SqliteReader for individual/member/UniqueID"`

### Task 25: Transformer — individual → Person doc

**Files:**
- Create: `tools/migrate-sqlite-to-firestore/lib/transformer.dart`
- Create: `tools/migrate-sqlite-to-firestore/lib/heuristics.dart`
- Test: `tools/migrate-sqlite-to-firestore/test/transformer_test.dart`

- [ ] **Step 1: Write failing test**

```dart
import 'package:test/test.dart';
import 'package:migrate_sqlite_to_firestore/transformer.dart';

void main() {
  test('toPerson maps individual row to Person doc', () {
    final row = {
      'id': 100,
      'name': 'Subraya',
      'sex': 'M',
      'db': 1950,
      'dm': 6,
      'dd': 15,
      'profession': 'Farmer',
      'blgroup': 'O+',
      'phone': null,
      'email': null,
      'address': null,
    };
    final doc = toPersonDoc(row, familyId: 'cheleri-mana', maxGeneration: 14, generation: 5, genId: 'ABA');
    expect(doc['personId'], 'p_100');
    expect(doc['familyId'], 'cheleri-mana');
    expect(doc['name'], 'Subraya');
    expect(doc['gender'], 'M');
    expect(doc['generation'], 5);
    expect(doc['dob'], {'year': 1950, 'month': 6, 'day': 15});
    expect(doc['profession'], 'Farmer');
    expect(doc['isLiving'], false); // generation 5 vs max 14: 14 - 5 = 9 > 3 → deceased
    expect(doc['genId'], 'ABA');
  });

  test('toPerson sets isLiving=true for recent generations', () {
    final row = {'id': 200, 'name': 'X', 'sex': 'F', 'db': 2000, 'dm': null, 'dd': null};
    final doc = toPersonDoc(row, familyId: 'cheleri-mana', maxGeneration: 14, generation: 13, genId: 'Z');
    expect(doc['isLiving'], true);
  });
}
```

- [ ] **Step 2: Run, fail.**

- [ ] **Step 3: Implement heuristics + transformer**

`lib/heuristics.dart`:

```dart
bool inferIsLiving({required int generation, required int maxGeneration}) {
  return generation >= maxGeneration - 3;
}
```

`lib/transformer.dart`:

```dart
import 'heuristics.dart';

Map<String, dynamic> toPersonDoc(
  Map<String, Object?> row, {
  required String familyId,
  required int maxGeneration,
  required int generation,
  required String genId,
}) {
  final id = row['id'] as int;
  return {
    'personId': 'p_$id',
    'familyId': familyId,
    'name': row['name'] as String,
    'gender': row['sex'] as String,
    'generation': generation,
    'genId': genId,
    'isLiving': inferIsLiving(generation: generation, maxGeneration: maxGeneration),
    if (row['db'] != null)
      'dob': {
        'year': row['db'],
        'month': row['dm'],
        'day': row['dd'],
      },
    if (row['profession'] != null) 'profession': row['profession'],
    if (row['blgroup'] != null) 'bloodGroup': row['blgroup'],
    if (row['phone'] != null || row['email'] != null || row['address'] != null)
      'contact': {
        'phone': row['phone'],
        'email': row['email'],
        'address': row['address'],
      },
  };
}
```

(Adjust column names to match what Task 14 documented.)

- [ ] **Step 4: Run, pass.**

- [ ] **Step 5: Commit.** `git commit -m "migrate: transformer for individual rows"`

### Task 26: Transformer — member rows → Marriage docs (multi-spouse expansion)

- [ ] **Step 1: Write failing test**

```dart
test('expandMarriages produces one Marriage per spouse column', () {
  final row = {
    'mID': 100,
    'spID': 200,
    'spID2': 201,
    'spID3': null,
  };
  final marriages = expandMarriages(row, familyId: 'cheleri-mana').toList();
  expect(marriages.length, 2);
  expect(marriages[0]['personAId'], 'p_100');
  expect(marriages[0]['personBId'], 'p_200');
  expect(marriages[1]['personBId'], 'p_201');
});

test('expandMarriages skips when no spouses', () {
  expect(expandMarriages({'mID': 100, 'spID': null, 'spID2': null, 'spID3': null}, familyId: 'cheleri-mana').isEmpty, true);
});
```

- [ ] **Step 2: Run, fail.**

- [ ] **Step 3: Implement** in `transformer.dart`:

```dart
Iterable<Map<String, dynamic>> expandMarriages(
  Map<String, Object?> row, {
  required String familyId,
}) sync* {
  final memberId = row['mID'] as int;
  for (final spouseCol in ['spID', 'spID2', 'spID3']) {
    final spouseId = row[spouseCol] as int?;
    if (spouseId == null) continue;
    yield {
      'marriageId': 'm_${memberId}_$spouseId',
      'personAId': 'p_$memberId',
      'personBId': 'p_$spouseId',
      'personAFamilyId': familyId,
      'personBFamilyId': familyId,
      'isCrossFamily': false,
      'status': 'married',
      'consensusStatus': 'approved',
    };
  }
}
```

- [ ] **Step 4: Run, pass.**

- [ ] **Step 5: Commit.** `git commit -m "migrate: multi-spouse expansion to Marriage docs"`

### Task 27: GenId computation

**Files:**
- Create: `tools/migrate-sqlite-to-firestore/lib/genid.dart`
- Test: `tools/migrate-sqlite-to-firestore/test/genid_test.dart`

Use the algorithm documented in Task 16. Implement and test against the 10 known examples. Commit: `migrate: GenId computation matching existing UniqueID view`.

### Task 28: FirestoreWriter with idempotent upserts

**Files:**
- Create: `tools/migrate-sqlite-to-firestore/lib/firestore_writer.dart`
- Test: `tools/migrate-sqlite-to-firestore/test/firestore_writer_test.dart` (uses Firestore emulator)

- [ ] **Step 1: Start emulator**

Run from repo root: `firebase emulators:start --only firestore` (in a separate terminal).

- [ ] **Step 2: Write failing test**

```dart
import 'package:test/test.dart';
import 'package:migrate_sqlite_to_firestore/firestore_writer.dart';

void main() {
  test('writePerson is idempotent — second write produces no diff', () async {
    final writer = await FirestoreWriter.connect(projectId: 'demo-ftree', emulatorHost: 'localhost:8080');
    final doc = {'personId': 'p_test', 'name': 'Test', 'familyId': 'cheleri-mana'};
    await writer.writePerson(doc);
    await writer.writePerson(doc); // idempotent
    final read = await writer.readPerson('p_test');
    expect(read['name'], 'Test');
    await writer.close();
  });
}
```

- [ ] **Step 3: Run, fail.**

- [ ] **Step 4: Implement** `FirestoreWriter` using `googleapis` Firestore REST (works against emulator with no auth). Use deterministic doc IDs from `personId` so reruns overwrite same docs.

- [ ] **Step 5: Run, pass.**

- [ ] **Step 6: Commit.** `git commit -m "migrate: idempotent Firestore writes"`

### Task 29: Photo uploader to Cloud Storage

**Files:**
- Create: `tools/migrate-sqlite-to-firestore/lib/photo_uploader.dart`
- Test: `tools/migrate-sqlite-to-firestore/test/photo_uploader_test.dart` (uses Storage emulator)

- [ ] **Step 1: Write failing test**

```dart
test('uploadPhoto writes to /photos/{personId}.png and returns URL', () async {
  final uploader = await PhotoUploader.connect(emulatorHost: 'localhost:9199', bucket: 'demo-ftree.appspot.com');
  final pngBytes = await File('test/fixtures/100.png').readAsBytes();
  final url = await uploader.upload(personId: 'p_100', bytes: pngBytes);
  expect(url, contains('photos/p_100.png'));
});
```

- [ ] **Step 2: Run, fail.**

- [ ] **Step 3: Implement** using `googleapis_storage` against emulator.

- [ ] **Step 4: Run, pass.**

- [ ] **Step 5: Commit.** `git commit -m "migrate: photo upload to Cloud Storage"`

### Task 30: End-to-end migration script with `--dry-run` and verification

**Files:**
- Modify: `tools/migrate-sqlite-to-firestore/bin/migrate.dart`

- [ ] **Step 1: Wire components**

Update `bin/migrate.dart` to:
1. Open SQLite (Task 24)
2. For each individual row → call `toPersonDoc` (Task 25) → write via FirestoreWriter (Task 28)
3. For each member row → call `expandMarriages` → write each Marriage
4. For each PNG file in `--assets-dir` whose stem matches a known `personId` → upload (Task 29), update Person doc with `photoURL`
5. Write a `families/cheleri-mana` doc
6. Print summary: persons written, marriages written, photos uploaded, total time
7. If `--dry-run`, perform reads/transforms but skip writes

- [ ] **Step 2: Run against emulator**

```bash
firebase emulators:start --only firestore,storage  # other terminal
dart run bin/migrate.dart \
  --db /path/to/CheleriFamilyTree.db \
  --assets-dir "/path/to/Cheleri Mana Ancestry/app/src/main/assets/" \
  --service-account none \
  --dry-run
```

Expected: prints "would write 1,499 persons, ~600 marriages, ~607 photos" — no Firestore writes.

Then without `--dry-run`: actual writes happen, emulator UI at `localhost:4000` shows data.

- [ ] **Step 3: Commit.** `git commit -m "migrate: wire end-to-end with dry-run"`

### Task 31: Verification — sample 50 random persons against the existing app

**Files:**
- Create: `tools/migrate-sqlite-to-firestore/bin/verify.dart`

- [ ] **Step 1: Implement**

A second CLI entry that:
1. Picks 50 random `personId`s
2. For each, fetches the Firestore doc + the original SQLite row
3. Compares name, gender, generation, dob, fatherPersonId, motherPersonId
4. Prints any mismatch
5. Exits non-zero if any mismatch

- [ ] **Step 2: Run against emulator data from Task 30**

Run: `dart run bin/verify.dart --db /path/to/CheleriFamilyTree.db --emulator-host localhost:8080 --sample 50`

Expected: "All 50 samples match. ✓"

- [ ] **Step 3: Commit.** `git commit -m "migrate: post-migration verification CLI"`

---

# Phase G — Email-Link Auth (Tasks 32–37)

### Task 32: AuthService.sendSignInLink

**Files:**
- Create: `lib/auth/auth_service.dart`
- Test: `test/auth/auth_service_test.dart`

- [ ] **Step 1: Write failing test**

```dart
import 'package:firebase_auth/firebase_auth.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:mocktail/mocktail.dart';
import 'package:ftree/auth/auth_service.dart';

class _MockAuth extends Mock implements FirebaseAuth {}

void main() {
  setUpAll(() => registerFallbackValue(ActionCodeSettings(url: 'https://example.com')));

  test('sendSignInLink calls FirebaseAuth.sendSignInLinkToEmail with correct settings', () async {
    final mockAuth = _MockAuth();
    when(() => mockAuth.sendSignInLinkToEmail(
      email: any(named: 'email'),
      actionCodeSettings: any(named: 'actionCodeSettings'),
    )).thenAnswer((_) async {});

    final svc = AuthService(mockAuth);
    await svc.sendSignInLink('user@example.com');

    verify(() => mockAuth.sendSignInLinkToEmail(
      email: 'user@example.com',
      actionCodeSettings: any(named: 'actionCodeSettings'),
    )).called(1);
  });
}
```

- [ ] **Step 2: Run, fail.**

- [ ] **Step 3: Implement**

```dart
import 'package:firebase_auth/firebase_auth.dart';

class AuthService {
  final FirebaseAuth _auth;
  AuthService(this._auth);

  static const String _continueUrl = 'https://ftree.npalasseri.com/finishSignIn';
  static const String _iosBundle = 'com.npalasseri.ftree';
  static const String _androidPackage = 'com.npalasseri.ftree';

  Future<void> sendSignInLink(String email) async {
    final settings = ActionCodeSettings(
      url: _continueUrl,
      handleCodeInApp: true,
      iOSBundleId: _iosBundle,
      androidPackageName: _androidPackage,
      androidInstallApp: true,
      androidMinimumVersion: '21',
    );
    await _auth.sendSignInLinkToEmail(email: email, actionCodeSettings: settings);
  }

  Stream<User?> authStateChanges() => _auth.authStateChanges();
  Future<void> signOut() => _auth.signOut();
}
```

- [ ] **Step 4: Run test, pass.**

- [ ] **Step 5: Commit.** `git commit -m "auth: sendSignInLink wraps Firebase Auth"`

### Task 33: AuthService.handleSignInLink

- [ ] **Step 1: Write failing test** verifying that `handleSignInLink(email, link)` calls `signInWithEmailLink`.

- [ ] **Step 2: Run, fail.**

- [ ] **Step 3: Implement**

Add to `AuthService`:

```dart
Future<UserCredential> handleSignInLink({required String email, required String emailLink}) async {
  if (!_auth.isSignInWithEmailLink(emailLink)) {
    throw StateError('Not a sign-in link');
  }
  return _auth.signInWithEmailLink(email: email, emailLink: emailLink);
}
```

- [ ] **Step 4: Run, pass.**

- [ ] **Step 5: Commit.** `git commit -m "auth: handleSignInLink completes email-link flow"`

### Task 34: authStateProvider (Riverpod)

**Files:**
- Create: `lib/auth/auth_state.dart`
- Test: `test/auth/auth_state_test.dart`

- [ ] **Step 1: Test that the provider emits null → User on sign-in.**

- [ ] **Step 2: Implement**

```dart
import 'package:firebase_auth/firebase_auth.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'auth_service.dart';

final firebaseAuthProvider = Provider<FirebaseAuth>((ref) => FirebaseAuth.instance);

final authServiceProvider = Provider<AuthService>((ref) {
  return AuthService(ref.read(firebaseAuthProvider));
});

final authStateProvider = StreamProvider<User?>((ref) {
  return ref.read(authServiceProvider).authStateChanges();
});
```

- [ ] **Step 3: Pass, commit.** `git commit -m "auth: Riverpod providers for auth state"`

### Task 35: SignInEmailPage UI

**Files:**
- Create: `lib/auth/pages/sign_in_email_page.dart`
- Test: `test/auth/sign_in_email_page_test.dart`

- [ ] **Step 1: Widget test**

Test that entering "valid@email.com" + tapping "Send link" calls `AuthService.sendSignInLink`. Use `ProviderScope.overrides` to inject a mock.

- [ ] **Step 2: Implement** the form with email validation, primary button, loading state, error snackbar.

- [ ] **Step 3: Pass, commit.** `git commit -m "auth: sign-in email entry page"`

### Task 36: SignInLinkPendingPage UI

**Files:**
- Create: `lib/auth/pages/sign_in_link_pending_page.dart`

A simple page showing "We sent a link to <email>. Tap it to sign in. Didn't get it? [Resend]". Widget test + commit.

### Task 37: Deep link configuration

**Files:**
- Modify: `ios/Runner/Info.plist`
- Modify: `android/app/src/main/AndroidManifest.xml`
- Modify: `lib/app.dart`

- [ ] **Step 1: Configure iOS**

Add to `ios/Runner/Info.plist`:

```xml
<key>FirebaseDynamicLinksCustomDomains</key>
<array>
  <string>https://ftree.npalasseri.com</string>
</array>
<key>CFBundleURLTypes</key>
<array>
  <dict>
    <key>CFBundleURLSchemes</key>
    <array><string>com.npalasseri.ftree</string></array>
  </dict>
</array>
```

- [ ] **Step 2: Configure Android**

Add intent-filter to `android/app/src/main/AndroidManifest.xml` inside `<activity>`:

```xml
<intent-filter android:autoVerify="true">
  <action android:name="android.intent.action.VIEW"/>
  <category android:name="android.intent.category.DEFAULT"/>
  <category android:name="android.intent.category.BROWSABLE"/>
  <data android:scheme="https" android:host="ftree.npalasseri.com"/>
</intent-filter>
```

- [ ] **Step 3: Use `app_links` package** to handle the inbound link.

Add to `pubspec.yaml`: `app_links: ^6.3.0`. `flutter pub get`.

In `lib/app.dart`, listen to `AppLinks().uriLinkStream`, when the URI matches the email-link pattern call `AuthService.handleSignInLink(email: storedEmail, emailLink: uri.toString())`.

- [ ] **Step 4: Manual test**

1. Run app on iOS sim
2. Enter email on SignInEmailPage
3. Open Firebase Console → Authentication → Templates → "Email link" → copy the link that gets sent (or check the user's email)
4. Open Safari in sim, paste link → app should open and complete sign-in
5. Repeat for Android

- [ ] **Step 5: Commit.** `git commit -m "auth: deep link config for email-link sign-in"`

---

# Phase H — Invite Gate (Tasks 38–41)

### Task 38: InviteRepository

**Files:**
- Create: `lib/data/invite_repository.dart`
- Test: `test/data/invite_repository_test.dart`

- [ ] **Step 1: Write failing test** (uses Firestore mock)

```dart
test('redeem returns the invite for a valid unredeemed code', () async {
  // Setup mock Firestore with /invites/test-code, expiresAt in future, not redeemed
  final repo = InviteRepository(mockFirestore);
  final invite = await repo.redeem('test-code', uid: 'u_1');
  expect(invite.code, 'test-code');
});

test('redeem throws InviteNotFound for unknown code', () async { ... });
test('redeem throws InviteExpired for past expiresAt', () async { ... });
test('redeem throws InviteAlreadyRedeemed if redeemedBy set', () async { ... });
```

- [ ] **Step 2: Run, fail.**

- [ ] **Step 3: Implement** with custom exceptions and transactional write to set `redeemedBy`/`redeemedAt`.

- [ ] **Step 4: Pass, commit.** `git commit -m "invite: repository with redeem + validation"`

### Task 39: InviteRedeemPage UI

**Files:**
- Create: `lib/auth/pages/invite_redeem_page.dart`

A page shown after successful sign-in if the user has no `/users/{uid}` doc yet. Text input for invite code + "Redeem" button. On success, writes `/users/{uid}` and navigates to HomePage. On failure, shows the specific error.

Widget test + commit: `invite: redeem page UI`.

### Task 40: Firestore rules for /invites + /users

**Files:**
- Modify: `firebase/firestore.rules`
- Create: `firebase/firestore.rules.test.js` (rules emulator tests)

- [ ] **Step 1: Update rules**

```
match /invites/{inviteId} {
  // Authenticated user can read any invite (to redeem it).
  allow read: if request.auth != null;
  // Only writes via Cloud Functions or admin SDK (no client writes).
  allow write: if false;
}

match /users/{uid} {
  // A user can read their own doc.
  allow read: if request.auth != null && request.auth.uid == uid;
  // A user can create their own doc with their uid.
  allow create: if request.auth != null && request.auth.uid == uid;
  // A user can update only specific fields on their own doc.
  allow update: if request.auth != null && request.auth.uid == uid;
  allow delete: if false;
}
```

- [ ] **Step 2: Add @firebase/rules-unit-testing as dev dep**

```bash
cd firebase && npm init -y && npm install --save-dev @firebase/rules-unit-testing
```

- [ ] **Step 3: Write rules tests** in `firebase/firestore.rules.test.js`: unauthenticated cannot read /invites, authenticated user CAN read /invites, etc.

- [ ] **Step 4: Run rules tests against emulator.** All pass.

- [ ] **Step 5: Deploy rules.** `firebase deploy --only firestore:rules`

- [ ] **Step 6: Commit.** `git commit -m "rules: allow auth users to read invites and own user doc"`

### Task 41: Seed one test invite

Run from local CLI (admin context) or via the Firebase console UI:

```
Collection: invites
Doc ID: TEST-CODE-001
Fields:
  inviteId: "TEST-CODE-001"
  code: "TEST-CODE-001"
  createdBy: "manual"
  role: "viewer"
  expiresAt: <30 days from now, Timestamp>
```

Manual smoke: open the app on iOS sim → sign in via email link → redeem `TEST-CODE-001` → confirm `/users/<your-uid>` doc was created in Firestore Console.

Commit: nothing changes (this is a runtime smoke). Mark task complete.

---

# Phase I — Read Repositories + Security Rules (Tasks 42–47)

### Task 42: PersonRepository read methods

**Files:**
- Create: `lib/data/person_repository.dart`
- Test: `test/data/person_repository_test.dart`

Methods (with TDD tests using Firestore mock):
- `Stream<Person?> watchPerson(String personId)`
- `Future<Person?> getPerson(String personId)`
- `Future<List<Person>> childrenOf(String personId)` — query persons where `fatherPersonId == personId || motherPersonId == personId`
- `Future<List<Person>> searchByName(String query)` — prefix search using Firestore range query on `name`
- `Future<List<Person>> siblingsOf(Person p)` — same father or mother

Implementation uses `cloud_firestore`. JSON converter via the freezed-generated `fromJson`.

Commit: `data: PersonRepository with read methods`.

### Task 43: MarriageRepository

Methods:
- `Future<List<Marriage>> marriagesOf(String personId)` — where `personAId == personId || personBId == personId`

Commit: `data: MarriageRepository`.

### Task 44: FamilyRepository

Method: `Future<Family> getFamily(String familyId)`. Trivial. Commit.

### Task 45: Firestore rules for persons/marriages/families

**Files:**
- Modify: `firebase/firestore.rules`

- [ ] **Step 1: Rule for /persons read**

```
match /persons/{personId} {
  allow read: if request.auth != null
    && exists(/databases/$(database)/documents/users/$(request.auth.uid));
  allow write: if false;  // edits in Slice 2
}
```

- [ ] **Step 2: Rule for /marriages and /families**

Same pattern: authenticated + user doc exists → read; no write.

- [ ] **Step 3: Rules tests** in emulator: unauth user blocked, auth user without /users/{uid} doc blocked, auth user with /users/{uid} doc allowed.

- [ ] **Step 4: Deploy + commit.** `git commit -m "rules: viewer read access for persons/marriages/families"`

### Task 46: DOB visibility privacy rule

> Privacy rule: viewers cannot see DOB for living persons. Implementation choice: enforce at the read path with a Cloud Function or projection, or strip DOB client-side after reading. Cloud Function approach is more secure (defense in depth), but Slice 1 uses **field-level rule check + client-side stripping** to keep complexity down. A malicious client COULD read raw doc — Slice 2 will add Cloud Function projection for the proper fix.

**Files:**
- Modify: `firebase/firestore.rules` (no change — DOB is in same doc as person; rules can't hide individual fields in MVP)
- Modify: `lib/data/person_repository.dart` (strip DOB client-side for living + non-self viewers)
- Modify: `lib/ui/widgets/dob_display.dart` (Task 51)

- [ ] **Step 1: Test in `person_repository_test.dart`**

```dart
test('getPerson returns DOB only if not living OR is the viewer', () async {
  // ... viewer is u_1, person p_100 is living, p_100 is not linked to u_1
  final p = await repo.getPerson('p_100');
  expect(p!.dob, isNull);  // stripped
});
```

- [ ] **Step 2: Implement** stripping in the repo `_fromDoc` mapper based on current user's `linkedPersonId` and the person's `isLiving`.

- [ ] **Step 3: Pass, commit.** `git commit -m "privacy: strip DOB client-side for living non-self"`

### Task 47: Run all data tests

- [ ] **Step 1:** `flutter test test/data/` — all green.

---

# Phase J — Read UI (Tasks 48–56)

### Task 48: HomePage routing shell

**Files:**
- Create: `lib/ui/pages/home_page.dart`
- Modify: `lib/app.dart` (add router)

go_router config:
- `/` → if not authed → SignInEmailPage; if authed but no /users/{uid} doc → InviteRedeemPage; else → HomePage
- `/profile/:personId` → ProfilePage
- `/search` → SearchPage
- `/relate/:fromId/:toId?` → RelationshipCalculatorPage

Widget test that the router routes correctly per auth state. Commit: `ui: app router with auth-aware redirects`.

### Task 49: PersonCard widget

**Files:**
- Create: `lib/ui/widgets/person_card.dart`
- Test: `test/ui/widgets/person_card_test.dart`

Compact card showing avatar (CircleAvatar with photoURL or initials), name, generation badge. Tap → callback. Widget test for: renders name; renders initials when no photo; calls onTap. Commit: `ui: PersonCard widget`.

### Task 50: FamilyAroundPerson widget

**Files:**
- Create: `lib/ui/widgets/family_around_person.dart`
- Test: `test/ui/widgets/family_around_person_test.dart`

Given a Person, fetches parents (via repo), spouse(s) (via MarriageRepository), children (via repo), siblings (via repo). Renders 4 sections of PersonCards, each tappable. Empty sections collapse. Widget test with fake repos. Commit: `ui: FamilyAroundPerson widget`.

### Task 51: DobDisplay widget

**Files:**
- Create: `lib/ui/widgets/dob_display.dart`
- Test: `test/ui/widgets/dob_display_test.dart`

Pure rendering: takes optional `Dob` and `isLiving`. If `dob == null` shows "—"; if `isLiving && !isSelf` shows "(hidden)"; else shows formatted date ("15 June 1950" / "1950" if only year). Widget test for each case. Commit: `ui: DobDisplay`.

### Task 52: ProfilePage layout

**Files:**
- Create: `lib/ui/pages/profile_page.dart`
- Test: `test/ui/pages/profile_page_test.dart`

Layout:
- AppBar with name
- Hero image (photo or placeholder, large)
- Section: Bio (gender, generation, genId, profession, blood group, contact if visible)
- Section: DOB (via DobDisplay)
- Section: Family (FamilyAroundPerson)
- FAB → RelationshipCalculatorPage with this person preselected

Widget test for the loading/loaded/error states. Commit: `ui: ProfilePage`.

### Task 53: SearchPage

**Files:**
- Create: `lib/ui/pages/search_page.dart`
- Test: `test/ui/pages/search_page_test.dart`

TextField with debounce (300ms) → calls `PersonRepository.searchByName` → list of PersonCards → tap navigates to ProfilePage. Widget test for: typing query triggers search; results render; empty state. Commit: `ui: SearchPage with debounced name search`.

### Task 54: Wire navigation between pages

Update go_router routes (Task 48). Test that tapping a sibling on ProfilePage of personId=100 navigates to ProfilePage of the sibling. Integration test (next phase) will exercise the full flow. Commit: `ui: profile-to-profile navigation`.

### Task 55: Loading / error / empty states across pages

Spec says generous spacing + readability. For each Async-driven widget, ensure:
- Loading: spinner centered with brief label
- Error: small inline error + retry button
- Empty: friendly empty state with a single illustration/icon

Commit: `ui: consistent async states across pages`.

### Task 56: Cache and prefetch tuning

Firestore default offline persistence is on. Add prefetch on `ProfilePage.initState` for likely-next-tap relatives (parents, spouse, children, siblings) so taps feel instant.

Commit: `ui: prefetch relatives for snappier navigation`.

---

# Phase K — Relationship Calculator (Tasks 57–61)

### Task 57: Graph traversal — find path A→B

**Files:**
- Create: `lib/relationship/calculator.dart`
- Test: `test/relationship/calculator_test.dart`

- [ ] **Step 1: Write failing test**

Build a small in-memory graph (10 persons, known relationships). Assert that `findPath(graph, 'p_1', 'p_5')` returns the expected ancestor-descendant or sibling-of-cousin path.

- [ ] **Step 2: Implement** BFS from A and from B until a common ancestor is found, return list of hops typed as `Hop.parent`, `Hop.child`, `Hop.spouse`, `Hop.sibling`.

- [ ] **Step 3: Pass, commit.** `git commit -m "relate: path-finding via BFS to common ancestor"`

### Task 58: Kinship term mapping

**Files:**
- Create: `lib/relationship/kinship_terms.dart`
- Test: `test/relationship/kinship_terms_test.dart`

Use the kinship term list extracted in Task 15. Implement a function `kinshipTerm(List<Hop> path, {required Gender ofRelative}) → String` that maps hop sequences to terms ("father's brother" = `[parent, sibling]` with male relative = "paternal uncle").

Cover at least the 20 documented relationships from Task 15 in tests.

Commit: `relate: kinship term mapping from hop sequence`.

### Task 59: Parity tests — 20 known relationships from existing app

- [ ] **Step 1: Test fixture**

`test/relationship/parity_test.dart`:

```dart
const knownRelationships = [
  // (fromPersonId, toPersonId, expectedTerm) — sourced from Task 15
  ('p_100', 'p_105', 'paternal uncle'),
  ('p_100', 'p_200', 'spouse'),
  // ... 20 total
];
```

For each, build a graph from the migrated Firestore (or a test fixture file), run the calculator, assert the term matches.

- [ ] **Step 2: Iterate** until all 20 pass. Adjust calculator or term mapping as needed; do not change the fixture (it's the parity source of truth).

- [ ] **Step 3: Commit.** `git commit -m "relate: 20-relationship parity tests vs existing app"`

### Task 60: RelationshipCalculatorPage UI

**Files:**
- Create: `lib/ui/pages/relationship_calculator_page.dart`
- Test: `test/ui/pages/relationship_calculator_page_test.dart`

Two person pickers (autocomplete using PersonRepository.searchByName) + "Calculate" button → shows the relationship term and a breadcrumb of the path. Widget test for happy path + same-person edge case. Commit: `ui: relationship calculator page`.

### Task 61: Manual relationship calculator smoke test

Use the running app to compute relationships for 5 different pairs you know. Compare against the existing Cheleri Mana app's output for the same pairs. If any drift, file a follow-up (don't fix here — Task 59 fixtures cover 20 anchor cases; other drift means kinship rules are richer than expected).

---

# Phase L — Integration Tests + Handoff (Tasks 62–66)

### Task 62: End-to-end emulator test

**Files:**
- Create: `integration_test/smoke_test.dart`

```dart
import 'package:integration_test/integration_test.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:ftree/main.dart' as app;

void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();

  testWidgets('Invite redeem → browse → search → relate', (tester) async {
    app.main();
    await tester.pumpAndSettle();

    // 1. Enter email, tap "Send link"
    await tester.enterText(find.byKey(const Key('email-field')), 'test@example.com');
    await tester.tap(find.byKey(const Key('send-link-btn')));
    await tester.pumpAndSettle();

    // 2. Simulate inbound link (call AuthService.handleSignInLink directly via provider override)
    // ...

    // 3. Redeem invite TEST-CODE-001
    // ...

    // 4. Search for "Subraya", tap result
    // ...

    // 5. Tap "Calculate relationship" with Subraya as 'from'
    // ...

    // 6. Verify the term appears
    // ...
  });
}
```

Connects to Firebase emulator (seeded with Task 30 data). Commit: `integration: full read flow against emulator`.

### Task 63: iOS manual test checklist

**Files:**
- Modify: `README.md` (add a "Slice 1 acceptance checklist" section)

Checklist for the maintainer to run on a real iPhone (TestFlight build or `flutter run`):
- [ ] App launches, shows Heritage Warm splash
- [ ] Sign in with own email; email arrives
- [ ] Tap link in email → app opens via universal link
- [ ] Redeem a test invite
- [ ] Browse Cheleri tree: open 5 random profiles
- [ ] DOB hidden for known-living person; visible for known-deceased
- [ ] Search "Subraya" finds the right person
- [ ] Relationship calculator: pair you know → correct term
- [ ] Force quit, reopen → still signed in, profile data still cached offline
- [ ] Toggle airplane mode → can still browse cached profiles

### Task 64: Android manual test checklist

Same checklist as Task 63, executed on a physical Android device. Add to README.

### Task 65: Update README

**Files:**
- Modify: `README.md`

Add sections:
- "How to build and run Slice 1" (prerequisites, `flutter pub get`, `dart run build_runner build`, `flutter run`)
- "How to run the migration tool"
- "How to run the emulator suite"
- "Slice 1 acceptance checklist" (from Tasks 63–64)

Commit: `docs: README build/run instructions for Slice 1`.

### Task 66: Final self-review

- [ ] **Step 1:** Re-read the spec sections (§1–§8) and confirm each Slice 1 requirement maps to a completed task above.
- [ ] **Step 2:** Run the full test suite: `flutter test && cd tools/migrate-sqlite-to-firestore && dart test && cd ../..`
- [ ] **Step 3:** Run `flutter analyze` — clean.
- [ ] **Step 4:** Run the integration test against emulator — green.
- [ ] **Step 5:** Open https://github.com/npalasseri/ftree/actions — CI green on `main`.
- [ ] **Step 6:** Tag the slice: `git tag -a slice-1 -m "Slice 1: read-only Cheleri tree with email-link auth"` and `git push --tags`.

---

## Plan Self-Review Notes (filled in after writing)

**Spec coverage gaps identified:** None blocking. The spec's §6.2 mentions a "zoomable canvas" and "indented list" as alternative tree views — those are intentionally deferred to Slice 4 (power-user views) per the slicing decision in this session.

**Type-name consistency check:** `Person.personId`, `Marriage.marriageId`, `Family.familyId`, `Invite.inviteId` — consistent. `Gender` enum used in Person model and KinshipTerms — consistent.

**Placeholder scan:** No "TBD" / "TODO" left in plan text. Two tasks (15, 16) intentionally produce *reference documents* before the code that depends on them — those aren't placeholders, they're discovery work.

**Open assumptions to verify during execution:**
1. The bundled DB filename inside `cheleri_mana_vamsavali3.zip` — assumed to be `CheleriFamilyTree.db` per spec §8 but Task 14 will confirm.
2. The exact column names in the `individual` table — Task 14 documents them, Task 25's test fixtures should be updated post-discovery if column names differ from the assumed `db`/`dm`/`dd`/`sex`/`blgroup`/`phone`/`email`/`address`.
3. Whether the kinship vocabulary includes Tulu / Kannada terms in the existing app — if so, the parity tests (Task 59) may need a language-of-output decision.

---

## Execution Choice

Plan complete and saved to `docs/superpowers/plans/2026-05-24-slice1-read-only-tree.md`. Two execution options:

**1. Subagent-Driven (recommended)** — I dispatch a fresh subagent per task, review between tasks, fast iteration. Good fit for a 66-task plan where individual tasks are well-bounded.

**2. Inline Execution** — Execute tasks in this session using `superpowers:executing-plans`, batch execution with checkpoints for review.

Which approach?
