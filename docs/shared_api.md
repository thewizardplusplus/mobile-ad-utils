## Shared API

The following types and operations are shared across ad formats. Signatures describe the plugin API, not the underlying SDK.

See [Java coding conventions](coding_conventions.md) for identifier types, optional values, and timestamps.

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
  - `Optional<AdTargeting> targeting` — optional.
- `AdTargeting`
  - `Optional<Integer> age` — optional.
  - `Optional<Gender> gender` — optional; `MALE` or `FEMALE`.
  - `Optional<String> contextQuery` — optional; the user's actual search query.
  - `Optional<List<String>> contextTags` — optional; keywords describing the current content.
  - `Optional<Location> location` — optional; requires appropriate consent and SDK configuration.
- For each empty targeting field, leave the corresponding SDK setting unset.
- The plugin validates values and converts them to SDK types.
- Providing a location in `AdTargeting.location` requires `PrivacyOptions.locationTracking == true`.
  - Otherwise, the plugin rejects the request with a plugin-side validation error.
  - Enabling location tracking does not require providing an explicit location in each request.

### Common Behavior

- SDK calls and Android View operations run on the Android main thread.
- The plugin manages Activity lifecycle integration internally.
  - Showing requires an active, resumed Activity.
  - If showing is currently impossible, report a show failure instead of deferring the ad until an unexpected moment.
- Every ad event identifies its originating handle.
- Errors preserve available SDK details and distinguish SDK failures from plugin errors.
- Impression data contains advertising and revenue information, not a user profile.
- Event callbacks are serialized on the plugin's documented callback thread.
- The plugin does not automatically retry failed requests or show loaded ads.
