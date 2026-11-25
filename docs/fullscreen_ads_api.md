## Fullscreen Ads API

### Creation

- `InterstitialHandle createInterstitial(InterstitialListener listener)`
- `RewardedHandle createRewarded(RewardedListener listener)`
  - Return a handle immediately.
  - Require completed SDK initialization.
  - Do not load or show an advertisement.
  - Do not create an application-managed advertising View.
  - No separate creation event is required; SDK loader setup is handled internally when loading.

### Operations

Both handle types expose:

- `void load(AdRequest request)`
  - Starts asynchronous loading.
  - Allows one pending request per handle.
  - Reports completion through `onLoaded()` or `onLoadFailed()`.
  - Loading may be retried after failure.
  - Rejects calls while loading, while an ad is ready, or after a show attempt.
- `void show()`
  - Requires a successfully loaded advertisement.
  - Uses the current Activity internally.
  - Reports display through `onShown()` or failure through `onShowFailed()`.
  - Does not block until the advertisement closes.
  - Consumes the loaded advertisement; another show attempt requires a new handle.
- `void destroy()`
  - Releases retained resources and listeners.
  - Is idempotent.
  - Invalidates the handle and suppresses subsequent notifications.
  - During loading, discards late results; does not promise cancellation of the underlying network request.
  - Is not allowed while a show attempt is active; wait for `onDismissed()` or `onShowFailed()`.
  - Does not provide a way to forcibly close displayed advertising.

### Interstitial Events

- `InterstitialListener`
  - `void onLoaded(InterstitialHandle ad)`
    - The advertisement is ready for a show attempt.
  - `void onLoadFailed(InterstitialHandle ad, AdError error)`
    - Loading failed; no advertisement is ready.
  - `void onShown(InterstitialHandle ad)`
    - The advertising screen was displayed.
    - Does not indicate completion of a video.
  - `void onShowFailed(InterstitialHandle ad, AdError error)`
    - The show attempt failed.
    - The plugin releases the loaded advertisement.
  - `void onDismissed(InterstitialHandle ad)`
    - The advertising screen closed.
    - Does not distinguish early closure from closure after full viewing.
    - The plugin releases the loaded advertisement.
  - `void onClicked(InterstitialHandle ad)`
    - The SDK registered an ad click.
  - `void onImpression(InterstitialHandle ad, ImpressionData data)`
    - The SDK registered an impression.
    - `data` may be absent; otherwise, exposes the original ILRD JSON.
    - Does not indicate completion of viewing.

### Reward

- `Reward`
  - `int amount` — reward quantity.
  - `String type` — reward type or currency name.
- An immutable plugin value object.
- The plugin copies these values from the SDK's `Reward` object.
- For advertising, reward values are configured in the rewarded ad unit settings in the publisher interface.
- Receiving `onRewarded()` confirms reward eligibility; the application grants and persists the reward.

### Rewarded Events

- `RewardedListener`
  - `void onLoaded(RewardedHandle ad)`
    - The advertisement is ready for a show attempt.
  - `void onLoadFailed(RewardedHandle ad, AdError error)`
    - Loading failed; no advertisement is ready.
  - `void onShown(RewardedHandle ad)`
    - The advertising screen was displayed.
    - Does not indicate reward eligibility.
  - `void onShowFailed(RewardedHandle ad, AdError error)`
    - The show attempt failed.
    - The plugin releases the loaded advertisement.
  - `void onDismissed(RewardedHandle ad)`
    - The advertising screen closed.
    - Does not indicate whether a reward was earned.
    - The plugin releases the loaded advertisement.
  - `void onClicked(RewardedHandle ad)`
    - The SDK registered an ad click.
  - `void onImpression(RewardedHandle ad, ImpressionData data)`
    - The SDK registered an impression.
    - `data` may be absent; otherwise, exposes the original ILRD JSON.
    - Does not indicate reward eligibility.
  - `void onRewarded(RewardedHandle ad, Reward reward)`
    - The SDK confirmed reward eligibility.
    - The application grants the reward.
    - The plugin forwards this event at most once per handle.
    - Reward eligibility must not be inferred from dismissal or impression events.

### Application Responsibility

- Interstitial:
  - Select an appropriate natural break for showing the ad.
- Rewarded:
  - Obtain the user's explicit choice to view advertising for a stated reward.
  - Grant that reward upon `onRewarded()`.
- Both:
  - Manage game pause, audio, and any application-owned waiting UI.
  - Continue the game after dismissal or failure only when the application itself is active.
  - Create a new handle for the next advertisement.

### Design Notes

- Each handle represents a single show attempt, making it easier to associate events and rewards with that attempt.
- The underlying SDK loader may be reused internally.
- Disallowing `destroy()` during an active show attempt is a wrapper design decision intended to prevent accidental loss of reward notifications.
