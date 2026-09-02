# Noibu Android SDK Guide

[Android](https://central.sonatype.com/artifact/com.noibu.mobile.android/session-replay) · [Jetpack Compose](https://central.sonatype.com/artifact/com.noibu.mobile.android/session-replay-compose)

## Table of Contents

- [1. Requirements](#1-requirements)
- [2. Installation](#2-installation)
- [3. Configuration](#3-configuration)
- [4. Initialization](#4-initialization)
- [5. Page Navigation](#5-page-navigation)
- [6. Error Reporting](#6-error-reporting)
- [7. Network Monitoring](#7-network-monitoring)
- [8. WebView Support](#8-webview-support)
- [9. View Tagging](#9-view-tagging)
- [10. Custom Attributes](#10-custom-attributes)
- [11. Privacy & Security](#11-privacy--security)
- [12. Lifecycle Management](#12-lifecycle-management)

---

## 1. Requirements

- **Minimum Android SDK**: API 26 (Android 8.0)
- **Compile SDK**: 36
- **Kotlin**: 2.0+
- **Gradle**: 8.0+

---

## 2. Installation

Add the Noibu Session Replay SDK to your app module's `build.gradle.kts`:

```kotlin
dependencies {
    implementation("com.noibu.mobile.android:session-replay:1.0.1")
}
```

Jetpack Compose apps add the Compose add-on as well. It is published from this SDK's release tag
and always carries the same version:

```kotlin
dependencies {
    implementation("com.noibu.mobile.android:session-replay:1.0.1")
    implementation("com.noibu.mobile.android:session-replay-compose:1.0.1")
}
```

Fragment/XML-only apps depend on `session-replay` alone and never pull Compose.

Sync Gradle to download the artifacts. See the [changelog](CHANGELOG.md) for what changed in this
release.

---

## 3. Configuration

### SessionReplayConfig

The SDK is configured via `SessionReplayConfig`. All parameters:

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `domain` | `String` | **Yes** | - | Your Noibu domain endpoint (provided by Noibu) |
| `privacyMode` | `PrivacyMode` | No | `MASK_SENSITIVE` | How much displayed text reaches the replay (see [Privacy](#privacy)). |
| `logLevel` | `LogLevel?` | No | `null` | Diagnostic log verbosity. Leave unset for the default (silent in release builds). See [Diagnostic Logging](#diagnostic-logging). |
| `trackTouches` | `Boolean` | No | `true` | Capture clicks and scrolls (with DXA selectors). |
| `trackKeyboard` | `Boolean` | No | `true` | Capture keyboard-focus events — which field the user typed in, never the typed text. |
| `trackNetwork` | `Boolean` | No | `true` | Capture HTTP events. Off drops them SDK-wide even with the interceptor installed — a kill switch that needs no OkHttp change. |
| `trackWebViews` | `Boolean` | No | `true` | Allow webview capture. Off makes `WebViewTracking.enable()` a no-op SDK-wide. |
| `trackErrors` | `Boolean` | No | `true` | Capture uncaught exceptions (always chains to an existing handler). |
| `autoTrackNavigation` | `Boolean` | No | `false` | Open a page per resumed Activity/Fragment, with no per-screen calls. Don't combine with explicit navigation wiring — explicit wiring wins and silences the automatic boundaries. |

### Privacy

The mode controls how much **displayed** text (labels, button titles, field hints) reaches the replay.
What the user **types** is never captured in any mode — a field surfaces its hint, or `***`.

| Mode | Description |
|------|-------------|
| `PrivacyMode.ALLOW_ALL` | Displayed text is captured verbatim — nothing is redacted |
| `PrivacyMode.MASK_SENSITIVE` | Default. Displayed text is captured, with card / SSN / email / phone spans redacted in place — including PII rendered as a plain label, which input masking never sees |
| `PrivacyMode.MASK_ALL` | No readable text at all: every string becomes `xxxx`, keeping only its word/length shape |

The mode applies to both the View and Compose walkers, and is shared with the iOS and Flutter SDKs
(`com.noibu.mobile.core.privacy.TextMasking` in the KMP core owns the policy).

### Diagnostic Logging

The SDK can print diagnostic logs to Logcat (tag `NoibuMobile`) to help debug an integration. Control verbosity with `logLevel`:

| Level | What it logs |
|-------|--------------|
| `LogLevel.NONE` | Nothing |
| `LogLevel.ERROR` | Failures (send / connect / persist) |
| `LogLevel.WARNING` | Recoverable issues (discarded data, missing assets) plus errors |
| `LogLevel.INFO` | **Recommended for debugging** — lifecycle milestones (init, page recording, data sends, background sync) plus warnings and errors |
| `LogLevel.DEBUG` / `LogLevel.VERBOSE` | Internal Noibu diagnostics; high volume, intended for Noibu support |

```kotlin
val config = SessionReplayConfig(
    domain = "your-domain.noibu.com",
    logLevel = LogLevel.INFO
)
```

Logging is **off by default** — leave `logLevel` unset and the SDK logs nothing. Set a level to opt in.

---

## 4. Initialization

### Basic Setup

Create or update your `Application` class to initialize the SDK as early as possible:

```kotlin
import android.app.Application
import com.noibu.mobile.sessionreplay.SessionReplaySDK
import com.noibu.mobile.sessionreplay.SessionReplayConfig
import com.noibu.mobile.sessionreplay.PrivacyMode

class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()

        val config = SessionReplayConfig(
            domain = "your-domain.noibu.com",
            privacyMode = PrivacyMode.MASK_SENSITIVE
        )

        SessionReplaySDK.initialize(
            context = this,
            config = config
        )
    }
}
```

### Jetpack Compose Setup

Compose applications pass `NoibuComposeExtension` (from the `session-replay-compose` add-on) so the
SDK captures the Compose tree as well as the View tree:

```kotlin
import com.noibu.mobile.sessionreplay.compose.NoibuComposeExtension

SessionReplaySDK.initialize(
    context = this,
    config = config,
    extensions = listOf(NoibuComposeExtension())
)
```

An app that initializes with this extension is reported as a Compose app in Noibu, so its sessions
are distinguishable from a Fragment/XML app's.

### Register Your Application

Ensure your Application class is declared in `AndroidManifest.xml`:

```xml
<application
    android:name=".MyApplication"
    ... >
</application>
```

---

## 5. Page Navigation

Pick one of the three below — they are alternatives, not layers. Explicit wiring (the composable or
`trackNavigation`) wins and silences the automatic boundaries.

### Jetpack Navigation (Compose)

Drop the `NoibuNavigationTracking` composable into the scope that hosts your `NavHost`; every page
boundary is derived from the `NavController`, with no per-screen calls:

```kotlin
import com.noibu.mobile.sessionreplay.compose.NoibuNavigationTracking

val navController = rememberNavController()

NoibuNavigationTracking(navController)

NavHost(navController = navController, startDestination = "home") {
    // ...
}
```

A destination's page name defaults to its route; pass a `pageName` lambda to map it yourself.

### Automatic Activity/Fragment Tracking

Activity/Fragment apps can opt into a page per resumed screen with `autoTrackNavigation = true` in
the config, instead of calling anything per screen.

### Manual Page Tracking

For custom navigation flows or single-Activity architectures, call `trackNavigation()` from the
newly-shown screen (e.g. each `Fragment.onResume`):

```kotlin
// With a page name
SessionReplaySDK.trackNavigation("ProductDetails")

// Without a page name (uses the default)
SessionReplaySDK.trackNavigation()
```

**When to call `trackNavigation()`:**
- Navigation to a new screen in a single-Activity app
- Modal or bottom sheet presentations
- Tab switches that represent distinct pages
- Deep link handling

**What happens:**
1. Pending replay data is flushed for the current page
2. A new full snapshot is captured
3. Replay state is reset for fresh transformation
4. A new page visit is created in analytics

---

## 6. Error Reporting

Report errors to Noibu for tracking and analysis. Errors are associated with the current page visit.

### Reporting a Custom Error

```kotlin
// With a message only
SessionReplaySDK.addError("Payment processing failed")

// With a message and stack trace
SessionReplaySDK.addError(
    message = "Payment processing failed",
    stack = "at com.myapp.checkout.PaymentProcessor.process(PaymentProcessor.kt:42)"
)
```

### Reporting a Throwable

```kotlin
try {
    riskyOperation()
} catch (e: Exception) {
    SessionReplaySDK.addError(e)
}
```

The Throwable overload automatically extracts the message and stack trace.

### API Reference

| Method | Parameters | Description |
|--------|-----------|-------------|
| `addError(message, stack?)` | `message: String`, `stack: String? = null` | Report a custom error with an optional stack trace |
| `addError(throwable)` | `throwable: Throwable` | Report an error from a caught exception |

> **Note**: Up to 500 errors are reported per page visit; the count resets on the next page visit.

---

## 7. Network Monitoring

The SDK provides an OkHttp interceptor to automatically capture HTTP request and response data.

### Setup

Add the Noibu interceptor to your `OkHttpClient`:

```kotlin
import com.noibu.mobile.sessionreplay.api.installNetworkInstrumentation

val client = OkHttpClient.Builder()
    .installNetworkInstrumentation()
    .build()
```

### What Is Captured

| Data | Details |
|------|---------|
| Request metadata | HTTP method, URL, request size |
| Response metadata | Status code, response size, duration |
| Request/response headers | All headers (sensitive headers are automatically redacted) |
| Request/response bodies | Text-based content types only (JSON, XML, form data), up to 64KB |
| Errors | Network failures and exceptions |

WebSocket connections are not reported as requests — a WebSocket has no response to record and stays
open long after it opens, and a socket that reconnects would report a failed request for every
attempt. The interceptor is also safe to install before `SessionReplaySDK.initialize`, and to leave
in place after `shutdown()`: it passes a request straight through, without reading headers or bodies,
while capture is off or `trackNetwork` is off.

### Privacy

The following headers are automatically redacted:

`authorization`, `cookie`, `set-cookie`, `x-auth-token`, `x-api-key`, `proxy-authorization`, `x-forwarded-for`

Bodies and the remaining header values are PII-redacted before they reach the device cache: JSON
values under sensitive key names (`password`, `email`, `cardNumber`, `address`, …) are replaced
with `******`, then credit-card, email, SSN/SIN and phone patterns are scrubbed from the text.

### Advanced Configuration

You can disable raw HTTP body/header capture by using `NetworkInterceptor` directly:

```kotlin
import com.noibu.mobile.sessionreplay.api.NetworkInterceptor

val client = OkHttpClient.Builder()
    .addInterceptor(NetworkInterceptor(captureHttpData = false))
    .build()
```

When `captureHttpData` is `false`, only request/response metadata (method, URL, status code, duration, sizes) is captured.

---

## 8. WebView Support

The Noibu SDK can capture session replay data from WebViews, rendering webview content as part of the same mobile session recording. The SDK automatically injects a DOM recorder into the WebView — no additional setup is required on the web page.

### Prerequisites

- JavaScript must be enabled on the WebView.

### Setup

Call `WebViewTracking.enable()` after creating your WebView:

```kotlin
import android.webkit.WebView
import com.noibu.mobile.sessionreplay.api.WebViewTracking

val webView: WebView = findViewById(R.id.webview)
webView.settings.javaScriptEnabled = true

WebViewTracking.enable(webView)
```

### Jetpack Compose

For Compose apps, use `AndroidView` to host the WebView:

```kotlin
import android.webkit.WebView
import android.webkit.WebViewClient
import androidx.compose.ui.viewinterop.AndroidView
import com.noibu.mobile.sessionreplay.api.WebViewTracking

AndroidView(
    factory = { context ->
        WebView(context).apply {
            settings.javaScriptEnabled = true
            settings.domStorageEnabled = true
            webViewClient = WebViewClient()

            WebViewTracking.enable(this)

            loadUrl("https://example.com")
        }
    }
)
```

### Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `webView` | `WebView` | **Yes** | - | The WebView instance to track |

### How It Works

1. The SDK injects a DOM recorder into every page loaded in the WebView
2. DOM snapshots and mutations are captured and routed through a native bridge
3. Replay records are enriched with a `slotId` that links them to the native WebView container
4. Records are written directly to the replay outbox and sent alongside native snapshots
5. Input values are masked in replay, and a click on a field reports its label rather than its
   contents — the same rule the native view walker applies to a text field

### Stopping Capture

`WebViewTracking.disable(webView)` stops capture for one WebView. To turn webview capture off for
the whole app, set `trackWebViews = false` in the config — `enable()` then becomes a no-op and
nothing is injected into any webview.

### Important Notes

- Call `enable()` **before** loading any URL in the WebView
- Call `enable()` **after** `SessionReplaySDK.initialize()` — the SDK must be running
- Each WebView instance gets a unique `slotId` so multiple WebViews in the same screen are tracked independently
- WebView replay data is associated with the current native page view — report the navigation before presenting a new screen with a WebView

---

## 9. View Tagging

For Jetpack Compose apps, use the `noibuTag` modifier to add semantic identifiers to UI elements for tap/click action tracking.

### Usage

```kotlin
import com.noibu.mobile.sessionreplay.compose.noibuTag

Button(
    onClick = { /* ... */ },
    modifier = Modifier.noibuTag("AddToCartButton")
) {
    Text("Add to Cart")
}
```

### Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `name` | `String` | - | Semantic identifier for the UI element |
| `isImage` | `Boolean` | `false` | Kept for source compatibility with earlier releases; unused |

### Automatic Tagging

If you use the Noibu Gradle Plugin (`com.noibu.gradle.plugin`), Compose elements are auto-tagged at compile time. Manual `noibuTag` calls are useful for overriding auto-generated names or tagging elements that the plugin may not reach.

---

## 10. Custom Attributes

Add metadata to sessions for filtering and analysis in the Noibu dashboard.

### Adding Attributes

```kotlin
SessionReplaySDK.addCustomAttribute("customerId", "12345")
SessionReplaySDK.addCustomAttribute("orderId", "ORD-98765")
SessionReplaySDK.addCustomAttribute("appVersion", "2.1.0")
```

### Validation Rules

| Rule | Limit |
|------|-------|
| Maximum attributes per session | 10 |
| Attribute name length | 1-50 characters |
| Attribute value length | 1-50 characters |
| Duplicate names | Not allowed |

`addCustomAttribute()` is fire-and-forget and returns `Unit`. Attributes that violate these rules
are ignored; validate in application code if the caller needs immediate feedback.

---

## 11. Privacy & Security

### Privacy Mode Selection

```kotlin
// card / SSN / email / phone spans redacted in displayed text — default
privacyMode = PrivacyMode.MASK_SENSITIVE

// no readable text at all — every string keeps only its word/length shape
privacyMode = PrivacyMode.MASK_ALL

// displayed text verbatim
privacyMode = PrivacyMode.ALLOW_ALL
```

Typed input content never reaches the pipeline in any mode: the walker replaces it with the field's
hint (or `***`) as it walks, before an event is built. `MASK_ALL` additionally masks the hint.

### Data Storage

- Session data is streamed to Noibu's servers
- Local data is stored temporarily in the app's cache directory
- Data is automatically cleared after successful upload

---

## 12. Lifecycle Management

### Automatic Background Handling

The SDK automatically handles app lifecycle:
- **App goes to background**: Pending data is flushed
- **App returns after 30+ seconds**: A new page view is started automatically

### Checking Initialization Status

```kotlin
if (SessionReplaySDK.isInitialized) {
    Log.d("Noibu", "Noibu SDK is running")
}
```

### Shutdown

To completely stop the SDK (e.g., user revokes consent):

```kotlin
SessionReplaySDK.shutdown()
```

This will:
- Stop all recording
- Flush pending data
- Clear custom attributes
- Remove lifecycle observers


---

## Troubleshooting

### Verify Installation

Check logcat for SDK initialization:

```
adb logcat | grep -i noibu
```

### Common Issues

| Issue | Solution |
|-------|----------|
| No recordings appearing | Verify `domain` is correct |
| Need detail when debugging an integration | Set `logLevel = LogLevel.INFO` in the config, then `adb logcat | grep -i noibu` (see [Diagnostic Logging](#diagnostic-logging)) |
| Compose screens not captured | Ensure the `session-replay-compose` dependency is added and `NoibuComposeExtension` is passed to `initialize()` |
| Network requests not captured | Ensure `installNetworkInstrumentation()` is called on your `OkHttpClient.Builder` |
| Errors not appearing | Verify `addError()` is called after `initialize()` |
| Pages not splitting | Wire exactly one navigation source — `NoibuNavigationTracking`, `autoTrackNavigation = true`, or `trackNavigation()` (see [Page Navigation](#5-page-navigation)) |
| WebView content not in replay | Verify `WebViewTracking.enable()` runs after `initialize()` and before the URL loads, and that `trackWebViews` is left on |

---

## Support

Need help? Contact your Noibu solutions engineer with:
- SDK version
- Android version
- Initialization code snippet
- Logcat output
