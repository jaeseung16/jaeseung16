# Crash Report Analysis: SIGABRT in `-[NSManagedObjectContext save:]` When Posting Comments

**Date:** 2026-07-03
**Affected version:** BawiBrowser 0.7 (2), PersistenceSwift 0.2.17
**Incident:** FF71B61C-52B7-4203-8424-4C3A2E04DE5C (2026-06-27, macOS 26.5.1)
**Status:** Fixed (pending PersistenceSwift 0.2.18 release)

## Symptom

Posting a comment occasionally killed the app instantly. The crash report showed
`EXC_CRASH (SIGABRT)` triggered by an uncaught Objective-C exception rethrown out of
`-[NSManagedObjectContext save:]`:

```
Last Exception Backtrace:
  -[__NSCFSet removeObject:].cold.1            ← collection mutated concurrently
  -[NSManagedObjectContext _processPendingUpdates:]
  -[NSManagedObjectContext _processRecentChanges:]
  -[NSManagedObjectContext save:]
  Persistence.save()                            (Persistence.swift)
  PersistenceHelper.save(comment:)              (PersistenceHelper.swift:37)
  closure #1 in BawiBrowserViewModel.saveComment(_:)  (BawiBrowserViewModel.swift:268)
```

The decisive clue: the crashed thread ran on
`com.apple.root.user-initiated-qos.cooperative` — the Swift Concurrency thread pool,
**not the main thread**.

## Root cause

`NSPersistentContainer.viewContext` is a *main-queue-confined* context: every touch of
it must happen on the main queue (or inside `perform`/`performAndWait`).

`Persistence` (in the PersistenceSwift package) is an `actor`, so its methods execute on
the actor's executor — a cooperative-pool thread. `Persistence.save()` called
`container.viewContext.save()` directly from that executor. Whenever the main thread was
mutating the same context at that moment (SwiftUI faulting objects, `fetchAll()`, a
CloudKit remote-change merge), the context's internal pending-changes `NSMutableSet` was
mutated from two threads at once → "collection was mutated while being enumerated" →
`SIGABRT`.

The intermittency is inherent to the race: posting a comment fires the off-main save at
the same moment the `WKWebView` navigation completes and main-thread work runs. Most of
the time the windows miss each other; occasionally they collide.

### All violations found (same bug class)

| Location | Violation |
|---|---|
| `Persistence.save()` | `viewContext.save()` on the actor executor — **the reported crash** |
| `Persistence.save(completionHandler:)` | `viewContext.rollback()` on the actor executor |
| `Persistence.save(with:completionHandler:)` | `viewContext.name` mutated on the actor executor |
| `Persistence.count(_:)` | `viewContext.count(for:)` from any caller thread |
| `HistoryRequestHandler.fetchUpdates()` | `viewContext.mergeChanges(...)` on the actor executor — fires on **every CloudKit remote-change notification**; the most likely "other thread" in the race |
| `HistoryRequestHandler.purgeHistory()` / `fetchHistoryTransactions()` | `execute(...)` on a private-queue background context without `perform` — trapped at app launch once assertions were enabled |
| App: `PersistenceHelper` | Declared `Sendable` with nonisolated async methods; entity inserts (`Comment(context:)` etc.) could run on the cooperative pool |
| App: `SafariExtensionPersister` | An actor inserting, fetching, and saving the viewContext directly from its executor — the Safari extension could crash the same way |

## Fixes

### PersistenceSwift (`~/Developer/Persistence`, to be tagged 0.2.18)

All viewContext access now goes through the context's own queue:

- `save()` wraps the `hasChanges` check and `save()` in `await viewContext.perform { }`.
- The error-path `rollback()` and the `save(with:)` name juggling are wrapped the same way.
- `count(_:)` uses `viewContext.performAndWait { }`.
- `HistoryRequestHandler.fetchUpdates()` merges history notifications inside
  `viewContext.perform { }`.
- `purgeHistory()` and `fetchHistoryTransactions()` run `execute(...)` inside
  `backgroundContext.performAndWait { }`.

### BawiBrowser app (branch `0.8`)

- `PersistenceHelper` is now `@MainActor` and no longer claims `Sendable`, so all entity
  creation, fetches, and deletes are pinned to the viewContext's queue. (Its callers all
  live on the main actor already.)
- `SafariExtensionPersister` wraps every viewContext operation (insert, fetch, save,
  `transactionAuthor` set/reset) in `performAndWait`.
- The shared scheme's Debug run now passes `-com.apple.CoreData.ConcurrencyDebug 1`,
  which turns any future confinement violation into an immediate, pinpointed trap
  instead of a rare production crash. Test runs inherit it.
- Added a regression test, `testSaveCommentRespectsContextConfinement`, that drives the
  exact crash path (`PersistenceHelper.save(comment:)` → `Persistence.save()`) against
  an in-memory store with the assertions active.
- Housekeeping: the three test targets had `MACOSX_DEPLOYMENT_TARGET = 13.0` vs. the
  app's 26.0, so unit tests no longer compiled; bumped to 26.0.

## Verification

| Check | Before fix | After fix |
|---|---|---|
| Launch app with `-com.apple.CoreData.ConcurrencyDebug 1` | Aborts within seconds (SIGTRAP at first violation) | Runs cleanly through heavy CloudKit import/save activity |
| Unit tests (assertions active), incl. regression test | Test host crashed before bootstrapping | All pass |
| `swift build` / `swift test` on the package | — | Clean, zero warnings |

## Remaining steps

1. Commit the PersistenceSwift changes, tag `0.2.18`, push to GitHub.
2. Switch the Xcode project's PersistenceSwift reference back from the temporary local
   package (`../../Persistence`) to the remote pin at `0.2.18`.
3. Optional, longer term: perform writes on a background context
   (`container.performBackgroundTask` with `automaticallyMergesChangesFromParent = true`)
   and keep the viewContext read-only for the UI.

## Lesson

An `actor` does not make Core Data thread-safe. `NSManagedObjectContext` has its own
queue-confinement model that predates Swift Concurrency; wrapping a context in an actor
merely moves the illegal access off the main thread. The context's queue — via
`perform`/`performAndWait` — is the only safe door, and
`-com.apple.CoreData.ConcurrencyDebug 1` is the cheap way to keep that door honest.
