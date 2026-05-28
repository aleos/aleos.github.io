---
title: "When to use assert, precondition, and fatalError"
date: 2025-11-17
draft: false
tags: [swift, ios, dev]
---

Swift has three runtime checks that look similar at first: `assert`, `precondition`, and `fatalError`. They differ in what gets stripped at compile time and, more importantly, in what they say about who caused the problem. Picking the wrong one either hides bugs in release builds or pushes failure handling to places that don't need it.

## The trap: silent fallbacks

Here's a solution to a LeetCode problem. The function takes an array that should contain numbers `1...n`, with one number duplicated and another missing. It returns both — `[duplicate, missing]`:

```swift
func findErrorNums(_ nums: [Int]) -> [Int] {
    var unseen = Set<Int>(1...nums.count)
    var duplicate = 0
    for num in nums {
        guard let _ = unseen.remove(num) else {
            duplicate = num
            continue
        }
    }
    let missing = unseen.first ?? 0
    return [duplicate, missing]
}
```

It works. Tests pass. But `?? 0` is hiding something. The valid range for both values is `1...n`. Zero is not a valid answer. If `unseen` is somehow empty, the function returns `[something, 0]` and the caller gets a confident-looking wrong result. The same problem hides in `var duplicate = 0` — if no duplicate is found, the function returns `[0, missing]` without any warning.

This is a LeetCode problem, so the stakes are low. But the pattern is real. In production code, you see the same situation constantly: a function with a fixed signature, documented input constraints, and an edge case where silently returning a wrong value seems harmless but isn't.

The `?? defaultValue` pattern is fine when the default is a *legitimate* value — an empty state the caller expects. But when the default sits *outside the valid range*, it's not a fallback. It's a way to mask a problem:

```swift
// Returns 0 when the valid range is 1...n
let missing = unseen.first ?? 0

// Returns empty string when "not found" should mean something else
return dictionary[key] ?? ""

// Returns .max when no value exists — indistinguishable from a real maximum
return distributedCookies.max() ?? .max
```

Sometimes these are fine. But each one hides a question worth asking: should this case even be possible? And if not, do I want to know about it, or do I want to sweep it under the rug?

## Why crashing can beat silently continuing

Crashing usually feels like the worst possible outcome. The app disappears, the user loses their work. Isn't it better to keep running, even with degraded behaviour?

Not when the function is returning a wrong value. A wrong value propagates through the system, causing secondary bugs far from the original cause. Debugging becomes harder because the symptom (wrong output) doesn't point to the root cause (invalid input). And the developer who made the mistake never finds out — the bug stays in production, producing subtly wrong results.

A crash, by contrast, surfaces the bug immediately at the right location, produces a stack trace, and makes it impossible to ignore.

The distinction that matters is between **recoverable errors** (network failures, malformed user input, missing optional data) and **contract violations** (the caller passed input the function was never designed to handle). Recoverable errors should be handled gracefully. Contract violations are programming mistakes, and a crash with a clear message is the fastest path to fixing them.

The `?? 0` pattern is nasty because it converts a contract violation into a silent wrong answer. The input was invalid, but the function pretended everything was fine.

## Three tools, one question

When an invariant breaks, the useful question is: who's responsible?

### `assert` / `assertionFailure` — "I think I made a mistake"

For sanity checks on your own code. Stripped in release builds.

```swift
assert(state == .ready, "State should be .ready here")
```

Use for intermediate state validation, assumptions about your own algorithm, conditions that "should be true if I wrote this correctly." Use when the check is expensive, or when being wrong is recoverable enough that you don't want production users to see a crash. The downside is that if your assumption is wrong in production, you never find out.

### `precondition` / `preconditionFailure` — "The caller broke the contract"

For invariants the caller is responsible for. Active in release builds. Stripped only in `-Ounchecked`, which almost nobody uses.

```swift
precondition(k >= 1, "Window size must be at least 1")
```

This is the one you'll use most often. It states a contract upfront and crashes with a clear message if the contract is violated. A crash in release isn't fun, but a silent wrong answer is worse.

### `fatalError` — "The world is broken"

For situations where continuing is impossible regardless of who caused it. Never optimised away, even with `-Ounchecked`.

```swift
required init?(coder: NSCoder) {
    fatalError("init(coder:) has not been implemented")
}
```

Use sparingly. Most "impossible" branches are actually contract violations (use `precondition`) or recoverable errors (return `nil` or throw). `fatalError` is the right choice when continuing would be genuinely dangerous, or when the language gives you no other option — for example, inside a `guard`'s `else` block where there's no value to return, no loop to break out of, and no error to throw.

## The optimization table

| Tool | Debug (`-Onone`) | Release (`-O`) | Unchecked (`-Ounchecked`) |
|---|---|---|---|
| `assert` / `assertionFailure` | Active | Stripped | Stripped |
| `precondition` / `preconditionFailure` | Active | Active | Stripped |
| `fatalError` | Active | Active | Active |

`-Ounchecked` removes all runtime safety checks, including array bounds. Unless you're writing a real-time audio engine or a game where you've measured the cost of bounds-checking, you're not using it.

