# Platform-Conditional Frameworks

Patterns, pitfalls, and fixes for depending on Apple frameworks that don't exist — or are
non-functional stubs — on every platform the app binary can run on (FinanceKit, ActivityKit,
CarPlay, etc.). The core trap: **an iOS app running on a Mac ("Designed for iPad") is the
same binary that runs on the iPad**, so no compile-time condition can produce a
Mac-specific build.

---

## Issues & Case Studies

- [**SpendingKeeper dyld launch crash on Mac** — strong link to FinanceKitUI stub](spendingkeeper-financekit-dyld-crash.md)  
  TestFlight installs on macOS aborted at launch with `Termination Reason: DYLD, Symbol
  missing` for `FinanceKitUI.TransactionPicker.init(selection:label:)`. The macOS
  `/System/iOSSupport` copy of FinanceKitUI exists but doesn't export the symbol, and the
  binary linked it strongly. Fixed with `-weak_framework` plus a runtime
  `isiOSAppOnMac` guard.

---

## Choosing the right check

Each mechanism answers a different question. For "iPad app on Mac" only the last row helps.

| Check | Kind | Question it answers | iPad app on Mac |
|---|---|---|---|
| `#if canImport(X)` | compile-time | Does the SDK I'm building against have module X? | true (built with iOS SDK) |
| `#if os(iOS)` | compile-time | Which OS is this binary compiled for? | true (it *is* an iOS build) |
| `#if targetEnvironment(macCatalyst)` | compile-time | Is this a Catalyst build? | false (separate build you may not have) |
| `if #available(iOS N, *)` | runtime | Is the running OS version ≥ N? | Passes — macOS reports its iOS *compatibility version* (macOS 26 ≈ iOS 26) |
| `ProcessInfo.processInfo.isiOSAppOnMac` | runtime | Is this iOS binary running on a Mac? | **true — the only check that works** |

## Key Principles

- **Compile-time checks select code paths per *build*, not per *device*.** "Designed for
  iPad" on Apple silicon downloads and runs the literal iOS binary; `#if canImport` /
  `#if os(iOS)` are both true when it's built and cannot exclude anything for the Mac.
- **`import SomeFramework` auto-links strongly.** Swift emits an autolink directive; if a
  referenced symbol is missing at launch, dyld kills the process before `main()` runs —
  no code of yours, no crash-reporting SDK, nothing gets a chance to react.
- **Availability annotations only weak-link APIs *newer than your deployment target*.**
  If an API was introduced in iOS 17.4 and your deployment target is iOS 26, the compiler
  emits a strong reference. Raising the deployment target can silently convert previously
  weak imports into strong ones.
- **The fix is always two parts:** (1) `-weak_framework X` in `OTHER_LDFLAGS` so dyld
  resolves missing symbols to null instead of aborting at launch (this overrides Swift's
  strong autolink), and (2) a runtime guard (`isiOSAppOnMac`) so the null symbol is never
  actually called — calling it crashes at the call site instead of at launch.
- **Guard every entry point, not just the menu.** App Intents / Shortcuts, widgets, URL
  handlers, and state restoration can navigate straight to a screen whose UI entry you hid.
- **Verify the linkage in the built binary**, not just a successful build:
  ```bash
  # Framework should appear as LC_LOAD_WEAK_DYLIB, not LC_LOAD_DYLIB
  otool -l MyApp.app/MyApp | grep -B4 SomeFramework | grep "cmd \|name"
  # The specific symbol should be "(undefined) weak external"
  nm -m MyApp.app/MyApp | grep SomeSymbol
  ```
  In Debug builds with Xcode 16+ the app code lives in `MyApp.debug.dylib` inside the
  bundle — inspect that file, not the stub executable.
- **A crash log tells you which case you're in.** `Termination Reason: Namespace DYLD,
  Code 4, Symbol missing` + `Expected in: /System/iOSSupport/...` + `AppVariant:
  1:MacFamily...` = iOS binary on Mac hitting a hollow iOSSupport framework.
- The blunt alternative — unchecking **Mac (Designed for iPad)** in Supported
  Destinations — drops Mac users entirely. Prefer weak-link + runtime guard so the app
  keeps running on the Mac with the feature hidden.
