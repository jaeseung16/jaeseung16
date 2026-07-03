# Core Data & Swift Concurrency

Patterns, pitfalls, and fixes for using Core Data with Swift Concurrency (actors, async/await).

---

## Issues & Case Studies

- [**BawiBrowser SIGABRT crash** — viewContext mutation race on actor executor](bawi-browser-sigabrt-crash.md)  
  A Core Data context confined to the main queue was accessed from an actor's executor thread, causing concurrent mutation of the internal pending-changes set. Intermittent crash when posting comments coincided with main-thread work (CloudKit merges, SwiftUI faulting).

---

## Key Principles

- `NSManagedObjectContext` has its own queue-confinement model; wrapping it in an `actor` does not make it thread-safe.
- Main-queue contexts (`viewContext`) must always be accessed via `perform`/`performAndWait` on the main queue.
- Background contexts must always use `performAndWait` for synchronous access or `perform` for async access.
- Enable `-com.apple.CoreData.ConcurrencyDebug 1` in Debug schemes to catch confinement violations immediately.
