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
