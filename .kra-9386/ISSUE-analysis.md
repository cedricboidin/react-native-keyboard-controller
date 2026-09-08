---
name: Bug report
about: Create a report to help us improve
title: ""
labels: bug
assignees: kirillzyusko
---

**Describe the bug**

On iOS 26, when a `UIScrollView` with `keyboardDismissMode="on-drag"` dismisses the keyboard, `KeyboardAvoidingView` intermittently fails to follow it down — roughly one dismissal in two.

The cause is that **the tracked layer is not always animated**. On iOS 26 `KeyboardTrackingView.view` returns the library's own view, pinned with `bottomAnchor == topView.keyboardLayoutGuide.topAnchor`. On a scroll-driven dismissal that constraint is sometimes applied instantly rather than animated, so `layer.presentation()` carries no animation and reports its **final** position on the very first display-link tick. `updateKeyboardFrame` samples that position, and sampling cannot describe travel that has already happened.

Native trace, five dismissals (`animationKeys` and `frameY` from `frameTransitionInWindow`, `windowH` = 874):

| dismissal | `animationKeys` | `frameY`, first then second tick | outcome |
|---|---|---|---|
| #1 | `[]` | 539 → **874** | broken |
| #3 | `[]` | 539 → **874** | broken |
| #5 | `[position]` | 539 → 540.25 → 612 → 660 → … | correct |

Across 271 ticks: 177 with `animationKeys = []`, 94 with `[position]`. `crossFade` was `false` and `opacity` `1.0` throughout, so the opacity branch of `frameTransitionInWindow` is not involved.

The single root cause surfaces as two different symptoms, depending only on what `animation` happened to hold:

**`animation == nil`** — the show completed, so `keyboardDidAppear` cleared it. The watcher emits exactly one `onKeyboardMove` with `progress = 0` (~10 ms in), then `keyboardPosition == prevKeyboardPosition` on every later tick. `KeyboardAvoidingView` drops its whole `paddingBottom` in one frame while the keyboard still has ~380 ms of travel left.

**`animation` = the show's object, `isFinished == true`** — the dismissal cancelled the show's `did` task (`scheduleDidEvent` begins with `keyboardDidTask?.cancel()`), so `animation` survived with `lastValue == toValue`. `keyboardWillDisappear` calls `initializeAnimation(fromValue:toValue: 0)`, but that only assigns when it finds a CoreAnimation animation — here it finds none and silently keeps the stale object. Every tick then returns at `KeyboardMovementObserver+Watcher.swift:43` on `isFinished`, so **no** movement event is emitted: `progress` stays at 1 for the whole descent, then `onEnd` collapses it in a single un-animated frame.

One observation that is a co-symptom rather than a cause: iOS posts `keyboardWillHide` **twice** on the failing dismissals and once on the healthy ones. But `animationKeys` is already empty at the *first* of the two, so the duplicate is not what breaks it — and the iOS 18.3 run below shows the same duplicate on dismissals that are all correct. Its only effect here is to arm a second, degenerate transition.

**Not reproducible on iOS 18.3.** Same commit, same unpatched code, same gesture, on an iPhone 16 / iOS 18.3 simulator: six dismissals out of six correct, `move` 30-31 each, progress walking 0.98 -> 0.000. Under iOS 26 the failure rate is roughly one dismissal in two, so six clean runs would be a ~1.6% coincidence.

```
[ok] starts=2 move=30 interactive=0  p 0.984->0.000 p0@328ms end@1158ms iosDur=250
[ok] starts=2 move=30 interactive=0  p 0.983->0.000 p0@328ms end@1158ms iosDur=250
[ok] starts=2 move=30 interactive=0  p 0.982->0.000 p0@328ms end@1161ms iosDur=250
[ok] starts=2 move=30 interactive=0  p 0.981->0.000 p0@329ms end@1159ms iosDur=250
[ok] starts=1 move=31 interactive=0  p 0.948->0.000 p0@319ms end@ 608ms iosDur=250
[ok] starts=2 move=30 interactive=0  p 0.991->0.000 p0@324ms end@1156ms iosDur=250
```

That points at the iOS 26 tracking path specifically: below iOS 26 the observer samples the real keyboard view found by `KeyboardViewLocator`, and only from iOS 26 does it sample its own view pinned to `keyboardLayoutGuide`. Two things it does **not** settle, and I would not want to overstate: whether the guide-pinned view is simply attached to the wrong view controller (`attachToTopmostView` uses `UIApplication.topViewController()?.view`), or whether iOS 26 genuinely applies the guide constraint without animating it on a scroll-driven dismissal. The device also differs (iPhone 16 vs 17) and so does the reported duration (250ms vs 383ms).

**Code snippet**

```tsx
<KeyboardAvoidingView behavior="padding" style={{ flex: 1 }}>
  <FlatList
    inverted
    data={rows}
    keyboardDismissMode="on-drag"
    keyboardShouldPersistTaps="handled"
    renderItem={renderRow}
  />
  {/* this footer must follow the keyboard down */}
  <View style={{ padding: 12, backgroundColor: "#ffd60a" }}>
    <TextInput />
  </View>
</KeyboardAvoidingView>
```

**Repo for reproducing**

