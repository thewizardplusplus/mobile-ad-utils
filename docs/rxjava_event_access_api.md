# RxJava Event Access API

This specification defines RxJava 3 access to the plugin events previously defined for SDK initialization, sticky banners, interstitial ads, and rewarded ads. Signatures describe the plugin API, not the SDK.

## Scope

- One shared event source exists per `AdsPlugin` instance.
- The plugin exposes all events; each ad handle exposes a filtered view of the same source.
- This document defines the reactive alternative to the listener-based event delivery in [sticky_banner_api.md](sticky_banner_api.md) and [fullscreen_ads_api.md](fullscreen_ads_api.md).
  - Existing operation and event meanings remain unchanged.
  - The reactive API does not require application listeners during initialization or ad creation.
  - SDK listeners remain an internal implementation detail.

## Configuration

- `PluginOptions`
  - `Scheduler eventScheduler` — optional default scheduler for event delivery.
  - If omitted or `null`, use `AndroidSchedulers.mainThread()` from RxAndroid 3.
- `AdsPlugin(PluginOptions options)`
  - Creates the plugin and its event source; does not initialize the SDK.
  - Resolves the default event scheduler once for this plugin instance.
- Reactive operation signatures:
  - `void initialize(PrivacyOptions options)`
  - `BannerHandle createBanner(BannerOptions options)`
  - `FullscreenHandle createInterstitial()`
  - `FullscreenHandle createRewarded()`
  - Other operation signatures and preconditions remain as defined in the format specifications.

## Event Model

- `AdEvent` — immutable base type.
  - `String eventId` — unique event identifier, preserved during replay.
  - `String adInstanceId` — originating wrapper handle ID; `null` for SDK-wide events.
  - `long timestamp` — event creation time in milliseconds since the Unix epoch; not an ordering key.
- Handle IDs identify wrapper instances, not ad unit IDs.
  - Each handle receives its ID before it can publish events.
  - IDs are never reused, including after handle destruction.
- Payloads are immutable plugin value objects or copied snapshots.
  - Events retain no `Activity`, Android View, ad handle, or SDK advertising object.
  - SDK error details and available impression JSON are preserved.

### Event Types

- `SdkInitialized`
  - Maps to `onInitialized()`; SDK initialization completed.
  - No SDK initialization-failure event is invented.
- `BannerCreated`
  - Maps to banner `onCreated()`.
  - `BannerSize size` — calculated banner dimensions; the handle can accept a load request.
- `BannerCreationFailed`
  - Maps to banner `onCreationFailed()`.
  - `PluginError error` — plugin-side creation failure.
- `AdLoaded`
  - Maps to `onLoaded()` for all three formats.
  - Banner refreshes may produce multiple events over one handle's lifetime.
- `AdLoadFailed`
  - Maps to `onLoadFailed()` for all three formats.
  - `AdError error` — load failure details.
- `AdShown`
  - Maps to fullscreen `onShown()`; the advertising screen was displayed.
  - Does not mean video completion or reward eligibility.
- `AdShowFailed`
  - Maps to fullscreen `onShowFailed()`.
  - `AdError error` — show failure details.
- `AdDismissed`
  - Maps to fullscreen `onDismissed()`; the advertising screen closed.
  - Does not distinguish early closure from closure after full viewing.
- `AdClicked`
  - Maps to `onClicked()` for all three formats.
- `AdImpression`
  - Maps to `onImpression()` for all three formats.
  - `ImpressionData data` — optional; preserves available original ILRD JSON.
  - Represents an SDK-registered impression, not exact display start, viewing completion, or a user profile.
- `RewardEarned`
  - Maps to rewarded `onRewarded()`.
  - `Reward reward` — immutable copied values: `int amount`, `String type`.
  - Confirms reward eligibility; the application grants and persists the reward.
  - Published at most once per rewarded handle; replay may deliver that same event again.

## Event Access

The plugin and every `BannerHandle` and `FullscreenHandle` expose:

- `Flowable<AdEvent> events()`
  - Delivers events using the plugin's resolved default scheduler.
- `Flowable<AdEvent> events(Scheduler scheduler)`
  - Requires a non-null scheduler.
  - Uses this scheduler instead of the plugin default for the returned flow.
  - Does not change defaults or other subscriptions.

