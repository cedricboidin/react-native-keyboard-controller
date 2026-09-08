**Describe the bug**

On iOS 26, `KeyboardAvoidingView` with `behavior="padding"` does not follow the keyboard down when the keyboard is dismissed by dragging a scroll view (`keyboardDismissMode="on-drag"`). It fails roughly one dismissal in two, in one of two shapes: the padding either collapses instantly in a single frame while the keyboard is still travelling, or it stays fully applied for the whole descent and then snaps down when the keyboard is already gone.

Dismissing by tapping outside the input is always correct, and the same code on iOS 18.3 is always correct too.

The cause looks like this. From iOS 26 the observer no longer samples the real keyboard view — `KeyboardTrackingView.view` returns the library's own view, pinned with `bottomAnchor == topView.keyboardLayoutGuide.topAnchor`. On a drag dismissal that constraint is applied without an animation: `layer.presentation()?.animationKeys()` is empty and the presentation frame reports its final position on the very first display-link tick (`frameY` jumps 539 → 874 = window height between two ticks, where a healthy dismissal walks 539 → 540 → 612 → 660 → …). `updateKeyboardFrame` therefore has nothing meaningful to sample, and depending on whether `animation` is `nil` or a finished leftover from the show, it emits either a single `onKeyboardMove` at `progress = 0` or none at all.

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

<!-- fork URL + branch; the screen lives at example/src/screens/Examples/StaleAnimationRepro/ -->

**To Reproduce**

Steps to reproduce the behavior:

1. Check out the reproduction branch, run `yarn bootstrap`, then `yarn example ios` on an iOS 26 simulator
2. Open "KAV stale animation repro" from the examples list
3. Tap the input to raise the keyboard, then flick the list to dismiss it
4. Watch the yellow footer — it should travel down together with the keyboard
5. Repeat a few times: about half the dismissals are wrong, and the screen prints the event counts for each one

Tapping outside the input instead of flicking the list is a useful control — that path never fails.

**Expected behavior**

`onKeyboardMove` should describe the whole hide, so `progress` animates from 1 to 0 in step with the keyboard and `KeyboardAvoidingView` releases its padding as the keyboard travels — as it already does on a tap dismissal, and as it does on iOS 18.3.

**Screenshots**

Video attached, showing the three cases in order:

1. dismissing without scrolling — the footer follows the keyboard correctly
2. the footer drops instantly, with no animation, while the keyboard is still travelling
3. the footer stays up for the whole descent and only drops once the keyboard has gone

**Smartphone (please complete the following information):**

- Desktop OS: macOS 26.6.2
- Device: iPhone 17 simulator (not reproducible on an iPhone 16 / iOS 18.3 simulator)
- OS: iOS 26.5
- RN version: 0.81.4 (the example app); originally hit on 0.83.6
- RN architecture: new / fabric
- JS engine: Hermes
- Library version: `main` at `2f90cc4`; originally hit on 1.22.4

**Additional context**

`KeyboardMovementObserver+Watcher.swift` is byte-identical between 1.20.7 and `main`, so this is not a recent regression — on 1.20.7 it was worse, because the `animation` `didSet` then scheduled `keyboardDidAppear` instead of `keyboardDidDisappear` and the padding stayed applied indefinitely.

I have a candidate patch I am running locally, and a native `os_log` trace of the watcher across both the failing and healthy dismissals. Happy to attach either, or to open a PR, if useful. The short version of the patch: when the tracked layer carries no CoreAnimation animation, stop sampling it and drive the progression from the duration iOS announced instead.

Two things I could not settle: whether the guide-pinned view is simply attached to the wrong view controller (`attachToTopmostView` uses `UIApplication.topViewController()?.view`), or whether iOS 26 genuinely applies the guide constraint without animating it on a scroll-driven dismissal.
