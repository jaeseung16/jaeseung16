# Crash Report Analysis: dyld "Symbol missing" at Launch on Mac (FinanceKitUI)

**Date:** 2026-07-04
**Affected version:** SpendingKeeper 1.6 (1), via TestFlight on macOS 26.5.2
**Incident:** 5EED6B79-07E1-40FF-9C4D-9647D22BA4B7
**Status:** Fixed in commit `8ebf454` (pending next TestFlight build)

## Symptom

The app ran fine on iPhone/iPad but died instantly at launch when installed on a Mac
through TestFlight. The crash report showed `EXC_CRASH (SIGABRT)` raised by **dyld
itself** — the process never reached `main()`:

```
Termination Reason:  Namespace DYLD, Code 4, Symbol missing
Symbol not found: _$s12FinanceKitUI17TransactionPickerV9selection5labelACyxG05SwiftC07BindingVySay0aB00D0VGG_xyXEtcfC
Referenced from: .../SpendingKeeper.app/SpendingKeeper
Expected in:     /System/iOSSupport/System/Library/Frameworks/FinanceKitUI.framework/...
(terminated at launch; ignore backtrace)
```

Demangled, the missing symbol is `FinanceKitUI.TransactionPicker.init(selection:label:)`.

The decisive clues:

- `AppVariant: 1:MacFamily20,1:26` — this is the **iOS binary running as "Designed for
  iPad" on Apple silicon**, not a Mac build. It is byte-for-byte the binary that works
  on the iPad.
- `Expected in: /System/iOSSupport/...` — on the Mac, iOS frameworks are served from the
  iOSSupport layer. Its FinanceKitUI is essentially a stub: the framework file exists,
  but it does not export `TransactionPicker`. FinanceKit/Apple Wallet transaction access
  is simply not functional on macOS.
- The backtrace is all `dyld4::prepare` → `halt` — a launch-time link failure, so no
  app code, logging, or crash-handling SDK ever ran.

## Root cause

`ImportsListView.swift` used `TransactionPicker` behind `#if canImport(FinanceKit)`, and
every other FinanceKit touchpoint was gated the same way. But `canImport` is a
**compile-time** check evaluated against the iOS SDK — where FinanceKit *is* importable —
so the code compiled in, and `import FinanceKitUI` auto-linked the framework **strongly**
(there was no explicit framework entry in the project; Swift autolink did it).

Two more mechanisms that look like they should help, don't:

- `#if os(iOS)` is also compile-time and is *true* for this binary. The Mac never gets
  its own compilation pass in the Designed-for-iPad model.
- `if #available(iOS 17.4, *)` fails twice over: (1) the crash happens before any code
  runs, and (2) availability on iPad-on-Mac evaluates against macOS's iOS
  *compatibility version* (macOS 26 reports ≈ iOS 26), so the check would pass anyway.

Why the reference was strong rather than weak: the compiler only emits weak references
for APIs introduced *after* the deployment target. `TransactionPicker` is
`@available(iOS 17.4, *)` and the deployment target is iOS 26.0, so a strong reference
was the correct default — for real iOS devices.

## Fixes

Two parts, both required:

**1. Weak-link the frameworks** so dyld tolerates the missing symbols at launch.
In the app target's `OTHER_LDFLAGS` (Debug and Release):

```
OTHER_LDFLAGS = (
    "$(inherited)",
    "-weak_framework", FinanceKit,
    "-weak_framework", FinanceKitUI,
);
```

`-weak_framework` overrides Swift's strong autolink; unresolved symbols become null
instead of fatal.

**2. Runtime-guard every path that could touch the null symbols** using
`ProcessInfo.processInfo.isiOSAppOnMac` — the only check that distinguishes
"iOS binary on a Mac" at runtime:

- `SKMenu.availableCases` filters out the "Import from Wallet" menu on Mac;
  `ContentView` iterates it instead of `allCases`.
- `ImportsListView` shows a `ContentUnavailableView` on Mac instead of instantiating
  `TransactionPicker`. This is defense in depth: the `ImportFromWallet` App Intent
  (Siri/Shortcuts) can navigate to that screen even with the menu item hidden, and
  calling through the null weak symbol would crash at the call site.

## Verification

- Build succeeds for the iOS simulator; unit tests pass.
- `otool -l` on the built binary shows both frameworks as `LC_LOAD_WEAK_DYLIB`
  (previously `LC_LOAD_DYLIB`).
- `nm -m` shows the exact mangled symbol from the crash report as
  `(undefined) weak external ... (from FinanceKitUI)`.
- Note: in Debug builds the app code lives in `SpendingKeeper.debug.dylib` inside the
  bundle (Xcode previews architecture) — inspect that, not the stub executable.
  Release applies the same flags to the main binary.
- Definitive confirmation requires the next TestFlight build launching on a Mac.

## Prevention checklist for adopting a new Apple framework

1. Check the framework's platform availability in the docs — "iOS" alone means it may be
   a hollow stub under `/System/iOSSupport` on the Mac.
2. If the app keeps **Mac (Designed for iPad)** as a destination, add
   `-weak_framework` for the new framework up front.
3. Gate the feature's UI *and* all indirect entry points (App Intents, widgets, URL
   handlers) behind `ProcessInfo.processInfo.isiOSAppOnMac`.
4. After building, confirm `LC_LOAD_WEAK_DYLIB` with `otool -l`.
5. Smoke-test a TestFlight/Release build on an Apple silicon Mac before shipping —
   the simulator and iPad will never catch this class of crash.
