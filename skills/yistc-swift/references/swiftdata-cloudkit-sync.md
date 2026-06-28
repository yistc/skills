# SwiftData + CloudKit/iCloud Sync

Use this reference when a task involves SwiftData automatic iCloud sync, CloudKit-backed `ModelContainer`, `cloudKitDatabase`, CloudKit schema, production schema deployment, or diagnosing cross-device sync failures.

## Default stance

Do not treat SwiftData + CloudKit as a harmless toggle. First decide whether CloudKit is the right sync layer for the product.

If the app already has a server-side source of truth, prefer keeping SwiftData as a local cache and syncing each device with that server. Adding CloudKit over the same data creates two sync systems, duplicate conflict rules, and unclear ownership.

For lightweight preferences, consider `NSUbiquitousKeyValueStore` instead of CloudKit-backed SwiftData.

## Official hard constraints

CloudKit-backed SwiftData uses Core Data/`NSPersistentCloudKitContainer` under the hood. Design for those constraints:

- Add the iCloud capability with CloudKit enabled, plus Background Modes with Remote notifications.
- Specify the intended CloudKit database explicitly when ambiguity is possible: `.private("iCloud...")` or `.none`.
- `cloudKitDatabase: .none` disables SwiftData automatic iCloud sync.
- Do not use `@Attribute(.unique)` or `#Unique`; CloudKit cannot enforce SwiftData uniqueness.
- Every non-relationship property in a synced model must be optional or have a property-level default value.
- Every relationship must be optional, including to-many arrays.
- Relationships should have explicit inverses. Do not rely on inference for synced models.
- The `.deny` delete rule is not supported for CloudKit sync.
- Treat synchronization as eventual consistency. The app must work with partial, missing, delayed, duplicated, and out-of-order data.

## Production schema rule

CloudKit production schema is additive-only.

After record types or fields are deployed to production:

- Do not delete model types or fields.
- Do not rename model types or fields.
- Do not change field types.
- Add new fields or model types instead.
- Keep old fields/model types in the schema and deprecate them in code.

This is the source of the common rule: "CloudKit-synced schema entries cannot be removed after production; only abandoned."

Development schema can be reset only before it is promoted. If production already has the schema, resetting development reverts it to production's shape.

## Model design guidance

For synced models:

- Prefer flat query fields over relationship predicates.
- Keep natural keys as ordinary fields and implement app-level dedupe.
- Add stable IDs that are independent from `PersistentIdentifier`; persistent IDs are temporary before first save.
- Avoid large blobs in synced models. For `Data`, use `@Attribute(.externalStorage)` where appropriate, but remember it is a hint.
- Avoid deeply normalized graphs and many-to-many relationships unless the product absolutely needs them.
- Keep delete rules explicit. Use `.cascade` only when remote delete semantics are acceptable.
- Keep derived values either recomputable or defensively repairable.
- Avoid field values that depend on other fields unless the app has validation and repair logic.

For CloudKit-compatible to-many relationships, expose read-only computed helpers if needed:

```swift
@Relationship(deleteRule: .cascade, inverse: \Entry.feed)
var entriesRaw: [Entry]? = []

var entries: [Entry] {
  entriesRaw ?? []
}
```

Writes still need to mutate the stored optional relationship carefully.

## Predicate and query pitfalls

SwiftData predicates support only a subset of Swift. CloudKit's optional relationship requirement makes this worse.

Avoid predicates over optional to-many relationships. Patterns such as optional chaining, `flatMap`, `contains(where:)`, or nil checks on optional to-many relationships can compile and then crash at runtime with errors such as `to-many key not allowed here`.

Prefer:

- Querying the child/to-one side, then deriving parents in memory.
- Storing denormalized scalar fields used by list filters, such as `feedID`, `status`, `publishedAt`, `starred`.
- Filtering by stable IDs gathered from a prior query.
- Keeping UI lists based on scalar indexed fields rather than relationship traversal.

## Migration strategy

Before enabling CloudKit:

1. Create versioned schemas and a named `SchemaMigrationPlan`.
2. Make the schema CloudKit-safe before schema initialization.
3. Add structural tests that fail if a synced attribute is non-optional without a default.
4. Add structural tests that fail if a synced relationship is non-optional or lacks an explicit inverse.
5. Add migration tests from old on-disk stores, not just fresh installs.
6. Test upgrades with messy real-world data.

