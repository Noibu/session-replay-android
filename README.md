# Noibu Android SDK Guide

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
    implementation("com.noibu.mobile.kmp:session-replay-android:1.0.0-alpha04")
}
```

Sync Gradle to download the artifacts.

---

## 3. Configuration

### NoibuSessionReplayConfig

The SDK is configured via `NoibuSessionReplayConfig`. All parameters:

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `domain` | `String` | **Yes** | - | Your Noibu domain endpoint (provided by Noibu) |
| `privacyMode` | `PrivacyMode` | No | `MASK_SENSITIVE` | Privacy level for session recording |

### Privacy Modes

| Mode | Description |
|------|-------------|
| `PrivacyMode.ALLOW_ALL` | All content is visible in recordings |
| `PrivacyMode.MASK_SENSITIVE` | Masks passwords, credit card fields, and other sensitive inputs |
| `PrivacyMode.MASK_ALL` | Masks all text content |

---

## 4. Initialization

### Basic Setup

Create or update your `Application` class to initialize the SDK as early as possible:

```kotlin
import android.app.Application
import com.noibu.mobile.kmp.sessionreplay.NoibuSessionReplay
import com.noibu.mobile.kmp.sessionreplay.NoibuSessionReplayConfig
import com.noibu.mobile.kmp.sessionreplay.PrivacyMode

class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()

        val config = NoibuSessionReplayConfig(
            domain = "your-domain.noibu.com",
            privacyMode = PrivacyMode.MASK_SENSITIVE
        )

        NoibuSessionReplay.initialize(
            context = this,
            config = config
        )
    }
}
```

### Jetpack Compose Setup

For Compose applications, add the `NoibuComposeExtension` to enable proper gesture tracking:

```kotlin
import com.noibu.mobile.kmp.sessionreplay.NoibuComposeExtension

NoibuSessionReplay.initialize(
    context = this,
    config = config,
    extensions = listOf(NoibuComposeExtension())
)
```

### Material Design Support

For apps using Material Design components, add the `NoibuMaterialExtension`:

```kotlin
import com.noibu.mobile.kmp.sessionreplay.NoibuComposeExtension
import com.noibu.mobile.kmp.sessionreplay.NoibuMaterialExtension

NoibuSessionReplay.initialize(
    context = this,
    config = config,
    extensions = listOf(
        NoibuComposeExtension(),
        NoibuMaterialExtension()
    )
)
```

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

### Automatic Tracking

The SDK automatically tracks Activity-based navigation. For Compose apps with `NoibuComposeExtension`, navigation destinations are tracked automatically.

### Manual Page Tracking

For custom navigation flows or single-Activity architectures, call `didNavigate()` when the user navigates to a new screen:

```kotlin
// With a page name
NoibuSessionReplay.didNavigate("ProductDetails")

// Without a page name (uses default)
NoibuSessionReplay.didNavigate()
```

**When to call `didNavigate()`:**
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
NoibuSessionReplay.addError("Payment processing failed")

// With a message and stack trace
NoibuSessionReplay.addError(
    message = "Payment processing failed",
    stack = "at com.myapp.checkout.PaymentProcessor.process(PaymentProcessor.kt:42)"
)
```

### Reporting a Throwable

```kotlin
try {
    riskyOperation()
} catch (e: Exception) {
    NoibuSessionReplay.addError(e)
}
```

The Throwable overload automatically extracts the message and stack trace.

### API Reference

| Method | Parameters | Description |
|--------|-----------|-------------|
| `addError(message, stack?, attributes?)` | `message: String`, `stack: String?`, `attributes: Map<String, String>` | Report a custom error with an optional stack trace |
| `addError(throwable, attributes?)` | `throwable: Throwable`, `attributes: Map<String, String>` | Report an error from a caught exception |

> **Note**: The `attributes` parameter is reserved for future use.

---

## 7. Network Monitoring

The SDK provides an OkHttp interceptor to automatically capture HTTP request and response data.

### Setup

Add the Noibu interceptor to your `OkHttpClient`:

```kotlin
import com.noibu.mobile.kmp.sessionreplay.installNoibuNetworkInstrumentation

val client = OkHttpClient.Builder()
    .installNoibuNetworkInstrumentation()
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

### Privacy

The following headers are automatically redacted:

`authorization`, `cookie`, `set-cookie`, `x-auth-token`, `x-api-key`, `proxy-authorization`, `x-forwarded-for`

### Advanced Configuration

You can disable raw HTTP body/header capture by using `NoibuNetworkInterceptor` directly:

```kotlin
import com.noibu.mobile.kmp.sessionreplay.NoibuNetworkInterceptor

val client = OkHttpClient.Builder()
    .addInterceptor(NoibuNetworkInterceptor(captureHttpData = false))
    .build()
