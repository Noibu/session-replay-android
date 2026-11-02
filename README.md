# Noibu Android SDK Guide

[Android](https://central.sonatype.com/artifact/com.noibu.mobile.android/session-replay) · [Jetpack Compose](https://central.sonatype.com/artifact/com.noibu.mobile.android/session-replay-compose)

## Table of Contents

- [1. Requirements](#1-requirements)
- [2. Installation](#2-installation)
- [2b. Gradle Plugin (optional)](#2b-gradle-plugin-optional)
- [3. Configuration](#3-configuration)
- [4. Initialization](#4-initialization)
- [5. Pages (trackNavigation)](#5-pages-tracknavigation)
- [6. Error Reporting](#6-error-reporting)
- [7. Network Monitoring](#7-network-monitoring)
- [8. WebView Support](#8-webview-support)
- [9. View Tagging](#9-view-tagging)
- [10. Custom Attributes](#10-custom-attributes)
- [10b. Tracking Events](#10b-tracking-events)
- [11. Privacy & Security](#11-privacy--security)
- [12. Lifecycle Management](#12-lifecycle-management)

---

## 1. Requirements

- **Minimum Android SDK**: API 26 (Android 8.0)
- **Compile SDK**: 36 or higher
- **Kotlin**: 2.1+ — the SDK is built with Kotlin 2.2, and a Kotlin plugin older than 2.1 cannot read its metadata (the optional [Gradle plugin](#2b-gradle-plugin-optional)'s Compose compiler plugin is built against Kotlin 2.2.10)
- **Gradle**: 8.0+

---

## 2. Installation

Add the Noibu Session Replay SDK to your app module's `build.gradle.kts`:

```kotlin
dependencies {
    implementation("com.noibu.mobile.android:session-replay:2.1.0")
}
```

Jetpack Compose apps add the Compose add-on as well. It is published from this SDK's release tag
and always carries the same version:

```kotlin
dependencies {
    implementation("com.noibu.mobile.android:session-replay:2.1.0")
    implementation("com.noibu.mobile.android:session-replay-compose:2.1.0")
}
```

Fragment/XML-only apps depend on `session-replay` alone and never pull Compose.

Sync Gradle to download the artifacts. See the [changelog](CHANGELOG.md) for what changed in this
release.

### What the artifact brings

- The AAR is self-contained: Noibu's shared core is merged into it. Add only the coordinates above —
  never a separate core dependency.
- OkHttp 4.12.0 is a transitive dependency (the network interceptor is built on it). Having it on the
  classpath captures nothing by itself: see [Capture network requests](#capture-network-requests).
- Nothing to configure for R8/ProGuard: the consumer rules the SDK needs ship inside the AAR.
- From Java, call `SessionReplaySDK.INSTANCE.initialize(...)`, `SessionReplaySDK.INSTANCE.track(...)`
  and so on; `WebViewTracking.enable(webView)` is the one entry point that is a plain static.

---

## 2b. Gradle Plugin (optional)

The SDK works without any build plugin. The Noibu Gradle Plugin adds two build-time features:

| Feature | Setting | Default | What it does |
|---------|---------|---------|--------------|
| Bundled-font capture | `enableFontCapture` | `true` | Generates a manifest of the fonts under `src/main/res/font` and `src/main/assets/fonts` so the replay renders your custom fonts. Without it, text in the replay falls back to a default font. |
| Compose instrumentation | `enableComposeInstrumentation` | `false` | A Kotlin compiler plugin that tags clickable composables with their function name, as a last-resort selector for clickables that carry no text or `contentDescription` (see [Automatic Tagging](#automatic-tagging)). |

The plugin is published to Maven Central (artifact `com.noibu.mobile.android:gradle-plugin`) and is
built against Android Gradle Plugin 9.0, so it needs AGP 9.0 or newer. Gradle looks for plugins only
in the Gradle Plugin Portal unless told otherwise, so add Maven Central in `settings.gradle.kts`:

```kotlin
pluginManagement {
    repositories {
        gradlePluginPortal()
        google()
        mavenCentral()
    }
}
```

Then apply it in the app module:

```kotlin
plugins {
    id("com.noibu.gradle.plugin") version "1.0.0"
}

noibu {
    enableComposeInstrumentation = true   // Compose apps only
    enableFontCapture = true
}
```

XML and React Native apps that only want font capture can apply `com.noibu.gradle.fonts` (same version) instead —
the same font manifest with no Compose compiler plugin.

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

### Capture network requests

Unlike iOS, where HTTP capture starts with `initialize`, Android records only requests made through an
`OkHttpClient` you instrument. Add the interceptor to every builder the app uses, including the ones
your dependency-injection module hands to Retrofit:

```kotlin
val client = OkHttpClient.Builder()
    .installNetworkInstrumentation()
    .build()
```

Without it, sessions carry no HTTP data at all. Other HTTP stacks (Ktor's CIO or Android engines,
Volley, Cronet, `HttpURLConnection`) are not captured; Ktor's OkHttp engine is, through
`engine { addInterceptor(NetworkInterceptor()) }`. Details in [Network Monitoring](#7-network-monitoring).

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

`initialize()` is idempotent: a second call while the SDK is running is ignored, so re-running your
startup code (a plugin framework re-mounting, for instance) never doubles capture. After `shutdown()`
the next `initialize()` starts the SDK again.

### Register Your Application

Ensure your Application class is declared in `AndroidManifest.xml`:

```xml
<application
    android:name=".MyApplication"
    ... >
</application>
```

---

<a id="5-page-navigation"></a>

## 5. Pages (trackNavigation)

> **How pages work in 2.0**
> - Pages are reported explicitly. The SDK no longer detects screens: no Activity, Fragment or
>   navigation destination opens a page on its own.
> - Call `trackNavigation(screenName, attributes, title)` for every page, including tabs, relevant
>   modals and WebView steps (see [Web steps as pages](#web-steps-as-pages)).
> - Track page groups, not individual views: `screenName` is the page group (`Product Detail`,
>   `Checkout`), and attributes carry the variations (`productId`, `step`).
> - Report every page entry, including Back.
> - Repeating the same name and attributes continues the current visit; changing the attributes starts
>   a new one.
> - Navigation helpers (`NoibuNavigationTracking`) don't report pages; they only manage transitions.

### Page groups

**A screen name is a page group, not a view.** A *page* is everything the user experiences as one
place in your app — not a single Activity, Fragment or composable. Page analysis, heatmaps and Journeys
aggregate by screen name, so every view of that place reports the same one:

- every product detail, its gallery, size picker and reviews sheet → `Product Detail`
- every category or listing → `Product List`
- the address, payment and review steps of checkout → `Checkout`

| Think of it as… | Not as… |
|---|---|
| `Product Detail` (all product details) | `ProductDetailActivity`, `ProductDetailScreen` |
| `Product List` (any listing) | `WomenTopsCategoryFragment` |
| `Checkout` (all steps) | `CheckoutAddressScreen`, `CheckoutPaymentScreen` |

### Reporting a page

Call `trackNavigation()` from the newly shown screen — each `Fragment.onResume`, or a Compose
destination's `LaunchedEffect`:

```kotlin
import com.noibu.mobile.sessionreplay.PageType

SessionReplaySDK.trackNavigation(
    screenName = "Product Detail",
    attributes = mapOf("pageType" to PageType.PRODUCT, "category" to "Clothing", "productId" to "123"),
    title = "Sport Backpack",
)
```

- **`screenName`** — the page group the user is in (`Product Detail`, `Cart`, `Checkout`), shared by
  every view of that place. Trimmed, and the page group is cut at 100 characters; a blank name reports
  nothing.
- **`attributes`** — what varies inside the page (`category`, `productId`, `step`…). The first 20 keys
  of 1–50 characters; values are strings (cut at 200 characters), booleans, finite numbers or `null`.
  Anything else is dropped, never thrown.
- **`title`** (optional) — the page's human-readable title, what Noibu shows for it (the product's
  name on a product page, say). Without one, the screen name is the title. Trimmed and cut at 1024
  characters. The title does not identify the visit: a later call with the same name and attributes
  continues the visit and keeps its first title, so pass the title on the call that opens the page.
- **`pageType`** (optional attribute) — places the page in a standard ecommerce category:
  `PageType.HOME`, `COLLECTION`, `SEARCH`, `PRODUCT`, `CART`, `CHECKOUT`, `LOGIN` (matched ignoring
  case). Journeys groups and colours the page under that category; without it, the page is grouped by
  its screen name. Only the exact key `pageType` is read: `pagetype` or `page_type` is an ordinary
  attribute.

**Repeated calls are safe.** Lifecycle callbacks such as `onResume` or a `LaunchedEffect` can fire again
while the user stays on the same page (a dismissed dialog, a configuration change). Reporting the page
that is already current, with unchanged attributes, continues the existing visit; only a different name
or different attributes starts a new one.

**When to call `trackNavigation()`:**
- The user enters a different page (not when moving between views of the same page), including when
  Back returns to one
- Modal or bottom sheet presentations that are a page of their own
- Tab switches that represent distinct pages
- Web steps inside a WebView (see [Web steps as pages](#web-steps-as-pages))
- Deep link handling

When a call starts a new visit, it appears in the session timeline and the events that follow (taps,
errors, network requests, tracked events) are associated with it.

### Jetpack Navigation (Compose)

Drop the `NoibuNavigationTracking` composable into the scope that hosts your `NavHost`. It keeps replay
transitions clean (and holds events a new destination tracks until its `trackNavigation()` call); it
does not report pages:

```kotlin
import com.noibu.mobile.sessionreplay.compose.NoibuNavigationTracking

val navController = rememberNavController()
NoibuNavigationTracking(navController)

NavHost(navController = navController, startDestination = "home") {
    // ...
}
```

Report each destination from a `LaunchedEffect(entry)` inside its `composable { entry -> … }` block.

### Naming your pages

| Screen | `screenName` | `pageType` | Example attributes |
|---|---|---|---|
| Home | `Home` | `Home` | — |
| Category listing | `Product List` | `Collection` | `category`, `subcategory` |
| Search results | `Search` | `Search` | `query` |
| Product detail | `Product Detail` | `Product` | `category`, `productType`, `productId` |
| Cart | `Cart` | `Cart` | — |
| Checkout | `Checkout` | `Checkout` | `step` |
| Order confirmation | `Order Confirmed` | — | `orderId` |
| Sign in | `Login` | `Login` | — |

- **Template, not content:** every product detail is `Product Detail`; ids and user data never go in
  the name.
- **Hierarchy and ids go in attributes** (`Product Detail` + `category`, not
  `Product Detail - Clothing - Tops`).
- **Attributes identify the visit:** the same name with different attributes starts a new visit.
  `null` is a value: `"size" to null` and no `size` key are different visits.
  Every view of a page reports the same attributes; transient state (selected size, scroll position)
  stays out.
- **Small and stable:** well under 100 names, unchanged between releases. Reserved names (`None`,
  `Page Check`, `External traffic`, `Exit`, `Other pages`, `No Page Group`, `All`) are sent without a group.

### What happens without a call

The SDK opens a page on its own when there is no call to take it:

- **On a fresh start** (no previous screen), events, errors and replay land on an unnamed `view/<uuid>`
  page with no group until the first call. If the app reports its first screen within 2 seconds, or
  before any replay frame is captured, that screen takes the page over; otherwise the placeholder is
  sent as a visit of its own.
- **On a return from background, a relaunch or a new session**, the next page copies the previous
  screen's name, group, attributes and title.
- **An app that never calls `trackNavigation`** still records sessions, made of `view/<uuid>` pages.

### Migrating from 1.x

| 1.x | 2.0 |
|---|---|
| `SessionReplaySDK.trackNavigation(pageName)` / `trackNavigation()` | `trackNavigation(screenName, attributes)` |
| `autoTrackNavigation = true` (a page per resumed Activity/Fragment) | Removed — call `trackNavigation` from each screen's `onResume` |
| `NoibuNavigationTracking(navController, pageName = …)` | `NoibuNavigationTracking(navController)` + `trackNavigation` from each destination |

Apps that relied on automatic tracking must now call `trackNavigation` on every page; the
`NoibuNavigationTracking` install line now only keeps transitions clean.

---

### Bottom tab bars in page analysis

Page analysis pins the screen's bottom bar to the bottom of the snapshot while the rest of the page
expands. The SDK recognises Material's `BottomNavigationView`, `NavigationBarView` and `BottomAppBar`,
and any custom view whose class name contains `TabBar`, `BottomNavigation`, `NavigationBar` or
`BottomAppBar`. A framework that keeps the role elsewhere (React Native's `accessibilityRole`) names it
through `SessionReplaySDK.registerRoleHintResolver`.

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
Capture is OkHttp-only: traffic that does not go through an instrumented `OkHttpClient`
(`HttpURLConnection`, other HTTP stacks) is not recorded. Every captured request is reported as
first-party, and there is no URL ignore list — route traffic you do not want captured through a
client built without the interceptor.

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
while capture is off or `trackNetwork` is off. `installNetworkInstrumentation()` is idempotent — a
builder derived from an instrumented client (`client.newBuilder()`) keeps its single interceptor.

### Privacy

The following headers are automatically redacted:

`authorization`, `cookie`, `set-cookie`, `x-auth-token`, `x-api-key`, `proxy-authorization`, `x-forwarded-for`

Bodies and the remaining header values are PII-redacted before they reach the device cache. In a
JSON body, the value of any sensitive field is replaced with `******`, at any nesting depth:

| Match | Field names |
|-------|-------------|
| Exact (case-insensitive) | `firstname`, `lastname`, `fullname`, `street`, `city`, `town`, `county`, `postcode`, `zipcode`, `cc`, `cardtype`, `cardnumber`, `creditcard`, `email` |
| Contains | `password`, `address`, `credit`, `postal`, `token`, `phone`, `mobile`, `expiry`, `account`, `email` |

Then, in any text (JSON or not, headers included), card numbers (Visa, MasterCard, Amex, Diners Club,
Discover, JCB), email addresses, US SSNs, Canadian SINs and phone numbers are replaced with `******`.

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

One injected script captures the page's replay, its clicks, its JavaScript errors and its HTTP calls.
It opens no page: all of it lands on the current page, including after a navigation inside the WebView
(see [Web steps as pages](#web-steps-as-pages)).

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
| `onRouteChange` | `((String) -> Unit)?` | No | `null` | Called with the new route's `path?query`; see [Web steps as pages](#web-steps-as-pages) |

### How It Works

1. The SDK installs a DOM recorder as a document-start script, so it runs on every page the WebView
   loads — including in-page navigations — and survives a later `webViewClient` assignment
2. DOM snapshots and mutations are captured and routed through a native bridge
3. Replay records are enriched with a `slotId` that links them to the native WebView container
4. Records are written directly to the replay outbox and sent alongside native snapshots
5. Input values are masked in replay, and a click on a field reports its label rather than its
   contents — the same rule the native view walker applies to a text field

### Stopping Capture

`WebViewTracking.disable(webView)` stops capture for one WebView. To turn webview capture off for
the whole app, set `trackWebViews = false` in the config — `enable()` then becomes a no-op and
nothing is injected into any webview.

### Web steps as pages

To see each step of a web flow as its own page, report every step with `trackNavigation` under the
hosting screen's name, with the step's path as an attribute: the hosting screen reports the document
the WebView loads first, and `onRouteChange` reports each later one.

```kotlin
// path?query of the document the WebView loads first
private var webPath = "/checkout"

override fun onResume() {
    super.onResume()
    SessionReplaySDK.trackNavigation("Checkout", mapOf("path" to webPath))
}

// after creating the WebView, before loadUrl()
WebViewTracking.enable(webView, onRouteChange = { path ->
    webPath = path
    SessionReplaySDK.trackNavigation("Checkout", mapOf("path" to path))
})
webView.loadUrl("https://shop.example.com/checkout")
```

The hosting screen's first `path` is the key `onRouteChange` would use for that URL: its path and
query, with the fragment dropped and an empty path as `/` (`https://shop.example.com/checkout?step=1#top`
→ `/checkout?step=1`). For an inline document (`loadDataWithBaseURL`),
use its base URL (`https://example.com/` → `/`).

The SDK calls `onRouteChange` on the main thread with the new route's `path?query` (the fragment is
ignored) each time the page moves to a different one, through the browser history or a new document. It
skips the document the WebView loads first, which the hosting screen reports, and opens no page itself.

Returning to the first document later comes through `onRouteChange` with that same `path`.

When the hosting screen reports itself again (`onResume`), it reports the step it is on — the last
`path`, starting from the first document's — so the current visit continues.

### Important Notes

- Call `enable()` **before** the first `loadUrl()` so the first page is recorded from its start; a
  page already loaded when `enable()` runs is recorded from its next navigation
- Call `enable()` **after** `SessionReplaySDK.initialize()` — the SDK must be running
- `enable()` is idempotent: calling it again for the same WebView (e.g. on every Compose
  recomposition) is ignored rather than recording the page twice
- Each WebView instance gets a unique `slotId` so multiple WebViews in the same screen are tracked independently
- WebView replay data is associated with the current native page view — report the navigation before presenting a new screen with a WebView

---

## 9. View Tagging

Most Compose elements need no tagging: the SDK derives a stable selector for click tracking from the
element's own visible text or `contentDescription`. The `noibuTag` modifier overrides that derived
name. Use it for interactive elements with neither text nor a content description (an icon button, an
image), to tell repeated rows apart, or to pin a name that survives copy changes.

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

The [Noibu Gradle Plugin](#2b-gradle-plugin-optional) can also tag clickable composables with their
function name at compile time — only when `noibu { enableComposeInstrumentation = true }` is set
(off by default). That auto tag is the lowest-priority identity: an element's text or content
description wins over it, and a manual `noibuTag` wins over both. It matters only for clickables
that carry no content of their own.

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

## 10b. Tracking Events

`track(name, data)` reports ecommerce and custom events — the counterpart of NoibuJS's
`track(name, payload)`. Events appear on the session's timeline and in Explorations.

| Method | Parameters | Description |
|--------|-----------|-------------|
| `track(name, data)` | `name: String`, `data: Map<String, Any?>? = null` | Report an event; the map is encoded to JSON for you |
| `trackJson(name, dataJson)` | `name: String`, `dataJson: String?` | Same, with the payload already JSON-encoded (a `JSONObject.toString()`, for example) |

A `data` map may hold strings, numbers, booleans, `null`, nested maps with `String` keys, lists and
arrays, plus `JSONObject`/`JSONArray` values. Anything else (a `Date`, an enum, NaN) is rejected with
`Event payload is not JSON-serializable.` rather than being stringified.

```kotlin
// A standard ecommerce event: the payload follows the shared ecommerce schema.
SessionReplaySDK.track("product_added_to_cart", mapOf(
    "cartLine" to mapOf(
        "quantity" to 1,
        "merchandise" to mapOf("id" to "sku-42", "title" to "Running Shoes",
            "price" to mapOf("amount" to 89.99, "currencyCode" to "USD")),
    ),
))

// A custom event: any name, any JSON object.
val result = SessionReplaySDK.track("promo_banner_dismissed", mapOf("campaign" to "summer"))
if (!result.success) Log.w("Shop", result.errors.joinToString())

// Already-serialized JSON (e.g. a JSONObject) goes through trackJson.
SessionReplaySDK.trackJson("cart_viewed", cartJson.toString())
```

| Rule | Limit |
|------|-------|
| Event name length | ≤ 500 characters |
| Payload | a JSON object (or `null`), ≤ 10,000 characters serialized |
| Standard payloads | checked against the ecommerce schema; unknown properties dropped, wrong types rejected |
| Events per page visit | 500 |

`track()` never throws. It returns a `TrackResult`: `success` is `false` when the event was rejected
and `errors` lists why, in the same words as the web SDK. Before `initialize()` the result is
`success = false` with `SDK is not initialized.` — the event is not queued.

### Standard ecommerce events

A standard event's payload is validated against the shared ecommerce schema: every field is optional,
unknown properties are dropped, and a type mismatch is reported in `errors` and the event is not sent.
Any other event name is a custom event and only has to be a JSON object.

Shared shapes: `Money` is `{ amount, currencyCode }`; a `productVariant` is
`{ id, sku, title, price: Money, product: { id, title, type, url, vendor } }`; a `cartLine` is
`{ quantity, merchandise: productVariant, cost: { totalAmount: Money } }`.

| Event name | Payload key | Payload |
|---|---|---|
| `product_viewed` | `productVariant` | a product variant |
| `product_added_to_cart`, `product_removed_from_cart` | `cartLine` | a cart line |
| `cart_viewed` | `cart` | `{ id, totalQuantity, lines: [cartLine], cost: { totalAmount: Money } }` |
| `collection_viewed` | `collection` | `{ id, title, productVariants: [productVariant] }` |
| `search_submitted` | `searchResult` | `{ query, productVariants: [productVariant] }` |
| `checkout_started`, `checkout_address_info_submitted`, `checkout_contact_info_submitted`, `checkout_shipping_info_submitted`, `payment_info_submitted`, `checkout_completed` | `checkout` | `{ token, currencyCode, subtotalPrice, totalTax, totalPrice, lineItems: [{ id, title, quantity, variant: productVariant, finalLinePrice }], shippingLine: { price }, discountApplications, order: { id, customer: { id, isFirstOrder } }, transactions }` |

### Which page an event lands on

An event is attributed to the page visit that is current when it is tracked, with one allowance for
screen-view events fired while a screen is opening:

- With the Compose add-on and `NoibuNavigationTracking`, events tracked between a destination change
  and the `trackNavigation()` that follows it are held (up to three seconds) and land on the screen
  being entered, so `track("product_viewed", …)` from a `LaunchedEffect` is attributed correctly.
- With Fragments, call `trackNavigation()` in `onResume` before the screen's own events (LiveData
  observers deliver on `onStart`, before `onResume`).
- Off the main thread, `track()` is ordered behind a `trackNavigation()` already on its way to the main
  thread, so calling the two in sequence from a background thread keeps the event on the new page.

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

`privacyMode` is global — there is no per-view masking API on Android. Input values are always
masked; the mode only decides how much displayed text is redacted.

### Data Storage

- Session data is streamed to Noibu's servers
- Local data is stored temporarily in the app's cache directory
- Data is automatically cleared after successful upload

---

## 12. Lifecycle Management

### Automatic Background Handling

The SDK automatically handles app lifecycle:
- **App goes to background**: Pending data is flushed
- **App returns after 15+ minutes in the background**: A new session starts. The page is not
  changed by a return to the foreground — a shorter background stay resumes the same session and
  page.

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
| Pages not splitting | Call `trackNavigation()` from every screen (see [Pages](#5-pages-tracknavigation)) |
| WebView content not in replay | Verify `WebViewTracking.enable()` runs after `initialize()` and before the URL loads, and that `trackWebViews` is left on |
| `track()` answers `SDK is not initialized.` | Call `track()` after `initialize()`; events tracked before it are not queued |
| Custom fonts render as a default font in replay | Apply the [Gradle plugin](#2b-gradle-plugin-optional) with `enableFontCapture` left on |

---

## Support

Need help? Contact your Noibu solutions engineer with:
- SDK version
- Android version
- Initialization code snippet
- Logcat output
