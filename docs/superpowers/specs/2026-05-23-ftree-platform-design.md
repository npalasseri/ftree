# Family Tree Platform — Cross-Platform Multi-Family Community App

**Status:** Design (validated through brainstorming, ready for implementation planning)
**Date:** 2026-05-23
**Working name:** `ftree` (product name TBD)
**Existing app being replaced:** Cheleri Mana Ancestry (native Android, Java, bundled SQLite)

---

## 1. Context

The user has an existing native Android app, **Cheleri Mana Ancestry**, that serves as a single-family genealogy browser. It ships a curated SQLite database (1,499 individuals, 1,008 graph members across ~14 generations) as a bundled asset, with ~13 activities for browsing branches, individuals, profiles, relationships, and submitting correction/add-member requests.

The owner wants to **rebuild this as a cross-platform, multi-family community platform** — not just port the existing app. The new product extends the single-family model into one shared community genealogy graph where multiple families coexist, interconnected by cross-family marriages, with role-based access for admins and invited viewers. The Cheleri Mana tree becomes the seed family for the new platform.

The motivation is twofold:
1. **Cross-platform reach** — iOS users currently cannot access the tree at all
2. **Scale beyond one family** — the same software should serve any family within the community, not require a per-family fork/republish

---

## 2. Goals and Non-Goals

### Goals (MVP)