After production schema deployment:

- Use add-only migrations.
- Add optional/defaulted fields, then backfill.
- If a required semantic field is needed, use a bridge-version strategy: add optional/defaulted field first, populate it, then enforce app-level invariants later.
- Do not ship a destructive schema cleanup just because SwiftData local migration allows it; CloudKit production may not.

## Schema initialization and release gates

During development, initialize the CloudKit development schema using the Core Data bridge Apple documents:

- Build an `NSPersistentStoreDescription` using the same SwiftData store URL.
- Attach `NSPersistentCloudKitContainerOptions`.
- Load synchronously.
- Build an `NSManagedObjectModel` from the same SwiftData model types.
- Call `initializeCloudKitSchema()`.
- Remove/unload the store before creating the SwiftData `ModelContainer`.
- Keep this behind `#if DEBUG` or an explicit development-only path.

Before TestFlight/App Store:

- Verify CloudKit Console has every expected record type, field, relationship, and index.
- Deploy schema changes to production.
- Confirm there are no pending undeployed schema changes.
- Test the production environment with real devices.
- Confirm entitlements for iOS and macOS builds, including `com.apple.developer.icloud-container-environment`.

TestFlight and App Store builds use production CloudKit by default. Debug builds often use development. Do not infer production readiness from Xcode-only testing.

## Performance and reliability pitfalls

SwiftData + CloudKit has no app-level "sync now and wait" API. The system decides when exports and imports run.

Expect:

- Save writes to the local store first; export happens later.
- Import depends on silent push for private/shared databases.
- Public database changes are polled and can be much slower.
- The system can throttle imports/exports for seconds or hours.
- Large initial sync can be slow or appear stalled.
- Large records, many fields, or high write rates can hit CloudKit limits.
- Simulators are unreliable for iCloud sync testing; use physical devices.
- UI may need refresh logic after remote imports.

Design the UI to show local state and tolerate delayed convergence. Do not promise immediate cross-device propagation.

## Diagnostics

Use Apple's TN3163/TN3164 flow for serious sync failures:

1. Reproduce with exact timestamps on the affected devices.
2. Check CloudKit Console: if the record is absent, focus on export from source device; if present, focus on notification/import/UI on destination device.
3. Capture sysdiagnose from relevant devices soon after the issue.
4. Inspect `system_logs.logarchive` with Console.app or `log`.
5. Filter for `com.apple.coredata`, `CoreData+CloudKit`, `cloudd`, `apsd`, and `dasd`.
6. Look for setup, export, import, notification, permission, schema mismatch, limit, and throttle messages.

Common failure classes:

- Missing or invalid CloudKit/App ID entitlements.
- Development schema not deployed to production.
- CloudKit schema does not mirror the SwiftData/Core Data model.
- Record too large or too many fields.
- Multiple persistent containers loading the same CloudKit-backed store.
- Device not logged into iCloud, app iCloud disabled, or account switching.
- System throttling after heavy write bursts.

## RSS/client app guidance

For RSS reader or API-client apps with a server source of truth such as Miniflux:

- Do not CloudKit-sync the server-backed cache by default.
- Let each device sync independently with the server.
- Keep offline mutation queues local unless there is a concrete cross-device offline workflow.
- Dedupe by server keys in app logic, not CloudKit uniqueness.
- Avoid syncing credentials through CloudKit. Keep tokens in Keychain unless the product explicitly supports secure credential sync.
- Sync only small app preferences through a lightweight mechanism if needed.

If the user insists on CloudKit sync for server-backed data, first write a design doc that defines the single source of truth, conflict resolution, dedupe, migration path, and failure recovery.

## Sources to verify when needed

- Apple: SwiftData "Syncing model data across a person's devices"
- Apple: CloudKit "Deploying an iCloud Container's Schema"
- Apple: CloudKit "Inspecting and Editing an iCloud Container's Schema"
- Apple TN3163: synchronization of `NSPersistentCloudKitContainer`
- Apple TN3164: debugging synchronization of `NSPersistentCloudKitContainer`
- Fatbobman: rules for adapting Core Data/SwiftData models to CloudKit
- Hacking with Swift: syncing SwiftData with CloudKit