### Scope and Filtering

- `plugin.events()` includes SDK-wide events and events from every ad handle.
- `handle.events()` filters the shared source by that handle's `adInstanceId`.
  - SDK-wide events are excluded.
  - No separate replay history is created for the handle.
- Consumers may store a returned `Flowable`, branch it, and apply `ofType()` and `filter()`.
  - Each `subscribe()` creates an independent subscription.
  - Branches do not consume events on behalf of each other.
  - Common filters are evaluated per subscription; results are not automatically cached.
  - Creating a flow does not start consuming events.
- Consumers receive read-only `Flowable` views, not the writable processor.

## Scheduling

- Scheduler selection order:
  1. Explicit scheduler passed to `events(scheduler)`.
  2. `PluginOptions.eventScheduler`.
  3. `AndroidSchedulers.mainThread()`.
- Apply the handle filter to the raw source, then apply one `observeOn(selectedScheduler)`.
  - The explicit overload must not first switch through the default scheduler.
  - Downstream consumer filters and handlers run through the selected scheduler unless the consumer adds another scheduling operator.
- Scheduler configuration controls event delivery only.
  - SDK calls, Android View operations, and internal lifecycle management retain their required Android threading rules.
  - The Android main thread is not necessarily the game thread.
  - Engine integration supplies a scheduler backed by its task-dispatch mechanism, for example through `Schedulers.from(gameExecutor)`.
- The application owns custom schedulers and executors; the plugin never shuts them down.
- Asynchronous isolation depends on the selected scheduler; a caller-supplied synchronous scheduler does not guarantee it.
- Delivery is sequential and ordered within each subscription; no execution order is guaranteed between different subscriptions.
- `observeOn` may buffer events per subscription; a shared history does not imply zero additional queues.

## SDK Integration

- Internal SDK callbacks first update wrapper state and perform required resource cleanup, then publish immutable events.
- Use a serialized processor to prevent concurrent or reentrant publication from violating the event protocol:
  - `ReplayProcessor.<AdEvent>create().toSerialized()`.
- Initialization success and ad notifications do not require a return value or synchronous response from the application.
- Load, show, and banner-creation failures are event values, not terminal `onError()` signals.
  - One ad failure must not terminate the shared stream.
  - Validation errors retain their separately defined reporting behavior.
- Consumer processing must not control internal SDK cleanup or lifecycle correctness.

## History and Subscription Lifetime

- Initial history policy: retain all published events in memory for the plugin instance's lifetime using `ReplayProcessor.create()`.
  - New subscriptions receive retained matching events, followed by future events.
  - Unsubscribing does not erase history.
  - No durable storage or recovery after process death is provided.
- No manual deletion, clear-history, or delete-after-all-read API is exposed.
  - Size/time retention would be a separate policy change, not an implied feature of this contract.
- The application retains the `Disposable` returned by subscription and calls `dispose()` when that subscription is no longer needed.
- Ad handle destruction suppresses new event publication for that handle.
  - It does not retract already published events or remove replay history.
  - Events already scheduled may still arrive unless the subscription is disposed.
- Reward consumers must avoid duplicate grants when resubscribing, for example by deduplicating on `eventId`.
- Consumers provide an Rx error handler for processing or stream failures; these differ from ordinary `AdLoadFailed` and `AdShowFailed` values.

## Usage Example

```java
PluginOptions options = new PluginOptions();
options.eventScheduler = gameScheduler;
AdsPlugin plugin = new AdsPlugin(options);

// After SDK initialization, create a rewarded handle.
FullscreenHandle rewarded = plugin.createRewarded();

Flowable<AdEvent> adEvents = rewarded.events();

Disposable rewards = adEvents
    .ofType(RewardEarned.class)
    .subscribe(
        event -> grantRewardOnce(event.eventId, event.reward),
        error -> logError(error)
    );

Disposable impressions = plugin.events(dbScheduler)
    .ofType(AdImpression.class)
    .subscribe(
        event -> saveImpressionOnce(event.eventId, event),
        error -> logError(error)
    );

// When these consumers are no longer needed:
rewards.dispose();
impressions.dispose();
```

`gameScheduler`, `dbScheduler`, reward granting, deduplication, database access, and error logging belong to the application or engine integration.