<!-- TODO: fork URL + branch, e.g.
https://github.com/<org>/react-native-keyboard-controller/tree/repro/kav-stale-animation
Reproduction lives at example/src/screens/Examples/StaleAnimationRepro/.
-->

**To Reproduce**

1. Check out the reproduction branch and run `yarn bootstrap`
2. Run the example app on an iOS 26 simulator: `yarn example ios`
3. Open **"KAV stale animation repro"** from the examples list
4. Tap the input to raise the keyboard, then flick the list to dismiss it
5. Watch the yellow footer — it must travel down together with the keyboard
6. Tag the newest row **looked ok** or **looked wrong**, and repeat 4-6 a few times

The screen annotates each dismissal with the `onStart` / `onMove` / `onInteractive` counts and the progress trajectory, so the two broken shapes are identifiable without a debugger: `move=1` with progress reaching 0 within ~10 ms is the instant collapse, `move=0` is the frozen one. Dismissing by **tapping** instead of flicking is always correct — that path gets `animationKeys = [position]` — which makes a useful control.

Note on step 2: with Ruby 3.4, `yarn example ios` can fail before reaching Xcode, because the CLI runs `bundle install` and the native `ffi` gem does not build. `yarn bootstrap` has already installed the pods, so building directly works around it:

```sh
cd example/ios
xcodebuild -workspace KeyboardControllerExample.xcworkspace \
  -scheme KeyboardControllerExample -configuration Debug \
  -sdk iphonesimulator -destination 'id=<udid>' \
  -derivedDataPath /tmp/kc-build build
xcrun simctl install <udid> \
  /tmp/kc-build/Build/Products/Debug-iphonesimulator/KeyboardControllerExample.app
xcrun simctl launch <udid> com.example.reactnativekeyboardcontroller
```

**Expected behavior**

`onKeyboardMove` should describe the whole hide, so `progress` animates from 1 to 0 in step with the keyboard and `KeyboardAvoidingView` releases its padding as the keyboard travels.

**Screenshots**

Not provided as a still — the failure is a missing or mistimed animation. The tables above are the on-screen and native-log output of the reproduction.

**Smartphone (please complete the following information):**

- Desktop OS: macOS 26.6.2 (25G83)
- Device: iPhone 17 simulator
- OS: iOS 26.5
- RN version: 0.81.4 (the example app). Originally hit on 0.83.6.
- RN architecture: new / fabric
- JS engine: Hermes
- Library version: `main` at `2f90cc4`. Originally hit on 1.22.4; `KeyboardMovementObserver+Watcher.swift` is byte-identical between 1.20.7 and `main`.

**Additional context**

A candidate fix is attached as a patch, and we are running it locally. Since sampling is useless when the layer is inert, it falls back to driving the progression from the duration iOS announced — the strategy a notification-only keyboard handler uses, which is why RN's own `KeyboardAvoidingView` never shows this:

- New `transitionTarget` / `transitionFrom` / `transitionStart` / `transitionDuration` on the observer, set by `beginTransition(to:duration:)` from both `will` handlers and released by both `did` handlers.
- In `updateKeyboardFrame`, ahead of the existing guards: when `presentation()?.animationKeys()` is empty and a transition is in flight with actual travel, interpolate from elapsed time and emit that instead of the sampled position.
- The interpolation uses easeOutQuart. That is not arbitrary: fitted against the sampled trajectory of a healthy hide it gives rms 0.038 over the 383 ms travel, where easeInOutCubic gives 0.51 at the midpoint against a real 0.50 — the system curve is a spring that covers half its distance in the first ~19% of the reported duration, so an ease-in start lags very visibly.
- `transitionTarget` is released as soon as `elapsed >= 1`, otherwise the branch keeps re-emitting the final value on every tick — indefinitely whenever the `did` event has been cancelled. Worth flagging: this was a real defect in an earlier revision of the patch, invisible on screen and only caught in the native log.
- Two smaller corrections on the sampled path: `initializeAnimation` now clears `animation` when it finds nothing, instead of leaving the previous transition's object in place; and the `if animation == nil` fallback interpolates toward `transitionTarget ?? keyboardHeight` rather than always `keyboardHeight`, which pointed at the **open** height even while closing and pinned progress near 1 through the `race = max` branch.

Three caveats I would rather state than hide:

- easeOutQuart is a stand-in for the real spring. The residual is confined to the first ~40 ms, where the spring ramps up from zero velocity while the curve starts immediately, so the view very slightly *leads* the keyboard at the start. Reading the real curve out of the notification would be better, but `UIKeyboardAnimationCurveUserInfoKey` reports a private value for the keyboard and its control points are not public.
- I have not found why iOS applies the `keyboardLayoutGuide` constraint without animating it on these dismissals, nor why it posts `keyboardWillHide` twice on exactly those. Both may have a common cause upstream of the observer that would be better addressed directly.
- The fix is applied unconditionally, but the failure only showed up on iOS 26 (see the iOS 18.3 run above). Gating the fallback on `usesKeyboardLayoutGuideTracking` would be more conservative, and may well be the right call — I left it ungated because a layer carrying no animation cannot be sampled on any OS, so the fallback should be inert where it never triggers.

Happy to turn this into a PR if the direction looks right.