## Fixing the example

Back to `findErrorNums`. The problem constraints guarantee exactly one duplicate and one missing number. If those aren't met, the caller violated the contract — so this is a `preconditionFailure`.

Three small things to get right.

**Make the implicit defaults explicit.** Change `var duplicate = 0` to `var duplicate: Int?`. Now the compiler forces us to handle the case where no duplicate was found, instead of silently returning `0`.

**Guard the full invariant, not just part of it.** A first attempt might be:

```swift
guard let missing = unseen.first else {
    preconditionFailure(...)
}
```

But `unseen.first` succeeds as long as the set has *at least* one element. The invariant is that it has *exactly* one. If the input had multiple duplicates, we'd silently return only one of them.

Guard the complete contract:

```swift
guard let duplicate, unseen.count == 1, let missing = unseen.first else {
    preconditionFailure("Expected exactly one duplicate and one missing number")
}
```

**Choose a neutral message.** "Input must contain exactly one duplicate and one missing number" sounds right, but the check runs *after* the algorithm. If the bug is in my loop, the message blames the caller for my mistake. "Expected exactly one duplicate and one missing number" is neutral. It states what was expected without pointing fingers.

Final version:

```swift
func findErrorNums(_ nums: [Int]) -> [Int] {
    var unseen = Set<Int>(1...nums.count)
    var duplicate: Int?
    for num in nums {
        guard let _ = unseen.remove(num) else {
            duplicate = num
            continue
        }
    }
    guard let duplicate, unseen.count == 1, let missing = unseen.first else {
        preconditionFailure("Expected exactly one duplicate and one missing number")
    }
    return [duplicate, missing]
}
```

## When you can change the signature

Sometimes returning `[Int]?` or `throws` is the right answer — let the type system express the failure. But not every function can change its signature: protocol conformances, SDK callbacks, and fixed signatures (LeetCode-style) don't give you that option.

There's also a subtler problem with overusing optionals and `throws`: if every function returns `nil` on unexpected input, the caller has to handle it, and their caller, and so on. The decision *is this an error I can recover from, or should this never happen?* gets deferred indefinitely. The app limps along with `nil` values propagating through layers, producing wrong results far from the original cause.

When you can change the signature and the failure is recoverable, optionals or `throws` are usually best. When you can't, or when the failure is a genuine contract violation, a `precondition` with a clear message is the next best thing.

## Not every force unwrap is the same

`!` and `preconditionFailure` both crash on bad input. The difference is the message:

```
Fatal error: Unexpectedly found nil while unwrapping an Optional value
```

vs

```
Fatal error: Expected exactly one duplicate and one missing number
```

The second takes seconds to debug. The first can take an hour in a large codebase.

But not every `!` hides a contract violation. Some branches are unreachable — no input, valid or invalid, can trigger them. Wrapping those in `guard let` + `preconditionFailure` is misleading: it suggests the branch is reachable when it isn't.

Three categories show up in practice:

**1. Unreachable by adjacent code.** The value is guaranteed non-nil by the lines around it.

```swift
// append guarantees the array is non-empty,
// so .first is always non-nil
windowMaxIndexes.append(n)
maxes.append(nums[windowMaxIndexes.first!])
```

A `guard let` here would read like a warning: "this might be nil." But it can't be — `append` just ran. The `!` is clearer: "I know this has a value."

**2. Unreachable by mathematical constraints.** The value is guaranteed valid by the math.

```swift
// shift = columnNumber % 26, so shift is always 0...25
// UnicodeScalar(65 + 0...25) is always valid ASCII (A...Z)
String(UnicodeScalar(65 + shift)!)
```

No input to the outer function can make `shift` fall outside `0...25`. The modulo is the guarantee.

**3. Reachable with invalid input.** The branch *can* be reached if the caller violates the contract. These need explicit checks.

When the invariant can be checked directly on the parameters, use an upfront `precondition`:

```swift
func maxSlidingWindow(_ nums: [Int], _ k: Int) -> [Int] {
    precondition(k >= 1, "Window size must be at least 1")
    // ...
    // k >= 1 guarantees the loop ran at least once, so .first is non-nil
    maxes.append(nums[windowMaxIndexes.first!])
```

The precondition states the contract upfront, before any work. After it passes, `.first!` becomes Category 1 — guaranteed non-nil because the loop ran.

A tempting but wrong alternative: skip the upfront check, wrap the optional unwrap in a `guard let` + `preconditionFailure`. That conflates "the caller broke the contract" with "I need to unwrap an optional." The compiler's need to unwrap `.first` shouldn't dictate where you state your contracts.

### The grey area: bundle resources and static URLs

A `fatalError` vs `!` debate that's genuinely a matter of taste:

```swift
// Force unwrap — we're responsible for the bundle's contents
let url = Bundle.main.url(forResource: "Config", withExtension: "plist")!

// fatalError — same outcome, but with a diagnostic message
guard let url = Bundle.main.url(forResource: "Config", withExtension: "plist") else {
    fatalError("Config.plist is missing from the bundle")
}
```