- Cross-platform mobile app (Android + iOS) with shared codebase
- One global community graph; multiple families coexist as logical sub-branches
- Three-tier role model: super admin → family admins → invited viewers
- Invite-gated access; once invited, viewers can traverse the entire community graph
- Family admins can add/edit members within their family in-app (no spreadsheet-and-republish loop)
- Cross-family edits use a time-bounded consensus model with visible `*` indicator for unresolved approvals
- Migrate the existing 1,499 Cheleri individuals as Day-1 seed data
- Modern UI ("Heritage Warm" palette: cream + walnut + antique gold)
- Phone OTP authentication (free via Firebase Auth) with email fallback
- Real-time updates across devices (edits propagate live)
- Photo uploads per member
- Search by name across the whole graph
- Relationship calculator (preserve the existing app's "how am I related to X" feature)
- Offline read; writes queue and sync when online

### Non-Goals (deferred to later phases)

- **Phase 2:** Viewer-side correction/add-member requests (the existing app's `CorrectionRequest` / `AddMemberRequest` flow), GEDCOM import/export
- **Phase 3:** Multi-language UI (Tulu / Kannada / Malayalam), media gallery beyond profile photos, stories/biographies
- **Phase 4:** Web app (Flutter Web)
- **Out of scope entirely:** DNA matching, paid tiers, public family-tree directory

---

## 3. Architecture Overview

### Framework choice: Flutter / Dart

Selected over React Native because:

1. **Dart is syntactically closer to Java** than JavaScript/TypeScript (statically typed, class-based OOP, `async/await`, similar IDE — Android Studio). The maintainer's Java background transfers more directly.
2. **Single codebase for Android + iOS** today, **mature Flutter Web** path for Phase 4.
3. **Performance characteristics** (compiled to native, no JS bridge) suit interactive graph navigation.

### Backend: Firebase (all-managed, zero infrastructure)

| Component | Service | Free-tier coverage at MVP scale |
|---|---|---|
| Database | **Firestore** (NoSQL document store) | 50k reads/day, 20k writes/day, 1 GiB storage |
| Auth | **Firebase Auth** (phone OTP + email) | Phone OTP free, email free |
| Push | **Firebase Cloud Messaging (FCM)** | Free, unlimited |
| Photos / files | **Firebase Cloud Storage** | 5 GB storage, 1 GB/day download |
| Analytics | **Firebase Analytics** | Free, unlimited |

The free tier comfortably supports several hundred to low-thousands of active users for this community-app scale. No monthly cost at launch.

### High-level system shape

```
┌────────────────────┐         ┌────────────────────┐
│  Flutter app       │  reads  │   Firestore        │
│  (Android + iOS)   │◄────────│   (NoSQL graph)    │
│                    │  writes │                    │
│  - profile view    │────────►│  Collections:      │
│  - tree canvas     │         │   /users           │
│  - admin edits     │         │   /persons         │
│  - consensus UI    │         │   /families        │
│  - notifications   │         │   /marriages       │
└─────────┬──────────┘         │   /invites         │
          │                    │   /consensus       │
          │ auth/OTP            │   /audit           │
          ▼                    │   /notifications   │
┌────────────────────┐         └─────────┬──────────┘
│  Firebase Auth     │                   │
│  (phone + email)   │                   │  storage refs
└────────────────────┘                   ▼
                                ┌────────────────────┐
                                │  Cloud Storage     │
                                │  /photos/{personId}│
                                └────────────────────┘

       ┌──────────────────┐
       │  FCM Push        │  ← invites, consensus pings, approvals
       └──────────────────┘
```

---

## 4. Data Model (Firestore collections)

Designed to denormalize for read performance while preserving correctness for the graph.

### `/users/{uid}`
App account record. Distinct from `persons` — a person can exist in the genealogy without ever logging in.

```
{ uid, phone, email, displayName, photoURL, linkedPersonId?, createdAt, lastLoginAt }
```

### `/families/{familyId}`
A logical sub-branch (the seed entry is "Cheleri Mana"; additional families added later).

```
{ familyId, name, slug, foundedGeneration?, description?, createdAt, createdBy }
```

### `/families/{familyId}/admins/{uid}`
Many-to-many: which users admin which families. (Sub-collection for security-rule simplicity.)

```
{ uid, addedAt, addedBy, role: "admin" }
```

### `/persons/{personId}`
Genealogy individual. The core data of the app.

```
{
  personId, familyId,            // birth family
  name, gender, generation,
  fatherPersonId?, motherPersonId?,
  isLiving: bool,
  dob?: { year, month, day },    // hidden from viewers when isLiving (see security rules)
  dobVisibility: "default" | "public",
  profession?, bloodGroup?,
  contact?: { phone, email, address },  // visible per privacy rules
  photoURL?,
  createdBy, createdAt, updatedBy, updatedAt,
  consensusFlags?: ["dob", "name", ...]  // fields awaiting cross-family approval
}
```

### `/marriages/{marriageId}`
Junction table replacing the existing `spID/spID2/spID3` columns. Supports cross-family marriages and the consensus model cleanly.

```
{
  marriageId,
  personAId, personBId,
  personAFamilyId, personBFamilyId,
  isCrossFamily: bool,           // derived; indexed for queries
  marriedDate?, dissolvedDate?,
  status: "married" | "divorced" | "deceased-spouse",
  consensusStatus: "approved" | "pending" | "rejected",
  awaitingApprovals?: [uid],     // when isCrossFamily; the other family's admin(s)
  approvalDeadline?: timestamp,  // default 7 days from edit
  createdBy, createdAt
}
```

### `/invites/{inviteId}`
Invite codes/links.

```
{ inviteId, code, createdBy, familyId?, role: "viewer" | "admin",
  email?, phone?, expiresAt, redeemedBy?, redeemedAt? }
```

### `/consensus/{requestId}`
Cross-family edit tracking — the data behind the `*` indicator.

```
{
  requestId,
  targetType: "person" | "marriage",
  targetId,
  field, oldValue, newValue,
  proposedBy, proposedAt,
  affectedAdmins: [uid],
  approvals: { [uid]: { status: "approve" | "reject", at } },
  status: "pending" | "approved" | "rejected" | "expired",
  deadline,
  resolvedAt?
}
```

### `/audit/{entryId}`
Immutable edit log. Every write to `/persons` or `/marriages` triggers a Cloud Function that appends an audit entry. Used for trust, dispute resolution, and the `*` detail view.

```
{ entryId, targetType, targetId, field, oldValue, newValue, actor, at }
```

### `/notifications/{userId}/items/{notificationId}`
Per-user inbox.

```
{ type, message, link, read: bool, createdAt }
```

### Indexes / denormalization

- **`persons`** indexed by `familyId`, `generation`, `fatherPersonId`, `motherPersonId`
- **`marriages`** indexed by `personAId`, `personBId`, `isCrossFamily`, `consensusStatus`
- **Generation IDs** (the existing app's `Uid` letter-string scheme) are computed and cached on each person doc as `genId: string` for fast ordering
- Ancestor lookups beyond 2 levels happen client-side: fetch in batches, cache aggressively in Hive/sqflite local cache

---

## 5. Roles and Permissions

| Role | Scope | Can do |
|---|---|---|
| **Super admin** | Platform-wide | Create families, assign family admins, override `*` flags, arbitrate disputes, configure platform settings (consensus deadline default, etc.) |
| **Family admin** | One or more families | Add/edit/delete persons within their family; create/edit marriages including cross-family edges (triggers consensus); send invites; promote co-admins (with super admin approval) |
| **Viewer** | Whole community graph | Read all data subject to privacy rules; view `*` consensus details; cannot edit; cannot see DOB of living members |
| **Self** | Their own person record | Override DOB visibility on their own record; upload their own photo; edit their own contact PII |

Permission enforcement is **Firestore Security Rules**, not app-side logic. Rules are written and version-controlled in `firestore.rules` and tested in CI with the Firebase Emulator.

---

## 6. Key User Flows

### 6.1 Invite → onboard
1. Family admin or super admin sends invite (phone or email) → creates `/invites/{id}` doc
2. Recipient opens link → app deep-links into onboarding
3. Phone OTP / email magic link verifies identity → creates `/users/{uid}`
4. Optional: link to existing `personId` if the new user matches a person in the tree
5. Land on home screen showing their family's tree first

### 6.2 Browse the graph (primary)
- **Profile-centric navigation:** open a person, see parents / spouse / children / siblings as tappable cards, tap to walk the graph
- **Zoomable canvas** (secondary): button in app bar opens a Figma-style pannable graph view of the full family
- **Indented list** (tablet/wide): per-family browse mode shown on wide layouts

### 6.3 Family admin edits a member
1. Admin taps a person within their family → "Edit" button visible
2. Edits dispatched as a Firestore write; UI updates optimistically (real-time via Firestore listeners)
3. If the edit touches a cross-family edge (e.g., a marriage spouse from another family) → trigger consensus flow (§6.4)
4. All changes appended to `/audit`

### 6.4 Cross-family consensus

**What counts as a cross-family edit (triggers consensus):**
- Creating, editing, or dissolving a marriage where the two persons belong to different families (`marriages.isCrossFamily = true`)
- Editing any field on a marriage doc where `isCrossFamily = true`

**What does NOT trigger consensus:**
- Editing a person's bio details (name, DOB, contact, photo, profession). The person's **birth family** always owns their bio data — only that family's admin can edit, no cross-family approval needed.
- Editing parent-child edges within the same family
- Adding/removing children of an existing within-family marriage

**Flow:**
1. Family admin edits a cross-family marriage (creates, modifies, or dissolves it)
2. Cloud Function creates a `/consensus/{id}` doc with `affectedAdmins` = the OTHER family's admin(s) and a 7-day `deadline` (super-admin-configurable platform default)
3. Edit applies immediately; the affected marriage doc has the field name added to `consensusFlags`, which renders as `*` in the UI
4. FCM push notifies the other family's admins
5. Each affected admin approves or rejects in-app
6. **All approve before deadline** → `consensusFlags` cleared on the marriage doc, `*` removed
7. **Any reject** *(assumption: not explicitly confirmed in brainstorming — please verify during plan review)* → edit reverts to the previous value (read from `/audit`), super admin gets a notification with both versions for arbitration
8. **Deadline passes with no action** → `consensusFlags` remain indefinitely; `*` stays visible. Either admin can still act on it later to clear or reject.
9. Tapping the `*` opens a sheet showing: edit details, who proposed, when, who's still pending, deadline (if any), and the original vs new value

### 6.5 Super admin promotes a family admin
1. Super admin selects a user from `/users` → "Promote to family admin"
2. Picks the family
3. Writes `/families/{familyId}/admins/{uid}` doc
4. FCM push notifies the new admin

---

## 7. UI Direction

### Visual style: "Heritage Warm"
- Background: `#faf6ee` (cream)
- Surface: `#ede4d3` (warm sand)
- Primary: `#7a5230` (deep walnut)
- Accent: `#b8860b` (antique gold)
- Text: `#2c1810`
- Dark mode opt-in (warm graphite + gold) post-MVP

Photos and family imagery feel warm on cream. Reads as modern (clean type, generous spacing, restrained accents) without losing cultural rootedness.

### Tree visualization
- **Primary navigation:** profile-centric (open person → tappable family-around-them card layout)
- **Secondary "whole tree" view:** zoomable canvas (pannable, scalable graph) reachable from app-bar button
- **Tablet/wide:** indented collapsible list as a browse-by-branch alternative

### Typography
- System font (San Francisco on iOS, Roboto on Android) — best legibility and feels native on each platform. Optional Tulu/Kannada glyph fallback in Phase 3.

### Density
- Generous spacing; readability prioritized over information density. Family elders are part of the user base.

---

## 8. Migration from Existing SQLite

The bundled `CheleriFamilyTree.db` (237 KB, 1,499 individuals, 1,008 graph members) is the Day-1 seed.

### Migration script (one-time, run during pre-launch)
1. Read SQLite tables: `individual`, `member`, `UniqueID`
2. Create `families/cheleri-mana` document
3. For each row in `individual`: write to `/persons/{generatedId}` with `familyId: "cheleri-mana"`
4. For each row in `member`: resolve parent / spouse relationships → write to `/persons` (parent FKs) and `/marriages` (spouse junction; convert `spID` + `spID2` + `spID3` to multiple `marriage` docs per person)
5. Consolidate `db / dm / dd` → single `dob` field
6. Set `isLiving` heuristically: the existing schema has no `isLiving` or death-date column, so we infer it. Heuristic: `isLiving = (generation >= maxGeneration - 3)` — i.e., people in the most recent 4 generations are assumed living; older generations are assumed deceased. Edge cases corrected post-launch by family admins via the admin edit flow. The heuristic is documented in the migration script's README so the assumption is auditable.
7. Compute and cache `genId` (Uid letter-string) using the same algorithm as the existing `Unique_id` view
8. Skip the SQL views — relationship lookups become Dart-side graph traversals (with caching) or Firestore queries

### Data quirks handled during migration
- DOB stored as 3 columns → single `{year, month, day}` object
- Up to 3 spouses per row → multiple marriage docs (junction)
- `house` (free text) → normalized: every Cheleri person gets `familyId: "cheleri-mana"`; future families get distinct `familyId`
- Living-vs-deceased: heuristic for migration; family admins flag corrections post-launch
- Contact / email / address: present in seed data, kept as-is

Migration script lives in `tools/migrate-sqlite-to-firestore/` (Dart CLI using the Firebase Admin SDK). Idempotent. Logs every record written.

---

## 9. Phasing

| Phase | Scope | Estimated effort |
|---|---|---|
| **MVP (Phase 1)** | Viewer + family-admin editing, invites, consensus, migrate Cheleri seed data, Heritage Warm UI, Android + iOS | **2–3 months** |
| **Phase 2** | Viewer correction-request workflow (port from existing app), GEDCOM import/export, polish based on early feedback | 4–6 weeks |
| **Phase 3** | Multi-language (Tulu / Kannada / Malayalam), media gallery, stories/biographies, dark mode | 4–6 weeks |
| **Phase 4** | Flutter Web release (admin panel + viewer site) | 4–8 weeks |

---

## 10. Verification Plan

For the MVP, verification means:
- **Unit tests** for the relationship calculator (test against the existing app's output for the same input — sanity check)
- **Integration tests** with the Firebase Emulator covering: invite → onboard → view → admin edit → cross-family consensus → approval/rejection paths
- **Manual end-to-end** on iOS Simulator and a physical Android device for the happy path
- **Migration verification:** after running the SQLite → Firestore import script, randomly sample 50 persons from the existing app and confirm same parents/spouses/children appear in the new app
- **Security rules tests** in CI: viewer cannot read living-person DOB; non-admin cannot edit persons; cross-family edit triggers consensus doc; deadline-expired consensus does not revert
- **Performance budget:** profile view opens in < 200ms (cached); zoomable canvas renders 1,000 nodes at 60fps

---

## 11. Open Items (for implementation phase, not blockers for the plan)

| Item | Decision needed when |
|---|---|
| Product/brand name (working name `ftree`) | Before App Store submission |
| Final font choice if multi-language fallback needed | Phase 3 |
| Firebase free tier was chosen for MVP. If the community grows past it and Firestore reads/writes become expensive, revisit (Pockethost / Supabase / self-hosted Postgres are the migration targets) | Post-MVP, only if monthly Firebase bill exceeds ~$30/mo |
| Whether to use Firestore Bundles for offline-first seed data (faster first-launch) | During implementation |
| Profile photo crop UX (auto-detect face? manual?) | During implementation |
| Notification preferences UI (per-category opt-out) | Phase 2 |
| Account deletion + GDPR/DPDP compliance flow | Before public launch |

---

## 12. Decisions Summary (validated through brainstorming)

| Topic | Decision |
|---|---|
| Framework | Flutter / Dart |
| Platforms | Android + iOS (MVP); Web (Phase 4) |
| Architecture | Single global community graph; families are logical sub-branches |
| Roles | Super admin → Family admins → Viewers |
| Visibility | Invite-gated; whole graph visible to all invitees |
| MVP scope | Viewer + family-admin editing (~2–3 months) |
| Correction-request workflow | Phase 2 |
| Cross-family edits | Time-bounded consensus (7-day default), `*` flag, super-admin arbitration |
| Seed data | Migrate existing Cheleri SQLite (1,499 individuals) |
| Backend | Firebase (Firestore + Auth + FCM + Cloud Storage) |
| Auth | Phone OTP (primary) + email (fallback) |
| Privacy | Open inside invite gate, **except DOB hidden from viewers** |
| Visual style | Heritage Warm (cream + walnut + antique gold) |
| Tree visualization | Profile-centric primary; zoomable canvas secondary; indented list on wide layouts |
| GEDCOM | Phase 2 |
| Offline | Read (cached); writes queue |
