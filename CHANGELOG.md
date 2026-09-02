# Changelog

All notable changes to the **Noibu Session Replay Android SDK** are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.0.1]

### Fixed
- **Sessions are now attributed to the SDK that recorded them.** Every session reported an internal
  build number rather than the SDK and version installed in your app, so a session recorded by this
  SDK could not be told apart from one recorded by another Noibu mobile SDK, or by an earlier release
  of this one. A session now reports this SDK and its version — and an app that uses the Compose
  add-on is reported as a Compose app. Nothing changes in an app and there is nothing to configure;
  what is captured is unchanged.

## [1.0.0]

### Changed
- **`installNetworkInstrumentation()` costs nothing while capture is off.** The interceptor is meant
  to sit on a client your app owns, which can outlive a capture session, so it now passes a request
  straight through — without reading headers or bodies — whenever the SDK is not initialized or
  `trackNetwork` is off. What gets captured while capture is on is unchanged, so it is safe to install
  the interceptor before `SessionReplaySDK.initialize` and to leave it in place after `shutdown()`.
- **WebSocket connections are no longer reported as HTTP requests.** A WebSocket has no response to
  record and stays open long after it opens, and a socket that reconnected reported a failed request
  for every attempt.

## [1.0.0-rc.3]

Everything since `1.0.0-alpha06`, including what first shipped in the `1.0.0-rc.1` prerelease —
published without release notes of its own, so its contents are folded in here.

### Changed
- **Maven coordinates (breaking).** The artifact moved so the group carries the platform and the
  artifact no longer repeats "noibu":

  ```kotlin
  // before
  implementation("com.noibu.mobile:session-replay-android:1.0.0-rc.1")
  // now
  implementation("com.noibu.mobile.android:session-replay:1.0.0-rc.3")
  ```

  `1.0.0-rc.1` remains on Maven Central at the old coordinates and is not being updated. Everything
  is still pre-1.0, so the move happens now rather than behind a deprecation cycle later.
- The Compose add-on is now published by this SDK's release tag as
  `com.noibu.mobile.android:session-replay-compose`, at the same version. It no longer has a tag of
  its own, so the two can no longer drift apart.

### Added
- `addError` is capped at 500 errors per page visit. A component stuck in an error loop, or a
  tracked webview whose page throws on every interaction, wrote one event per throw for the whole
  life of the screen. The cap resets on every page visit — including the ones the SDK opens itself
  after a background return, a relaunch or a session rotation.
- GraphQL errors are now detected on successful responses. A GraphQL server answers a failed
  operation with `200 OK` and an `errors` array, so a declined payment looked like a successful
  request. Core parses the captured response body of a 2xx GraphQL request (JSON at a `graphql`
  URL, or `application/graphql`) and reports one `gql` error event per entry, with the message,
  locations, path and extensions.
