# Conventions

Shared conventions for the **localfiles** family of iOS apps:
[localgallery](https://github.com/j23n/localgallery),
[localcontacts](https://github.com/j23n/localcontacts),
[localmusic](https://github.com/j23n/localmusic).

These apps all do the same thing: make a folder of files (photos, vCards, audio)
a first-class iOS citizen — no library import, no cloud account, the file system
is the source of truth. They should look, feel, and read like one product.

This document is the **single canonical source** for cross-app standards.
It lives at the root of [localapps](https://github.com/j23n/localapps).
Each app's `.claude/CONVENTIONS.md` is a one-line stub that points here
(see [§18 Sync](#18-sync) for the stub format).

## How to read this document

- **Rules** are prescriptive ("do X, not Y") with rationale.
- **Per-app status** tables track where each repo currently stands vs. the rule.
  Cells marked with a GitHub issue link are tracked migrations.

---

## 1. Per-app status snapshot

| Convention | localgallery | localcontacts | localmusic |
|---|---|---|---|
| Deployment target iOS 18 | ✅ | ✅ | ✅ |
| Swift 6 strict concurrency | ✅ | ✅ | ✅ |
| `@Observable` state | ✅ | ✅ | ✅ |
| Decomposed Store + Services | ✅ | ✅ | ✅ |
| `Models/` `Services/` `Views/` layout | ✅ | ✅ | ✅ |
| `Log.<category>` (`os.Logger`) | ✅ | ✅ | ✅ |
| SHA-256 stable IDs | ✅ | n/a | ✅ |
| Swift Testing | ✅ | ✅ | ✅ |
| Tests in CI (separate `test.yml`) | ✅ | ✅ | ✅ |
| `macos-26` runner | ✅ | ✅ | ✅ |
| Bundle ID `com.localX.app` | ✅ | ✅ | ✅ |
| Settings UX (`List` + sections) | ✅ canonical | ✅ | ✅ |
| Folder access flow | ✅ | ✅ | ✅ |
| Atomic file writes | ✅ | ✅ | ✅ |
| Scene-phase rescan | ❌ | ✅ | ✅ |

Each `❌` should be tracked by a migration issue in the relevant repo.

---

## 2. Project layout

Each app is a single Xcode project generated from `project.yml` via
[XcodeGen](https://github.com/yonaskolb/XcodeGen). The `.xcodeproj` is **not
checked in** — `xcodegen` regenerates it from `project.yml` on demand.

```
LocalX.xcodeproj         # generated, gitignored
LocalX/
  LocalXApp.swift        # @main entry; tabs / scene wiring only
  Models/                # value types: Contact, Track, PhotoFile, …
  Services/              # I/O, parsers, stores: ContactsStore, MetadataLoader, …
  Views/                 # SwiftUI views, one per file
  Components/            # shared SwiftUI primitives (ThumbnailView, etc.)
  Shared/                # only when a second target needs the types
  Logging.swift          # Log.<category> namespace (see §6)
  Design.swift           # design tokens (see §10)
  Info.plist
  LocalX.entitlements
  Assets.xcassets/
LocalXTests/
  README.md              # documents test conventions + refactor seams
  Fixtures.swift         # shared builders
  …Tests.swift
project.yml
README.md
LICENSE                  # MPL 2.0
.claude/
  CLAUDE.md              # repo-specific Claude Code guide (optional)
  CONVENTIONS.md         # one-line stub → localapps CONVENTIONS.md (see §18)
.github/workflows/
  build.yml              # archive + IPA on tag (see §15)
  test.yml               # tests on PR (see §15)
.gitignore
```

**Why subfolders.** Xcode is happy with a flat directory, but once an app has
20+ source files, navigation cost compounds. Subfolders are the cheapest
intervention.

**Why `Shared/`.** When a second target (a widget extension, a watch app)
needs to compile some types, put them in `Shared/` and list the path under
both targets in `project.yml`. Cross-process state goes through files in the
App Group, never via in-memory references — see localgallery's `project.yml`
comment for the canonical statement.

---

## 3. Build settings

Set these in every `project.yml`:

```yaml
options:
  deploymentTarget:
    iOS: "18.0"
  xcodeVersion: "16.0"

settings:
  base:
    SWIFT_VERSION: "6.0"
    IPHONEOS_DEPLOYMENT_TARGET: "18.0"
    SWIFT_STRICT_CONCURRENCY: complete
```

**iOS 18 baseline.** Current iOS is 26 (April 2026). iOS 18 is two majors back —
covers ~95 % of in-use iPhones and unlocks `@Observable` propagation, modern
`NavigationStack` behaviour, and Swift 6 concurrency. Bumping further (iOS 19+)
is fine when the App Store usage data justifies it.

**Swift 6 with `SWIFT_STRICT_CONCURRENCY: complete`.** This is non-negotiable.
The whole architecture (next section) depends on the compiler enforcing actor
isolation. Use of `@unchecked Sendable` is allowed only with a comment
explaining what invariant the human is upholding instead of the compiler.

**iOS 26 features** (e.g. `scrollEdgeEffectStyle(.soft)`) may be adopted via
`#available(iOS 26.0, *)` checks for progressive enhancement. localgallery's
`softTopScrollEdge()` extension is the model.

---

## 4. State management

Use Apple's **Observation framework** (`@Observable`), not Combine
(`ObservableObject` / `@Published` / `@StateObject` / `@EnvironmentObject`).

```swift
import Observation

@Observable
@MainActor
final class ContactsStore {
    var contacts: [Contact] = []
    var folderURL: URL?
    // …
}
```

In views:

```swift
struct ContactListView: View {
    @Environment(ContactsStore.self) private var store
    // …
}
```

In the app entry:

```swift
@main
struct LocalContactsApp: App {
    @State private var store = ContactsStore()

    var body: some Scene {
        WindowGroup {
            ContentView()
                .environment(store)
        }
    }
}
```

**Why.** `@Observable` re-evaluates only views that *read* a changed property
(field-level granularity), avoids `@Published`'s Combine overhead, and composes
cleanly with Swift 6 isolation. `ObservableObject` is the iOS 13–16 pattern.

**One Store per app, decomposed by responsibility.** The Store owns
view-facing state. I/O and protocol concerns (bookmarks, parsers, system
integrations) live in separate service types it composes. localcontacts is
the canonical example: `ContactsStore` + `BookmarkManager` +
`FolderAccessManager` + `CNSyncService` + `VCardParser` + `VCardWriter`.

**Avoid `static var shared` on the Store.** A SwiftUI-injected `@State`
lifetime is enough. If a non-SwiftUI entry point (BG task handler, app
extension) needs access, expose a small actor or value-type service for that
specific need — don't hand it the whole store.

---

## 5. Folder access (security-scoped bookmarks)

This is the central feature of all three apps. The flow is identical and
should not be invented per app.

### 5.1 Document picker

A trivial wrapper, identical across all three:

```swift
struct DocumentPicker: UIViewControllerRepresentable {
    let onPick: (URL) -> Void

    func makeUIViewController(context: Context) -> UIDocumentPickerViewController {
        let picker = UIDocumentPickerViewController(forOpeningContentTypes: [.folder])
        picker.allowsMultipleSelection = false
        picker.delegate = context.coordinator
        return picker
    }

    func updateUIViewController(_: UIDocumentPickerViewController, context: Context) {}

    func makeCoordinator() -> Coordinator { Coordinator(onPick: onPick) }

    final class Coordinator: NSObject, UIDocumentPickerDelegate {
        let onPick: (URL) -> Void
        init(onPick: @escaping (URL) -> Void) { self.onPick = onPick }

        func documentPicker(_ controller: UIDocumentPickerViewController, didPickDocumentsAt urls: [URL]) {
            guard let url = urls.first else { return }
            onPick(url)
        }
    }
}
```

### 5.2 Picker callback dance

The picker hands you a URL with a transient security scope. **Claim it
synchronously in the callback**, save the bookmark, release, then dispatch
the rescan as a `Task`:

```swift
.sheet(isPresented: $showFolderPicker) {
    DocumentPicker { pickerURL in
        // Transient scope must be claimed and turned into a bookmark
        // synchronously — not from inside a Task.
        _ = pickerURL.startAccessingSecurityScopedResource()
        bookmarkManager.saveBookmark(for: pickerURL)
        pickerURL.stopAccessingSecurityScopedResource()
        Task { await store.adoptSavedFolder() }
    }
}
```

### 5.3 Bookmark service

Bookmark persistence is its own type with `UserDefaults` injected (see §13
on testability):

```swift
final class BookmarkManager {
    private let defaults: UserDefaults
    static let bookmarkKey = "folderBookmark"

    init(userDefaults: UserDefaults = .standard) { self.defaults = userDefaults }

    func saveBookmark(for url: URL) throws { /* … */ }
    func loadBookmark() -> URL? { /* with isStale handling */ }
    func clearBookmark() { /* … */ }
}
```

### 5.4 Lifecycle: restore on launch + rescan on resume

Restore once on launch via `.task`; rescan when the app returns to the
foreground via `scenePhase`. Local files can change while the app is
backgrounded (Files.app, Syncthing, etc.) — re-checking on resume is the
right default.

```swift
@Environment(\.scenePhase) private var scenePhase

WindowGroup {
    ContentView()
        .environment(store)
        .task { await store.restoreFolder() }
        .onChange(of: scenePhase) { _, phase in
            if phase == .active {
                Task { await store.checkForExternalChanges() }
            }
        }
}
```

---

## 6. Logging

Use `os.Logger` through a `Log` namespace. Never use `print(...)` outside
test code.

```swift
import os

enum Log {
    private static let subsystem = "localcontacts"

    static let scan   = Logger(subsystem: subsystem, category: "scan")
    static let store  = Logger(subsystem: subsystem, category: "store")
    static let sync   = Logger(subsystem: subsystem, category: "sync")
    static let ui     = Logger(subsystem: subsystem, category: "ui")
}

// Usage:
Log.scan.info("Scanning folder \(url.path, privacy: .private)")
Log.store.error("Failed to save: \(error.localizedDescription)")
```

**Why.** `os.Logger` integrates with Console.app, supports privacy redaction,
filters by category and level, has lazy interpolation (free when filtered
out), and survives release builds. `print` writes to stderr and disappears.

**Gotcha.** `os.Logger` interpolations are `@escaping @autoclosure`. Inside
a closure you must write `self.foo` explicitly — the compiler will complain
otherwise.

**Categories** are app-specific (gallery has `scan`, `enrich`, `thumb`,
`cache`, `index`, `memory`, `bg`, `widget`, …). The set should reflect the
app's actual subsystems, not be copy-pasted.

---

## 7. Settings sheet

Every app has a Settings sheet, and it should look the same. localgallery's
is the canonical version; all three apps already follow it closely.

### Shape

```swift
struct SettingsView: View {
    @Environment(Store.self) private var store
    @Environment(\.dismiss) private var dismiss
    @State private var showFolderPicker = false

    var body: some View {
        NavigationStack {
            List {
                // 1. Folder section — always first
                Section("<Domain> Folder") {  // "Photo Library", "Contacts Folder", "Music Folder"
                    Button {
                        showFolderPicker = true
                    } label: {
                        LabeledContent {
                            Text(store.folderURL?.lastPathComponent ?? "Not selected")
                                .foregroundStyle(.secondary)
                        } label: {
                            Label("Folder", systemImage: "folder")
                        }
                    }
                    .tint(.primary)        // suppress accent on the chevron

                    Button("Reload <Domain>") {     // "Reload Library", "Reload Contacts", "Reload Music"
                        Task { await store.rescan() }
                    }

                    if let lastSync = store.lastSyncedAt {
                        LabeledContent("Last Synced") {
                            Text(lastSync, style: .relative)
                                .foregroundStyle(.secondary)
                        }
                    }
                }

                // 2. Domain-specific sections (People, Tags, Sync, …)

                // 3. Info section — always last data section
                Section("Info") {
                    LabeledContent("<Items>", value: "\(store.items.count)")
                    LabeledContent("Version", value: appVersion)
                }
            }
            .navigationTitle("Settings")
            .navigationBarTitleDisplayMode(.inline)
            .toolbar {
                ToolbarItem(placement: .confirmationAction) {
                    Button("Done") { dismiss() }
                }
            }
            .sheet(isPresented: $showFolderPicker) { /* picker dance from §5.2 */ }
        }
    }
}
```

### Rules

- **Folder section first**, **Info section last**, domain sections in between.
- **Folder row** uses `LabeledContent` + `Label("Folder", systemImage: "folder")`.
- `.tint(.primary)` on the folder Button to suppress accent on the chevron
  (otherwise it inherits the app accent, which clashes with the muted label).
- **Reload button** label is `Reload <Domain>`, not "Refresh" or "Sync".
- **Done** in `.toolbar` at `.confirmationAction`.
- **`.navigationBarTitleDisplayMode(.inline)`** — the small inline title.
- **Counts in Info** — single source of truth for "how big is this thing."

---

## 8. App shell & navigation

### When to use a TabView

If the app has **two or more independent top-level surfaces**, use a TabView
with one `NavigationStack` per tab. Otherwise (localcontacts) just have a
single root view that switches between an empty-state folder picker and the
main content.

```swift
// Multi-surface (gallery, music)
TabView(selection: $router.selectedTab) {
    NavigationStack(path: $router.foldersPath) { FolderBrowserView() /* … */ }
        .tabItem { Label("Folders", systemImage: "folder") }

    NavigationStack { AllPhotosView() }
        .tabItem { Label("Photos", systemImage: "square.stack.3d.up") }
}

// Single-surface (contacts)
Group {
    if store.folderURL != nil {
        ContactListView()
    } else {
        FolderPickerView()
    }
}
```

### Deep-link router

Apps with deep links (widgets, notifications) have a small `AppRouter`
`@Observable` object that holds `selectedTab` and per-tab `NavigationPath`s,
and consumes pending route ids when the backing data is ready. localgallery's
`AppRouter` is the model.

### Settings access

Settings is a `.sheet` opened from a toolbar button on the root of each
top-level tab — never a tab of its own. This keeps the tab bar focused on
content surfaces.

---

## 9. Stable IDs

When a model represents an on-disk file (Track, PhotoFile, Contact),
**derive its ID from the file URL via SHA-256**. This keeps SwiftUI list
identity stable across rescans (no flicker, no scroll jump, no selection
loss) without storing IDs on disk.

```swift
import CryptoKit

extension Track {
    static func stableID(for url: URL) -> UUID {
        let path = url.standardized.path
        let digest = SHA256.hash(data: Data(path.utf8))
        var bytes = Array(digest.prefix(16))
        // RFC 4122 layout — variant + version-5 nibbles. Foundation's UUID
        // only validates the layout, so this is well-formed even without a
        // strict v5 namespace input.
        bytes[6] = (bytes[6] & 0x0F) | 0x50
        bytes[8] = (bytes[8] & 0x3F) | 0x80
        return UUID(uuid: (bytes[0],  bytes[1],  bytes[2],  bytes[3],
                           bytes[4],  bytes[5],  bytes[6],  bytes[7],
                           bytes[8],  bytes[9],  bytes[10], bytes[11],
                           bytes[12], bytes[13], bytes[14], bytes[15]))
    }
}
```

**Standardize the URL** before hashing (`url.standardized.path`) — otherwise
`/Users/foo/x` and `/private/Users/foo/x` collide differently.

**Why SHA-256, not MD5.** Both work. SHA-256 is the better default in 2026
and CryptoKit makes them equally easy. All three apps are now on SHA-256.
