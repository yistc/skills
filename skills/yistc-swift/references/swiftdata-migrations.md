# SwiftData Migrations

Agent rules for SwiftData migration work.

## Scope
- Use for `VersionedSchema`, `SchemaMigrationPlan`, `MigrationStage`, `ModelContainer(... migrationPlan:)`, legacy `.sqlite` stores, and migration tests.
- Do not build a generic migration engine. SwiftData internals are not stable/transparent enough.
- Keep concrete schemas, model classes, stages, and data transforms in each app.

## Implementation
- Add a migration plan early, even with one current schema and empty `stages`.
- Register the plan in `ModelContainer(for:migrationPlan:configurations:)`.
- Centralize container creation in a factory so tests can pass a temporary store URL.
- Explicitly list every persistent model in each schema's `models`; include relationship-only models.
- Use a `CurrentSchema` alias, then alias app model names through it.
- Do not create a fake new schema version by typealiasing old `@Model` classes and adding a lightweight stage. This can crash in SwiftData/CoreData.
- For real schema bumps, define distinct historical model classes and explicit stages.

## Duplicate Version Checksums
- Treat `Duplicate version checksums across stages detected` as an invalid migration graph, not as user data corruption.
- SwiftData/Core Data staged migration uses model version checksums derived from the persistent model graph, not just the `VersionedSchema` enum name or `versionIdentifier`.
- Do not build a new schema version by reusing previous models with `typealias`, for example `typealias Entry = SchemaV1.Entry`. Each schema version must be a complete model snapshot with distinct nested `@Model` type identities.
- Inline-default property additions such as `var inputTokens: Int = 0` can be lightweight/automatic. Do not add a fake schema stage only to represent changes SwiftData can already infer unless a real shipped migration requires it.
- If the product has not shipped to users, prefer squashing historical schemas into one current schema version and deleting/recreating local dev stores when incompatible.
- If the product has shipped, preserve historical schemas by copying the full model set into each `VersionedSchema`; never typealias old `@Model` classes into the new version.
- Add a real app/container initialization test that uses `ModelContainer(for:migrationPlan:configurations:)`; tests that create only `Schema(versionedSchema:)` without a migration plan will not catch duplicate checksum crashes.

## Legacy Stores
- If the app shipped before `VersionedSchema`, inspect a real old store before choosing versions.
- If needed, add a legacy anchor schema that exactly matches the old model graph and version metadata.
- Do not copy another app's legacy version like `1.0.0` unless a real store proves it.
- Test migration from real or fixture old stores, not only empty stores.

## Tests
- Use isolated temp store URLs; never touch the user's real store in tests.
- Provide a reset helper that deletes `.sqlite`, `-wal`, and `-shm`.
- Test: plan shape, registered model completeness, create/reopen store, relationships/optional edges, and legacy/custom migration fixtures.
- After changes, run targeted migration tests, full tests when feasible, and launch the app once against a backed-up real dev store.

## Failure Policy
- If migration crashes, stop and inspect schema/version mismatch. Do not keep adding stages blindly.
- Prefer no-op scaffold over unsafe pseudo-migration.
- Back up real stores before launch verification.
