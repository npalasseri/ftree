# ftree

Cross-platform multi-family community genealogy app. Flutter + Firebase. Replaces and extends the existing Cheleri Mana Ancestry Android app into a shared community graph for multiple interconnected families.

Working name: `ftree` (product name TBD).

## Status

Pre-implementation. The design has been validated through brainstorming.

- **Design spec:** [`docs/superpowers/specs/2026-05-23-ftree-platform-design.md`](docs/superpowers/specs/2026-05-23-ftree-platform-design.md)
- **Next step:** implementation plan (via the superpowers `writing-plans` workflow), then Flutter scaffold.

## Headline decisions

| Topic | Decision |
|---|---|
| Framework | Flutter / Dart |
| Platforms | Android + iOS (MVP); Web (Phase 4) |
| Backend | Firebase (Firestore + Auth + FCM + Cloud Storage) |
| Auth | Phone OTP (primary) + email (fallback) |
| Architecture | Single global community graph; families as logical sub-branches |
| Roles | Super admin → Family admins → Viewers (invite-gated) |
| Cross-family edits | Time-bounded consensus (7-day default), `*` flag, super-admin arbitration |
| Seed data | Migrate the existing Cheleri SQLite (1,499 individuals) |
| Visual style | Heritage Warm (cream + walnut + antique gold) |

See the design doc for the full architecture, data model, user flows, migration plan, and phasing.