```

When `captureHttpData` is `false`, only request/response metadata (method, URL, status code, duration, sizes) is captured.

---

## 8. WebView Support

The Noibu SDK can capture session replay data from WebViews, rendering webview content as part of the same mobile session recording. The SDK automatically injects a DOM recorder into the WebView — no additional setup is required on the web page.

### Prerequisites

- JavaScript must be enabled on the WebView.

### Setup

Call `NoibuWebViewTracking.enable()` after creating your WebView:

```kotlin
import android.webkit.WebView
import com.noibu.mobile.sessionreplay.api.NoibuWebViewTracking

val webView: WebView = findViewById(R.id.webview)
webView.settings.javaScriptEnabled = true

NoibuWebViewTracking.enable(webView)
```

### Jetpack Compose

For Compose apps, use `AndroidView` to host the WebView:

```kotlin
import android.webkit.WebView
import android.webkit.WebViewClient
import androidx.compose.ui.viewinterop.AndroidView
import com.noibu.mobile.sessionreplay.api.NoibuWebViewTracking

AndroidView(
    factory = { context ->
        WebView(context).apply {
            settings.javaScriptEnabled = true
            settings.domStorageEnabled = true
            webViewClient = WebViewClient()

            NoibuWebViewTracking.enable(this)

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

### Important Notes

- Call `enable()` **before** loading any URL in the WebView
- Call `enable()` **after** `NoibuSessionReplay.initialize()` — the SDK must be running
- Each WebView instance gets a unique `slotId` so multiple WebViews in the same screen are tracked independently
- WebView replay data is associated with the current native page view — call `didNavigate()` before presenting a new screen with a WebView

---

## 9. View Tagging

For Jetpack Compose apps, use the `noibuTag` modifier to add semantic identifiers to UI elements for tap/click action tracking.

### Usage

```kotlin
import com.noibu.mobile.kmp.sessionreplay.noibuTag

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
| `isImage` | `Boolean` | `false` | Set to `true` if the element represents an image |

### Automatic Tagging

If you use the Noibu Gradle Plugin (`com.noibu.gradle.plugin`), Compose elements are auto-tagged at compile time. Manual `noibuTag` calls are useful for overriding auto-generated names or tagging elements that the plugin may not reach.

---

## 10. Custom Attributes

Add metadata to sessions for filtering and analysis in the Noibu dashboard.

### Adding Attributes

```kotlin
NoibuSessionReplay.addCustomAttribute("customerId", "12345")
NoibuSessionReplay.addCustomAttribute("orderId", "ORD-98765")
NoibuSessionReplay.addCustomAttribute("appVersion", "2.1.0")
```

### Validation Rules

| Rule | Limit |
|------|-------|
| Maximum attributes per session | 10 |
| Attribute name length | 1-50 characters |
| Attribute value length | 1-50 characters |
| Duplicate names | Not allowed |

### Return Values

The `addCustomAttribute()` method returns a `CustomAttributeResult`:

| Result | Description |
|--------|-------------|
| `SUCCESS` | Attribute added successfully |
| `TOO_MANY_IDS_ADDED` | Maximum of 10 attributes reached |
| `ID_NAME_ALREADY_ADDED` | Attribute with this name already exists |
| `NAME_TOO_LONG` | Name exceeds 50 characters |
| `VALUE_TOO_LONG` | Value exceeds 50 characters |
| `INVALID_NAME_TYPE` | Name is not a valid string type |
| `INVALID_VALUE_TYPE` | Value is not a valid string type |
| `NAME_HAS_NO_LENGTH` | Name is empty |
| `VALUE_HAS_NO_LENGTH` | Value is empty |
| `NOT_INITIALIZED` | SDK not initialized |

### Example with Error Handling

```kotlin
val result = NoibuSessionReplay.addCustomAttribute("customerId", customerId)
when (result) {
    CustomAttributeResult.SUCCESS -> {
        // Attribute added successfully
    }
    CustomAttributeResult.TOO_MANY_IDS_ADDED -> {
        Log.w("Noibu", "Maximum custom attributes reached")
    }
    else -> {
        Log.w("Noibu", "Failed to add attribute: ${result.message}")
    }
}
```

---

## 11. Privacy & Security

### Privacy Mode Selection

Choose the appropriate privacy mode based on your app's requirements:

```kotlin
// For general apps - masks only sensitive fields (default)
privacyMode = PrivacyMode.MASK_SENSITIVE

// For apps handling highly sensitive data
privacyMode = PrivacyMode.MASK_ALL

// For internal/debug builds only
privacyMode = PrivacyMode.ALLOW_ALL
```

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

### Shutdown

To completely stop the SDK (e.g., user revokes consent):

```kotlin
NoibuSessionReplay.shutdown()
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
| Compose gestures not tracked | Ensure `NoibuComposeExtension` is added |
| Network requests not captured | Ensure `installNoibuNetworkInstrumentation()` is called on your `OkHttpClient.Builder` |
| Errors not appearing | Verify `addError()` is called after `initialize()` |
| WebView content not in replay | Ensure the web page includes the Noibu browser snippet with session replay enabled, and that the host is in `allowedHosts` |

---

## Support

Need help? Contact your Noibu solutions engineer with:
- SDK version
- Android version
- Initialization code snippet
- Logcat output
- App version
