# Apple Foundation Models

Resources and notes for developing with Apple's Foundation Models framework (`FoundationModels`, iOS/macOS 26+).

---

## `@Generable` — Optional parameters generate `0` / `0.0` instead of `nil`

**Official reference:** [Generable — Apple Developer Documentation](https://developer.apple.com/documentation/foundationmodels/generable)

### Problem

The on-device constrained-decoding model generates `0.0` for `Double?` fields (and `0` for `Int?` fields) in `@Generable` structs when it intends to omit a parameter, rather than generating `nil`. This causes any `providedCount` guard that treats a non-nil value as "provided" to count the placeholder as a real input, leading to a validation failure and a retry loop with the same wrong arguments.

Rephrasing `@Guide` descriptions or session instructions from "omit" to "set to nil" does not change model behavior.

### Workaround (applied in NMRAssistant, 2026-06-13)

Normalize `0.0 → nil` (and `0 → nil` for `Int?`) for all optional numeric parameters at the top of each tool's `call()`, before the `providedCount` check. A genuine user-supplied `0` is subsequently caught by existing `<= 0` validation with a clear error message.

### Future fix

`@Generable(representNilExplicitlyInGeneratedContent: true)` — reportedly available from iOS 26.4 / macOS 26.4. Not yet in public documentation as of 2026-06-14. Raises the minimum deployment target from 26.0 to 26.4.

> **Note:** This issue does not appear to be described in Apple's official documentation or developer forums as of 2026-06-14.
