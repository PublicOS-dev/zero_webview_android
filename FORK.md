# zero_webview_android — vendored fork of `webview_flutter_android`

**Pin:** `webview_flutter_android` **4.10.1** (pub.dev), copied from the local
pub cache on 2026-09-05. The pristine copy stays in
`%LOCALAPPDATA%\Pub\Cache\hosted\pub.dev\webview_flutter_android-4.10.1\`;
`diff -r` against it is the authoritative fork delta.

**Package name: UNCHANGED.** `webview_flutter` resolves its platform
implementation by package name (`implements: webview_flutter` +
`default_package`), and a `dependency_overrides:` entry must name the package
it overrides. So the repo folder follows our `zero_webview_*` precedent while
`pubspec.yaml` stays `name: webview_flutter_android`, with
`publish_to: 'none'`. An app consumes it through a `dependency_overrides:`
path or git entry.

## Why it exists

Engine storage (cookies, localStorage, IndexedDB, cache) is process-wide on
Android unless a WebView is bound to a **Profile**
(`androidx.webkit.WebViewCompat.setProfile`, androidx.webkit ≥ 1.10, WebView
≥ 121). The upstream plugin never calls it, and the `WebView` instance is not
reachable from outside the plugin. Checked 2026-09-05: pub.dev latest (4.14.1)
still has no profile binding. PublicOS needs one jar per account key.

## Our changes on top of the pin

1. **`publish_to: 'none'`** and **`pigeon: 25.3.2` pinned exactly** (was
   `^25.3.2`), so `dart run pigeon --input pigeons/android_webkit.dart`
   reproduces upstream's generated code plus ours. Upstream post-formats the
   generated files (`dart format` / `ktfmt`); ours are `dart format`ed but
   not `ktfmt`ed, so a line diff of `AndroidWebkitLibrary.g.kt` is noisy and a
   **token diff** is the honest comparison (2026-09-05: tokens differ only by
   the `setProfile` addition).
2. **`WebView.setProfile(String profileName)`** — new ProxyApi method
   (`pigeons/android_webkit.dart`), regenerated into `android_webkit.g.dart`
   and `AndroidWebkitLibrary.g.kt`; implemented in `WebViewProxyApi.java` as
   `WebViewCompat.setProfile(instance, name)`. Creates the profile if it does
   not exist. Native exceptions (already navigated, already set, no
   `MULTI_PROFILE`) surface to Dart as `PlatformException` — loud on purpose.
3. **`AndroidWebViewControllerCreationParams.profileName`** — bound in the
   controller's constructor as the **first** message after the WebView is
   constructed, before any settings call. Pigeon messages are delivered in
   send order, which is what makes "first" true.
4. **`WebViewFeatureConstants.multiProfile`**, **`WebViewFeatureType.multiProfile`**
   and the static **`AndroidWebViewController.isMultiProfileSupported()`** —
   the pre-flight the app runs before deciding to construct with a profile.

## Pin review

Upstream may add its own profile API one day. If it does, migrate the app to
theirs and retire this fork rather than reconciling ours toward it.