- Configurable diagnostic logging via `SessionReplayConfig.logLevel`. Logging is **off by
  default**; set `LogLevel.INFO` to surface integration-debugging diagnostics in Logcat
  (tag `NoibuMobile`). See [Diagnostic Logging](README.md#diagnostic-logging).

### Removed
- `SessionReplayConfig.ingestHost`. Where sends go is not an integration setting — a customer build
  always delivers to Noibu production — and a public field made it a supported knob that could be
  pointed anywhere. Noibu-side development redirects it with a debuggable app's
  `<meta-data android:name="com.noibu.dev.ingestHost">`, which only a debug SDK build reads.

### Fixed
- Diagnostic logging is genuinely off by default in the published SDK. The SDK's internal debug gate
  was inferred from the Gradle task name containing "release", and the publishing tasks are not
  named that way — so the released artifact shipped with verbose logging plus on-device log files
  enabled, and honoured the Noibu-internal dev ingest-host override. Publishing now forces the gate
  off and rejects any attempt to re-enable it.
- Line height is captured. `lineSpacingExtra` is broken on Android, so React Native sets `lineHeight`
  with a `LineHeightSpan` instead — nothing the view's own metrics report — and replay fell back to
  CSS `normal` (roughly 1.2x the font size). A paragraph styled looser replayed cramped and one
  styled tighter lost its last line to the box's clip. The span is now asked how tall it makes a
  line, and the answer ships as a `line-height` CSS override.
- Truncated text no longer spills across the elements beside it. A node carries the full string
  while the device drew an ellipsis, and single-line text is deliberately allowed to overflow its
  box in replay (the platform measures a hair narrower than the browser), so a
  `numberOfLines={1}` product title replayed on top of the price next to it. A `TextView` whose
  line is wider than its content box under an `ellipsize` mode now clips to its box.
- Underlined and struck-through text is captured. The walk asked the `TextView` for its typeface
  but never for its decoration flags, and `NodeStyle` carries no field for them, so a struck-out
  original price beside a sale price replayed as a second live price and an underlined link replayed
  as plain text. The decoration is now read off the paint the text's middle character is actually
  drawn with — the view's own `paintFlags` plus every span covering it, which is where React Native
  puts `textDecorationLine` (`ReactUnderlineSpan`/`ReactStrikethroughSpan`) — and ships as a
  `text-decoration` CSS override.
- Per-edge borders are captured. `NodeStyle`'s border is uniform, so a view bordered on one edge only
  (a list separator, an input underline, a tab indicator) replayed with no border at all — the React
  Native resolvers read the `borderWidth` shorthand and approximated a per-edge width by the left
  edge, which is unset for the common `borderBottomWidth` case. A non-uniform border now ships as
  `border-width`/`border-color` CSS overrides, and a sub-pixel width (React Native's
  `StyleSheet.hairlineWidth`) rounds up to 1px instead of truncating to nothing.
- Text weight and font are read from the spans that draw the text. The walk took them from
  `TextView.typeface`, which is the theme default whenever a framework styles text through spans
  instead — React Native puts `fontWeight`/`fontStyle`/`fontFamily` on a `CustomStyleSpan` (not the
  platform's `StyleSpan`) that resolves the typeface inside `updateDrawState`, so every bold heading,
  price and button label replayed at regular weight and a bundled custom font never resolved to its
  `@font-face` asset. iOS has always reported this (`UIFont` is set on the label).
- Children are captured in the order the framework draws them. Sibling order in a capture is
  replay's paint order, and the walk used child-index order — but a child raised with
  `elevation`/`translationZ` is re-sorted above its siblings at draw time, and a parent can install
  a custom drawing order (React Native does this for `zIndex` on the old architecture). So a raised
  card, a dropdown, a tooltip or a floating action button replayed underneath what it covers on
  device. iOS has always reported this (UIKit draws `subviews` in array order).
- Scaled and rotated views are captured at the size they cover. The walk read a view's position
  through `getLocationOnScreen`, which maps only the origin point, and its size from the layout
  bounds — so any view carrying a transform (React Native compiles every `transform` style and
  `Animated` scale/rotate into the platform's scale/rotation/translation properties) replayed at its
  untransformed size, and a rotated one also at the wrong place. The walk now maps the bounds rect
  through the same ancestor chain, so a transformed view and everything under it report the box they
  actually cover. iOS has always reported this (`convert(bounds, to: window)`).
- Gradient fills are captured. A node has no typed field for a gradient and no solid colour to fall
  back to, so a React Native view styled `backgroundImage: 'linear-gradient(...)'` (RN 0.76+)
  replayed as an empty box. The RN background resolver now converts the first linear-gradient layer
  to CSS and the walker ships it as a `background` override; radial gradients are skipped rather
  than replayed as the wrong shape. iOS has always reported this (`CAGradientLayer`).
- Per-corner rounding is captured. A node carries ONE `cornerRadius`, and the React Native
  background resolvers reported the top-left corner as that value — so a bottom sheet, a card
  rounding only its top or a chat bubble replayed with all four corners at the top-left's radius
  (square when the rounding was on the bottom). The walker now ships the four real corners as a
  `border-radius` CSS override when they differ, on both the 0.76+ typed drawables and the
  reflective pre-0.76 path. iOS has always reported this (`maskedCorners`).
- A view that does not clip its children is captured as `overflow: visible`. Replay divs clip
  unconditionally, which matches the framework's ViewGroup default but not React Native's
  `ReactViewGroup` — it clears `clipChildren` and re-clips only when the `overflow` style says so —
  so a badge past an avatar, a ribbon on a product card or a tab-bar label was cut out of the
  replay. The style is read through RN's public `getOverflow()`, so an explicitly hidden view still
  clips, as do scroll views (their overflow defaults to hidden, which is what keeps an off-screen
  list row from rendering outside the viewport).
- A `ProgressBar` is captured in replay. It paints an indeterminate/progress DRAWABLE — nothing the
  walker's background or image branches read — so every spinner (React Native's
  `<ActivityIndicator>`) and progress bar replayed as an empty box, and the session gave no sign the
  app was waiting on something. It is now rasterized like an unlabelled toggle. An INDETERMINATE
  indicator is frozen at its first raster: its rotation carries no information, and every accepted
  change stages and uploads a separate asset. A determinate bar keeps following its progress,
  bounded at 30 accepted changes.
- The replay walker's rasterization fingerprint no longer collides on flat fills. It hashed pixels
  with a `hash * 31 + pixel` fold, which is linear — a UNIFORM raster reduces to a fixed geometric
  sum times the colour, so two flat fills differing only in low-order bits (black → half-red) hashed
  identically and the view froze at its first render. It now uses FNV-1a, the same hash the iOS
  walker uses.
- A switch/checkbox with no label is captured in replay. A `CompoundButton` draws its track and
  thumb as BUTTON drawables — nothing the walker's background or image branches read — so every
  toggle (React Native's `<Switch>`, a plain unlabelled `CheckBox`) replayed as an empty box. The
  walker now rasterizes it the way it already rasterizes a react-native-svg vector: a fingerprint
  raster derives the asset ref, so the replayed toggle follows its on/off state, bounded at 30
  accepted changes per view. A LABELLED compound button is unchanged — its text node already
  renders it, and rasterizing would draw the label twice.
- A focused field with no hint is named by its view tag. Keyboard events fell back to the literal
  string `TextField` when the field had neither a hint nor a content description, so a form of
  unlabelled inputs produced a run of identical, unusable events. The naming rule is now
  hint → content description → view tag → `TextField` — the view tag is where React Native puts
  `testID` (already shipped as the replay node's `accessibilityId`), and it is what iOS reads as
  `accessibilityIdentifier`.
- Every field of a form is reported, not just the first. Keyboard events fired on the IME's show
  transition only, and moving from one text field to the next leaves the keyboard up — so a
  checkout or sign-up form produced ONE event, naming the first field the user tapped. Focus moving
  to another `EditText` while the keyboard is up is now its own event; the IME hiding and re-showing
  on the same field (which Android does when the input type changes) still reports once. Compose
  text fields are unchanged — focus stays on the `AndroidComposeView`, so moving between them is not
  a view-level focus change.
- A `react-native-svg` graphic that re-renders is updated in replay. Its replay asset was keyed on
  the `SvgView`'s identity, but React Native keeps one such view for the component's whole life and
  redraws its vector in place — so a tab-bar icon recolouring on focus, an outline/filled toggle or
  a redrawn chart replayed its first render for the rest of the session. The asset ref now comes
  from a small rasterized fingerprint of what the view actually draws (bounded at 30 changes per
  view, so an animating vector doesn't upload a frame per capture).
- Dialog windows are captured. Anything shown in its own window — `AlertDialog`, `DialogFragment`,
  Compose `Dialog`, a popup menu, a toast, React Native's `<Modal>` — lives in a separate view root,
  and replay only ever walked the Activity's decor view, so those were missing from the session
  entirely: the replay showed the screen underneath while the user was interacting with the dialog.
  Capture now also walks the windows stacked above the Activity's and ticks on their draws, so a
  dialog is recorded live and its controls resolve as click targets.
- Taps and scrolls inside a dialog window are captured. Touch tracking wrapped the Activity
  window's `Window.Callback` only, and a dialog dispatches through its own window — so every tap on
  a dialog's buttons (a confirm/cancel, a React Native `<Modal>`, a Compose `Dialog`) produced no
  click event and no clickmap selector. Those windows are now observed for as long as they are up.
- Keyboard events are reported for text fields inside a dialog window. The IME and the focused view
  are both per-window: the monitor listened for IME insets on the Activity's window and read
  `Activity.currentFocus`, so a keyboard opened on a dialog (`AlertDialog`, Compose `Dialog`, a
  React Native `<Modal>`) went either unreported or reported as `Unknown` with no field name. Those
  windows' insets are now watched too, and focus is resolved from the top-most window that has it.
- An `ImageView` whose content changes in place — every React Native `<Image>` whose `source` is
  swapped, and any `DraweeView` — now replays the new image instead of the first one. The replay
  asset was keyed on the top-level drawable's identity, which Fresco creates once per view and
  mutates in place, so it was the same key for every image the view ever showed. The key is now
  derived from the drawable's nested content (its delegate / layer children).
- Images drawn through Fresco (every React Native `<Image>`, and any `DraweeView`) are no longer
  captured as blank boxes. A drawee view keeps one top-level drawable for its whole life and fills
  it asynchronously, so the replay asset — keyed on that drawable — was registered empty on the
  first walk and never revisited. Capture now retries a ref until its drawable actually paints,
  bounded so an image that never loads costs a few renders.
- WebView capture no longer records what the user types in the page. The injected rrweb recorder ran
  with the library's default input masking, which covers `<input type="password">` only, so a webview
  checkout shipped the card number, email and address into replay — while the native walkers have
  always masked a text field (they ship its placeholder, never its content). Every input value is now
  masked, and a click on a text field no longer reports that field's value as the click label (it fell
  back to `.value`, which is a custom control's visible caption but a field's typed text — wrappers
  such as `ion-input` nest a real `<input>` and leaked the same way). Buttons keep their caption.
- HTTP body capture is bounded again. `NetworkInterceptor`'s 64 KB cap was read from OkHttp's
  declared content length, which is `-1` for a chunked or transparently gunzipped body — most JSON
  APIs — so the cap never applied and the whole response was buffered in memory, once to capture it
  and again to build the replacement body handed back to the caller. Capture now peeks a bounded
  copy and leaves the original body unread, so a large download no longer risks the app's heap or a
  giant cached payload. Request bodies that declare no length are capped after reading, and a
  one-shot or duplex request body is skipped rather than consumed — reading one corrupted the
  outgoing request. `Network.responseBytes` now reports the body's exact size whenever it was
  captured whole, and is omitted when the body's length is genuinely unknown.
- Stack frames are parsed more accurately. The frame parser now uses the pattern the JavaScript SDKs
  use, which covers the JVM `at pkg.Class.method(File.kt:12)` shape as well: method names no longer
  carry a trailing space, and a throwable's leading message line is no longer turned into a bogus
  frame when it happens to contain a parenthesis. JavaScript stacks — those a WebView or a
  cross-platform shim reports — were the ones actually broken before, losing the file and line of
  every anonymous frame.
- Captured HTTP payloads and header values are now PII-redacted before they are cached or uploaded.
  `NetworkInterceptor` captures request/response bodies verbatim, so a login or checkout POST
  carried the plaintext password and card number. Core scrubs values under sensitive keys and
  card / email / SSN / SIN / phone patterns at the raw-cache boundary.
- Opening a page visit is now atomic. Events arrive on whatever thread produced them (OkHttp's
  dispatcher for network, the main thread for replay and taps, a shim's JS/platform thread for
  custom errors), and the page open below was a read-modify-write spanning two lock acquisitions —
  so concurrent callers could each mint a page for the same screen.
- Returning from the background no longer records into a closed page visit. Backgrounding finalizes
  every open page so it can flush, and a relaunch finds the previous run's page in the same state,
  but that page stayed *current* — so everything captured after the app resumed was filed under a
  page visit that had already been processed and sent, and after a session rotation the new
  session's events landed in the previous session's page. A finalized page now yields to a
  successor, which keeps the screen's name.
- The first `trackNavigation` no longer leaves an extra `view/<uuid>` page visit behind. Capture
  opens a nameless page as soon as the first event lands; a host that names its opening screen a
  moment later (React Native and Flutter do, once their async init resolves) split off it, shipping
  that placeholder as its own page visit. The first page name now adopts it instead.
- Replay no longer captures what a user types. Only password-variation inputs were masked, so an
  `EditText`'s content — an email, a shipping address, a card number in a plain field — was shipped
  in the replay tree *and* in the click-selector snapshot. An input now surfaces its hint (the label
  the user sees), or `***` when it is filled but unlabelled. Matches the iOS SDK.
- Custom attributes are no longer lost after a session rotation. The 10-attribute budget and the
  no-duplicate-names rule are documented as per session, but they only reset on SDK shutdown — so
  once the SDK rotated to a new session (a background stay of 15+ minutes), an app re-sending its
  customer id had it dropped as a duplicate and the new session carried no attributes at all.
- `installNetworkInstrumentation()` is now idempotent — a builder derived from an already
  instrumented client (`OkHttpClient.newBuilder()`) no longer captures every request twice.
- Session replay video now aligns with the session timeline — clicks, keyboard input, and
  navigation appear at the same moment in the replay as on the timeline.
- Replay video is attributed to the correct screen during rapid navigation between pages.
- Single-page sessions (no navigation) now reliably deliver session replay and events.
- Pending data is delivered more reliably after the app is backgrounded or terminated.

## [1.0.0-alpha06]

Initial public alpha.

- Session replay capture.
- RUM events: taps, navigation, and errors.
- Network monitoring.
- WebView capture.
- Custom attributes and custom errors.
- Privacy modes: `MASK_ALL`, `MASK_SENSITIVE`, `ALLOW_ALL`.
