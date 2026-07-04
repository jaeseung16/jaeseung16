# dSYM (Debug SYmbol Map)

Debug symbol files and strategies for crash symbolication, debugging, and build configuration.

---

## Overview

**dSYM (Debug SYmbol Map)** is a bundle containing debug symbols extracted from an executable. It enables crash report analysis, debugging, and stack trace symbolication—converting memory addresses back to function names and line numbers.

---

## When dSYM is Needed

| Scenario | dSYM Required? | Rationale |
|----------|---|---|
| **Release / App Store builds** | ✅ Yes | Production app is stripped of symbols; dSYM is the only way to symbolicate user crashes. Mandatory. |
| **Debug builds on Simulator** | ⚠️ Optional | Debug symbols remain in the executable; dSYM adds build overhead without much local benefit. |
| **Debug builds on Device** | ✅ Recommended | Easier crash analysis during testing if app crashes occur. |

---

## Build Setting: DEBUG_INFORMATION_FORMAT

### Options

- **`dwarf`** — Debug info embedded in executable; no separate dSYM file. Faster builds for simulator.
- **`dwarf-with-dsym`** — Debug info in executable + separate dSYM bundle. Larger build output, but dSYM available for symbolication.
- **`stabs`** — Older format; not recommended for modern projects.

### SpendingKeeper Configuration

**Debug builds:** `dwarf-with-dsym`
- Ensures dSYM is available even during simulator testing.
- Minimal overhead on simulator builds.
- Consistency with Release configuration.
- Enables crash symbolication if simulator crashes occur during development.

**Release builds:** `dwarf-with-dsym`
- Essential for analyzing user crash reports from production.

---

## The Empty dSYM Warning

**Symptom:** Xcode log shows:
```
empty dSYM file detected, dSYM was created with an executable with no debug info.
```

**Cause:** Mismatch between `DEBUG_INFORMATION_FORMAT` setting and actual symbol generation.
- Build setting says `dwarf` (no dSYM), but linker still generates empty dSYM file.

**Fix:** Align build setting to actual behavior.
- Change `DEBUG_INFORMATION_FORMAT` from `dwarf` to `dwarf-with-dsym`.
- Ensures dSYM is properly generated with full debug symbols.

---

## Decision Matrix

| Goal | Recommendation | Trade-off |
|------|---|---|
| **Fast simulator builds, willing to sacrifice crash symbolication** | Use `dwarf` | No dSYM; need to keep executable for crash analysis |
| **Balance speed & symbolication** | Use `dwarf-with-dsym` (current) | Minimal build overhead; consistent with Release config |
| **Maximum crash debugging support** | Use `dwarf-with-dsym` + strip Release | Best of both worlds if Release dSYM is optional |

---

## Related

- Apple's [Debugging with Xcode](https://developer.apple.com/documentation/xcode/debugging-with-xcode)
- Apple's [Understanding Crash Reports](https://developer.apple.com/documentation/xcode/diagnosing-crashes)
