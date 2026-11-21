## Shared API

The following types and operations are shared across ad formats. Signatures describe the plugin API, not the underlying SDK.

### SDK Initialization

- `void initialize(PrivacyOptions options, InitializationListener listener)`
  - Applies privacy settings before initializing the SDK.
  - Completes asynchronously through `onInitialized()`.
  - Repeated calls must not restart initialization.
  - Automatic SDK initialization must be disabled when using this explicit initialization flow.
- `InitializationListener`
  - `void onInitialized()`
  - No SDK initialization-failure callback is exposed by the documented API.
  - Plugin validation errors must be reported separately.

### Privacy Options

- `PrivacyOptions`
  - `ConsentStatus userConsent` — consent status supplied by the application.
    - `UNKNOWN` — consent has not been determined.
    - `GRANTED` — consent has been granted.
    - `DENIED` — consent has been denied or withdrawn.
  - `boolean ageRestricted` — enables SDK restrictions for child users.
  - `boolean locationTracking` — enables SDK location tracking, subject to user consent and required Android permissions.
- `ConsentStatus` is a plugin type.
  - `GRANTED` maps to `YandexAds.setUserConsent(true)`.
  - `DENIED` maps to `YandexAds.setUserConsent(false)`.
  - `UNKNOWN` must not be treated as granted; it maps to `false` while remaining distinguishable within the plugin.
- Privacy settings are applied before SDK initialization.
- The application determines the settings, obtains any required consent, and stores the user's choices.
- The plugin does not display consent dialogs or request Android permissions automatically.

### Ad Requests

- `AdRequest`
  - `String adUnitId` — required.
  - `AdTargeting targeting` — optional.
- `AdTargeting`
  - `Integer age` — optional.
  - `Gender gender` — optional; `MALE` or `FEMALE`.
  - `String contextQuery` — optional; the user's actual search query.
  - `List<String> contextTags` — optional; keywords describing the current content.
  - `Location location` — optional; requires appropriate consent and SDK configuration.
- Omitted targeting fields must remain unset.
- The plugin validates values and converts them to SDK types.
- Providing `AdTargeting.location` requires `PrivacyOptions.locationTracking == true`.
  - Otherwise, the plugin rejects the request with a plugin-side validation error.
  - Enabling location tracking does not require providing an explicit location in each request.

### Common Behavior

- SDK calls and Android View operations run on the Android main thread.
- The plugin manages Activity lifecycle integration internally.
- Every ad event identifies its originating ad object.
- Ad errors preserve the available SDK error details.
- Impression data contains advertising and revenue information, not a user profile.

## Sticky Banner API

### Creation

- `BannerHandle createBanner(BannerOptions options, BannerListener listener)`
  - Returns a handle immediately.
  - Creates and configures the Android View asynchronously.
  - Reports completion and calculated dimensions through `onCreated()`.
  - Requires completed SDK initialization.
  - Does not load an advertisement.
  - Creates the banner hidden by default.

- `BannerOptions`
  - `BannerWidth width`
    - `BannerWidth.available()` — available application width.
    - `BannerWidth.fraction(double fraction)` — fraction of available width; `0 < fraction <= 1`.
    - `BannerWidth.dp(int width)` — explicit positive width in dp.
  - `BannerPlacement placement`
    - Supports top or bottom placement.
    - Supports left, center, or right alignment.
  - Available width excludes applicable system insets.

- `BannerSize`
  - `int widthDp`, `int heightDp`
  - `int widthPx`, `int heightPx`
  - Height is calculated by the SDK before ad loading.
  - Dimensions remain stable while the relevant window configuration and requested width remain unchanged.

### Operations

- `void load(AdRequest request)`
  - Available after `onCreated()`.
  - Starts asynchronous ad loading.
  - A loaded advertisement renders automatically when the banner is visible.
  - Subsequent sticky-banner refreshes are managed by the SDK.

- `void setVisible(boolean visible)`
  - Shows or hides the entire banner View without destroying the handle.
  - Does not modify or cover individual advertising elements.
  - Does not promise to suspend SDK network activity.

- `void destroy()`
  - Removes the View and releases SDK resources.
  - Is idempotent.
  - Suppresses subsequent listener notifications.
  - Invalidates the handle for further operations.

### Events

- `BannerListener`
  - `void onCreated(BannerHandle banner, BannerSize size)`
    - The banner is ready to accept a load request.
  - `void onCreationFailed(BannerHandle banner, PluginError error)`
    - Plugin-side creation failed.
  - `void onLoaded(BannerHandle banner)`
    - An advertisement loaded successfully.
    - May occur repeatedly during the handle's lifetime.
  - `void onLoadFailed(BannerHandle banner, AdError error)`
    - Ad loading failed.
    - Does not automatically hide or destroy the banner.
  - `void onClicked(BannerHandle banner)`
    - The SDK registered an ad click.
  - `void onImpression(BannerHandle banner, ImpressionData data)`
    - The SDK registered an impression.
    - `data` may be absent; when available, it exposes the original ILRD JSON.

### Layout Responsibility

- The application reserves space using the reported dimensions.
- Visibility changes must be coordinated with the application's layout.
- For the initial implementation, changes to window configuration or requested width require destroying and recreating the banner.