The force unwrap relies on you controlling the input. The `fatalError` form gives a better message if something does go wrong. John Sundell has [argued](https://www.swiftbysundell.com/articles/picking-the-right-way-of-failing-in-swift/) that force-unwrapping a compile-time constant is fine — you *know* it's valid, and wrapping it adds noise without meaningful safety. Others find `!` ugly and prefer the explicit message. Both positions are defensible.

The "who's responsible" framework doesn't give a clean answer here. The framework helps you think about the problem, but it doesn't dictate the answer when there's no real contract violation possible.

## Preconditions checked late

Most preconditions belong at the top of the function, before any work. If you can check the invariant directly on the parameters, do it upfront. This is the standard library pattern: `Array.remove(at:)` checks `index < count` before touching the buffer.

But some invariants genuinely can't be checked upfront. "Does the input contain exactly one duplicate and one missing number?" There's no cheap way to verify this without running the algorithm — the algorithm itself is the most efficient detector of the property.

Apple's [swift-numerics](https://github.com/apple/swift-numerics) shows both patterns in a single function:

```swift
static func dftWeight(k: Int, n: Int) -> (r: Self, i: Self) {
    precondition(0 <= k && k < n, "k is out of range")
    guard let N = Self(exactly: n) else {
        preconditionFailure("n cannot be represented exactly.")
    }
    let theta = -2 * .pi * (Self(k) / N)
    return (r: .cos(theta), i: .sin(theta))
}
```

The first check is a classic upfront precondition — you can verify the range immediately. The second can't be checked without attempting the conversion. Both are the caller's responsibility. Both are preconditions. The placement is a pragmatic detail, not a semantic difference.

The question is about **cause**, not **location**: if the failure is caused by the caller's input, it's a precondition, regardless of where in the function you detect it.

## Testing precondition failures

For a long time, `preconditionFailure` branches were untestable — calling them crashed the test suite. Swift Testing's exit tests (Swift 6.2) fix this:

```swift
@Test("Crashes when input has no duplicate")
func noDuplicate() async {
    await #expect(processExitsWith: .failure) {
        _ = findErrorNums([1, 2])
    }
}
```

The macro spawns a child process, runs your code in isolation, and verifies the process terminates as expected. The child process crashes, the parent test passes, the branch gets coverage.

One important caveat: exit tests aren't supported on iOS, tvOS, watchOS, or visionOS as of Swift 6.2. macOS, Linux, FreeBSD, OpenBSD, and Windows only. That's a real limitation for iOS apps — on iOS projects, `preconditionFailure` branches will remain untested for now, and that's something to accept rather than work around.

## Cheat sheet

When code hits a state that shouldn't be possible, ask in order:

1. Can the failure be expressed in the type system? Return `nil` or throw. But don't push optionals outward without resolving them — the decision has to happen somewhere.
2. Did the caller break a documented contract? `precondition` / `preconditionFailure`.
3. Is this a check on your own logic during development? `assert`.
4. Is the situation unrecoverable regardless of who caused it? `fatalError`.

Two rules that matter more than the choice:

- **Never return an out-of-domain value as a fallback.** If `0` isn't a valid answer, don't return it.
- **Always give the check a descriptive message.** The difference between "Unexpectedly found nil" and "Expected exactly one duplicate and one missing number" is the difference between a 30-second fix and a 30-minute investigation.

## References

**Apple documentation:**
- [precondition(_:_:file:line:)](https://developer.apple.com/documentation/swift/precondition(_:_:file:line:))
- [preconditionFailure(_:file:line:)](https://developer.apple.com/documentation/swift/preconditionfailure(_:file:line:))
- [Swift Testing: Exit Testing](https://developer.apple.com/documentation/testing/exit-testing)

**Apple source:**
- [swift-numerics RealModule](https://github.com/apple/swift-numerics/blob/main/Sources/RealModule/README.md) — `precondition` and `preconditionFailure` in the `dftWeight` example
- [StandardLibraryProgrammersManual.md](https://github.com/swiftlang/swift/blob/main/docs/StandardLibraryProgrammersManual.md) — guidelines on `_precondition` vs `_debugPrecondition`

**Swift Evolution:**
- [ST-0008: Exit Tests](https://github.com/swiftlang/swift-evolution/blob/main/proposals/testing/0008-exit-tests.md)

**Other writing:**
- [Picking the right way of failing in Swift — Swift by Sundell](https://www.swiftbysundell.com/articles/picking-the-right-way-of-failing-in-swift/)
- [Using preconditions, assertions, and fatal errors in Swift — Donny Wals](https://www.donnywals.com/using-preconditions-assertions-and-fatal-errors-in-swift/)
- [Understanding assertions — Hacking with Swift](https://www.hackingwithswift.com/plus/intermediate-swift/understanding-assertions)
- [Assert vs Precondition vs FatalError — Vadim Bulavin](https://www.vadimbulavin.com/assert-vs-precondition-vs-fatalerror/)
