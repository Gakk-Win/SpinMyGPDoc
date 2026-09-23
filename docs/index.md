# SpinMyGP SDK — Partner Integration Guide

**SDK version:** `0.0.5`  
**Min Android SDK:** 21 (Android 5.0)  
**Kotlin:** 2.1+  
**Compose BOM:** 2026.04.01+

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Add the Dependency](#2-add-the-dependency)
3. [Internet Permission](#3-internet-permission)
4. [Observing Events](#4-observing-events)
5. [Integration Path A — View-based Host (Activity / Fragment)](#5-integration-path-a-view-based-host-activity-fragment)
6. [Integration Path B — Pure Compose Host](#6-integration-path-b-pure-compose-host)
7. [Configuration Reference](#7-configuration-reference)
8. [Event Reference](#8-event-reference)
9. [Reward & Tab Types](#9-reward-tab-types)
10. [ProGuard / R8](#10-proguard-r8)
11. [Common Patterns](#11-common-patterns)
12. [FAQ](#12-faq)

---

## 1. Prerequisites

| Requirement | Minimum |
|---|---|
| Android Gradle Plugin | 8.0+ |
| Kotlin | 2.1.0+ |
| Jetpack Compose | BOM 2026.04.01+ |
| `compileSdk` | 35+ |
| `minSdk` | 21 |

The SDK ships as an AAR whose POM declares all of its own dependencies (Compose Material3, Material Components, Ktor, Coil, Lottie, Haze, DataStore). Gradle pulls them in transitively, so you do not need to declare any of those yourself.

---

## 2. Add the Dependency

### 2a. Add the Maven repository

In your **project-level** `settings.gradle.kts` (or `build.gradle.kts`), add the private repository inside `dependencyResolutionManagement`:

```kotlin
// settings.gradle.kts
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()

        // SpinMyGP SDK repository
        maven {
            url = uri("https://jfrog.deenislamic.com/artifactory/win")
            credentials {
                username = "" // Keep it empty
                password = providers.gradleProperty("WIN_JFROG_TOKEN").orNull
                    ?: System.getenv("WIN_JFROG_TOKEN")
            }
        }
    }
}
```

> **Important:**  The Token will be provided from Gakk business team.


Store credentials either in your project's `local.properties` (never commit this file):

```properties
# local.properties  — add to .gitignore
WIN_JFROG_TOKEN=the_token_provided_from_gakk
```

…or as CI environment variables `WIN_JFROG_TOKEN`.

### 2b. Declare the dependency

In your **app module** `build.gradle.kts`:

```kotlin
dependencies {
    implementation("com.gakk.spin:mygp:0.0.5-dev")
}
```

> **Dev and production builds:** SDK versions ending in `-dev` connect to the dev server. They also show a small test label at the top of the sheet with the subscriber's MSISDN and segment, read from the access token. Use them only in test builds of your app. Production versions have no suffix and no label.

Sync Gradle — the SDK is now available.

---

## 3. Internet Permission

The SDK makes network calls. Ensure your `AndroidManifest.xml` declares internet permission (most apps already have this):

```xml
<uses-permission android:name="android.permission.INTERNET" />
```

---

## 4. Observing Events

Before you show the sheet, set up an event observer so you can react to what happens inside it. The SDK surfaces all interactions through a single `SharedFlow`:

```kotlin
SpinSdkCore.events: SharedFlow<SpinEvent>
```

### Recommended — observe in an Activity

```kotlin
class MainActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        lifecycleScope.launch {
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                SpinSdkCore.events.collect { event ->
                    handleSpinEvent(event)
                }
            }
        }
    }

    private fun handleSpinEvent(event: SpinEvent) {
        when (event) {
            is SpinEvent.SheetOpened         -> { /* sheet is now visible */ }
            is SpinEvent.SheetDismissed      -> { /* sheet is closed */ }
            is SpinEvent.SpinTriggered       -> { /* wheel is spinning */ }
            is SpinEvent.SpinCompleted       -> showReward(event.reward)
            is SpinEvent.SpinFailed          -> showError(event.error)
            is SpinEvent.AuthFailed          -> redirectToLogin()
            is SpinEvent.TabChanged          -> trackTabChange(event.tab)
            is SpinEvent.GoDetailsClicked    -> openRewardDetails(event.rewardType)
            is SpinEvent.NotificationScheduleRequested ->
                scheduleSpinReminder(event.nextSpinAtEpochMs)
        }
    }
}
```

### In a ViewModel

```kotlin
class HomeViewModel : ViewModel() {

    init {
        viewModelScope.launch {
            SpinSdkCore.events.collect { event ->
                when (event) {
                    is SpinEvent.SpinCompleted -> _reward.value = event.reward
                    is SpinEvent.AuthFailed    -> _navigateToLogin.emit(Unit)
                    else -> Unit
                }
            }
        }
    }
}
```

---

## 5. Integration Path A — View-based Host (Activity / Fragment)

Use `SpinSdkCore.show()` when your host screen is a classic View-based Activity or Fragment. The SDK wraps the spin UI inside a native `BottomSheetDialog` automatically.

```kotlin
SpinSdkCore.show(context, config, onTokenRefresh)
```

- Calling `show()` while an SDK sheet is already showing does nothing, so repeated taps on your button cannot stack dialogs.
- Each call creates a fresh sheet. Data is re-fetched and `config.initialTab` is applied on every open.

### Minimal example

```kotlin
// In an Activity
binding.spinButton.setOnClickListener {
    SpinSdkCore.show(
        context = this,
        config  = SpinConfig(accessToken = authRepo.currentToken),
    )
}
```

### Programmatic Dismissal (Crucial)

To prevent "WindowLeaked" crashes, you **must** dismiss the SDK sheet (View based dialog) if your Activity is destroyed while the sheet is still open.

```kotlin
override fun onPause() {
    super.onPause()
    // Optional: dismiss if you want to close the wheel when the user leaves the screen
    // SpinSdkCore.dismiss()
}

override fun onDestroy() {
    // Strongly Recommended: prevent window leak crashes
    SpinSdkCore.dismiss()
    super.onDestroy()
}
```

`SpinSdkCore.dismiss()` is safe to call when no sheet is showing. Dismissing a visible sheet emits `SpinEvent.SheetDismissed`, just like a user-initiated close. `dismiss()` only affects sheets opened with `SpinSdkCore.show()`. On the Compose path, set your `visible` flag to `false` instead.

### With all options

```kotlin
SpinSdkCore.show(
    context        = this,
    config         = SpinConfig(
        accessToken = authRepo.currentToken,
        initialTab  = SpinTab.Details,
    ),
    onTokenRefresh = { authRepo.refreshToken() }, // return null if refresh is not possible
)
```

> **Important:** Pass an `Activity` (or a theme-compatible context) — passing an `Application`
> context will cause a window-token crash when the dialog tries to attach.

### Full Activity example

```kotlin
class MainActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        observeSpinEvents()

        findViewById<Button>(R.id.btnSpin).setOnClickListener {
            SpinSdkCore.show(
                context        = this,
                config         = SpinConfig(accessToken = authRepo.currentToken),
                onTokenRefresh = { authRepo.refreshToken() },
            )
        }
    }

    private fun observeSpinEvents() {
        lifecycleScope.launch {
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                SpinSdkCore.events.collect { event ->
                    when (event) {
                        is SpinEvent.SpinCompleted -> {
                            Toast.makeText(
                                this@MainActivity,
                                "You won: ${event.reward.title}",
                                Toast.LENGTH_SHORT,
                            ).show()
                        }
                        is SpinEvent.SpinFailed -> {
                            Toast.makeText(this@MainActivity, event.error, Toast.LENGTH_SHORT).show()
                        }
                        is SpinEvent.AuthFailed -> {
                            startActivity(Intent(this@MainActivity, LoginActivity::class.java))
                        }
                        else -> Unit
                    }
                }
            }
        }
    }
}
```

---

## 6. Integration Path B — Pure Compose Host

Use the `SpinAndWin` composable when your host screen is already written in Jetpack Compose. The SDK renders as a `ModalBottomSheet` inside your existing composition.

> **Note:** `SpinSdkCore.events` still fires all events from this path **except**
> `SpinEvent.SheetOpened` and `SpinEvent.SheetDismissed`. Those two lifecycle events are owned
> by your composable via `onDismiss`.

### Step 1 — hoist a visibility flag

```kotlin
var showSpin by rememberSaveable { mutableStateOf(false) }
```

### Step 2 — place the composable

```kotlin
SpinAndWin(
    visible        = showSpin,
    onDismiss      = { showSpin = false },
    config         = SpinConfig(accessToken = authRepo.currentToken),
    onTokenRefresh = { authRepo.refreshToken() },
)
```

### Step 3 — trigger show

```kotlin
Button(onClick = { showSpin = true }) {
    Text("Spin the wheel")
}
```

The sheet plays its slide-down animation and leaves composition automatically when `showSpin`
becomes `false` — you only own the boolean.

Unlike `SpinSdkCore.show()`, the SDK's state on this path lives as long as `viewModelStoreOwner`
(see [below](#advanced-scoping-the-sdks-state-with-viewmodelstoreowner)). Loaded tabs, the selected
tab, and a result screen the user closed on all carry over when you open the sheet again, and survive
rotation. `config.initialTab` applies only when that state is first created: on the first open, or
after `accessToken` changes.

The SDK calls `onDismiss` whenever the sheet should close: swipe-down, scrim tap, Back, the close
button, and the **View Details** button on the result screen. Always set your flag to `false` there.

> **Keep `accessToken` stable.** Each distinct `SpinConfig.accessToken` value makes the SDK start over
> with a new HTTP client and fresh internal state. Don't build a new token during composition, e.g.
> `SpinConfig(accessToken = generateToken())` inside a composable. Keep the token in a stable holder
> such as your `ViewModel` or auth repository, and let it change only when it has actually been
> refreshed.

### Full composable screen example

```kotlin
@Composable
fun HomeScreen(viewModel: HomeViewModel = viewModel()) {
    var showSpin by rememberSaveable { mutableStateOf(false) }

    // Observe events
    LaunchedEffect(Unit) {
        SpinSdkCore.events.collect { event ->
            when (event) {
                is SpinEvent.SpinCompleted -> { /* handle reward */ }
                is SpinEvent.SpinFailed    -> { /* handle error  */ }
                is SpinEvent.AuthFailed    -> { /* redirect to login */ }
                else -> Unit
            }
        }
    }

    // Your screen content
    Column(
        modifier            = Modifier.fillMaxSize(),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center,
    ) {
        Button(onClick = { showSpin = true }) {
            Text("Play Spin & Win")
        }
    }

    // SDK sheet — place outside your main layout so it overlays correctly
    SpinAndWin(
        visible        = showSpin,
        onDismiss      = { showSpin = false },
        config         = SpinConfig(accessToken = viewModel.accessToken),
        onTokenRefresh = { viewModel.refreshToken() },
    )
}
```

### Controlling visibility from a ViewModel

```kotlin
// ViewModel
class HomeViewModel : ViewModel() {
    private val _showSpin = MutableStateFlow(false)
    val showSpin: StateFlow<Boolean> = _showSpin.asStateFlow()

    fun onSpinClicked() { _showSpin.value = true }
    fun onSpinDismissed() { _showSpin.value = false }

    suspend fun refreshToken(): String? = authRepo.refreshToken()
}

// Composable
@Composable
fun HomeScreen(viewModel: HomeViewModel = viewModel()) {
    val showSpin by viewModel.showSpin.collectAsStateWithLifecycle()

    SpinAndWin(
        visible        = showSpin,
        onDismiss      = { viewModel.onSpinDismissed() },
        config         = SpinConfig(accessToken = viewModel.accessToken),
        onTokenRefresh = { viewModel.refreshToken() },
    )
}
```

### (Advanced) Scoping the SDK's state with `viewModelStoreOwner`

By default the SDK scopes its internal `ViewModel`, and the network client it owns, to
`LocalViewModelStoreOwner.current`. That is usually your host Activity, which is the right choice
for almost every integration. Override it only when you need the SDK's state to follow a narrower
lifecycle, e.g. a navigation destination so it is cleared automatically when the user navigates
away:

```kotlin
val navBackStackEntry = navController.currentBackStackEntryAsState().value

SpinAndWin(
    visible             = showSpin,
    onDismiss           = { showSpin = false },
    config              = SpinConfig(accessToken = token),
    viewModelStoreOwner = navBackStackEntry ?: LocalViewModelStoreOwner.current!!,
)
```

> **Caveat:** The owner you pass must outlive the sheet. An owner that is cleared while `visible`
> is `true` will drop the SDK's in-progress state mid-session. When in doubt, omit the parameter
> and let it default.

---

## 7. Configuration Reference

```kotlin
data class SpinConfig(
    val initialTab  : SpinTab = SpinTab.Spin,
    val accessToken : String  = "",
)
```

| Parameter     | Type      | Default        | Description |
|---------------|-----------|----------------|-------------|
| `initialTab`  | `SpinTab` | `SpinTab.Spin` | The tab pre-selected when the sheet opens. |
| `accessToken` | `String`  | `""`           | A valid Bearer token used to authenticate all SDK API calls. The subscriber's msisdn and segment are extracted from the token server-side. The token payload should contain an `msisdn` claim (see [FAQ](#12-faq)). Pass an empty string only for unauthenticated testing — all API calls will return `401`. |

### `onTokenRefresh` callback

Both `SpinSdkCore.show()` and `SpinAndWin` accept an optional `onTokenRefresh: suspend () -> String?` callback:

| Return value | SDK behaviour |
|---|---|
| Non-null `String` | SDK retries the failed request once with the new token, and uses it for later requests in the same session. If the retry also returns `401`, `SpinEvent.AuthFailed` is emitted. |
| `null` | SDK surfaces an auth-error state and emits `SpinEvent.AuthFailed`. |

`onTokenRefresh` is a `suspend` function, so you can call your refresh endpoint directly inside it. Wrap blocking calls in `withContext(Dispatchers.IO)`. The SDK may call it for any request, including loading the wheel, details, or history, not only for spins.

If you do not supply `onTokenRefresh`, the default is `{ null }` — any 401 immediately results in `SpinEvent.AuthFailed`.

---

## 8. Event Reference

```kotlin
sealed interface SpinEvent
```

| Event | Payload | When fired |
|---|---|---|
| `SheetOpened` | — | The sheet opened by `SpinSdkCore.show()` appears on screen. **Not fired** on the Compose path. |
| `SheetDismissed` | — | The sheet opened by `SpinSdkCore.show()` closes for any reason: swipe-down, Back, close button, the **View Details** button, or `SpinSdkCore.dismiss()`. **Not fired** on the Compose path. |
| `SpinTriggered` | — | The user tapped the Spin button. Fired before the spin request is sent; the wheel starts turning once the server responds. |
| `SpinCompleted` | `reward: Reward` | The wheel stopped on the prize returned by the server. Also fired for "no prize" results (`RewardType.NOTHING`). |
| `SpinFailed` | `error: String` | The spin could not be completed: network or server error, daily spin limit reached, or an invalid server response. `error` is a localised, user-facing message, not a stable error code. |
| `AuthFailed` | — | Any SDK request got a `401` that could not be recovered: `onTokenRefresh` returned `null` or the retry also failed. Redirect the user to your login screen. |
| `TabChanged` | `tab: SpinTab` | The user selected a tab. Not fired for `initialTab` when the sheet opens. |
| `GoDetailsClicked` | `rewardType: RewardType` | The user tapped **View Details** on the result screen. The button is shown only for winning results (any type except `NOTHING`). The SDK closes the sheet right after this event. |
| `NotificationScheduleRequested` | `nextSpinAtEpochMs: Long` | The user opted into spin reminders (**Notify Me** → **Turn on Notification**), and again after every later spin while the opt-in is still active, because the next-spin time has moved. `nextSpinAtEpochMs` is wall-clock UTC milliseconds and is always in the future. **You** schedule and post the reminder — see [Scheduling the reminder notification](#scheduling-the-reminder-notification). |

### `Reward` properties

```kotlin
data class Reward(
    val productId  : Int,
    val type       : RewardType,
    val icon       : String,      // URL — SVG or raster; may be empty
    val title      : String,      // Display name as configured on the server
)
```

---

## 9. Reward & Tab Types

### `RewardType`

| Value | Meaning |
|---|---|
| `PHYSICAL` | A physical, deliverable item (e.g. merchandise). |
| `INTERNET` | A mobile-internet data bundle. |
| `GP_POINTS` | Grameenphone loyalty points. |
| `TALKTIME` | Mobile airtime credit. |
| `VOUCHER` | A gift voucher. |
| `UNKNOWN` | A prize whose type this SDK version does not recognise, typically a type added on the server after your SDK release. The user still won it, so handle it generically, e.g. by showing `reward.title`. |
| `NOTHING` | The user did not win anything this spin. |

> Every type except `NOTHING` is a win. The SDK's result screen shows "Congratulations" and the
> **View Details** button for all of them, including `UNKNOWN`.

### `SpinTab`

| Value | Description |
|---|---|
| `Spin` | Main spinning wheel — default opening tab. |
| `Details` | Terms & conditions and reward catalogue. |
| `History` | User's past spin results. |

---

## 10. ProGuard / R8

The SDK ships a `consumer-rules.pro` file that is automatically merged into your app's ProGuard configuration. **No manual rules are required.**

If you observe obfuscation issues in release builds, ensure `minifyEnabled = true` is paired with the `proguard-android-optimize.txt` default rules in your app module.

---

## 11. Common Patterns

### Deep-link into reward details

When `GoDetailsClicked` fires, the user expects to land on your in-app product details page. The SDK closes its sheet automatically, so you only need to navigate:

```kotlin
is SpinEvent.GoDetailsClicked -> {
    when (event.rewardType) {
        RewardType.INTERNET   -> navController.navigate("internet_bundles")
        RewardType.GP_POINTS  -> navController.navigate("gp_points_info")
        RewardType.TALKTIME   -> navController.navigate("recharge")
        RewardType.PHYSICAL   -> navController.navigate("physical_prizes")
        RewardType.VOUCHER    -> navController.navigate("vouchers")
        else -> Unit
    }
}
```

### Scheduling the reminder notification

The sheet offers the user a **Notify Me** reminder while the spin is on cooldown, but the SDK never
posts a notification and never requests `POST_NOTIFICATIONS`. It only tells you *that* the user
opted in and *when* the next spin unlocks — scheduling, the notification channel, and the runtime
permission are yours.

```kotlin
is SpinEvent.NotificationScheduleRequested -> scheduleSpinReminder(event.nextSpinAtEpochMs)
```

Schedule it however your app already schedules work — an `AlarmManager` alarm at `RTC_WAKEUP`, or
`WorkManager` with `initialDelay = nextSpinAtEpochMs - System.currentTimeMillis()` (alarms do not
survive a reboot; `WorkManager` restores its own work). Four things to keep in mind:

- **The event repeats**, at opt-in and again after each later spin while the opt-in is still on,
  because the next-spin time has moved. Schedule idempotently — a stable `PendingIntent` request
  code, or a fixed unique work name — so the new reminder replaces the pending one.
- **Request `POST_NOTIFICATIONS` yourself** on API 33+. Handling this event is a natural moment to
  ask, since the user just opted in.
- **The timestamp is wall-clock UTC milliseconds**, always in the future when the event is emitted.
  The SDK does not emit while a spin is already available, because there would be nothing to
  schedule.
- **The SDK remembers the opt-in, not your schedule.** The opt-in is stored per subscriber on the
  device, so the sheet keeps showing "We'll notify you" even if your reminder was never scheduled
  or was later cancelled.

### Open directly to the History tab

```kotlin
SpinSdkCore.show(
    context = this,
    config  = SpinConfig(
        accessToken = authRepo.currentToken,
        initialTab  = SpinTab.History,
    ),
)
```

### Refresh your UI after the sheet closes

```kotlin
// View-based path (SpinSdkCore.show)
is SpinEvent.SheetDismissed -> viewModel.refresh()

// Compose path (SpinAndWin) — SheetDismissed is not emitted, use onDismiss instead
SpinAndWin(
    visible   = showSpin,
    onDismiss = { showSpin = false; viewModel.refresh() },
    config    = SpinConfig(accessToken = token),
)
```

### Handle token expiry gracefully

```kotlin
SpinSdkCore.show(
    context        = this,
    config         = SpinConfig(accessToken = authRepo.currentToken),
    onTokenRefresh = {
        // Called on 401 — attempt a refresh and return the new token,
        // or null to trigger SpinEvent.AuthFailed
        authRepo.refreshToken()
    },
)

// In your event handler:
is SpinEvent.AuthFailed -> startActivity(Intent(this, LoginActivity::class.java))
```

---

## 12. FAQ

**Q: What do I pass as `accessToken`?**  
A: A valid JWT Bearer token from your authentication layer. The SDK attaches it as `Authorization: Bearer <token>` on every API call. The server extracts the subscriber's msisdn and segment from the token payload.

The SDK also reads the `msisdn` claim from the token on the device, without verifying it. It uses the claim to keep the sound setting, notification opt-in, and next-spin countdown separate for each subscriber, which matters on shared devices. Make sure your tokens include an `msisdn` claim.

**Q: Can I show the SDK from a Fragment?**  
A: Yes. Pass `requireActivity()` as the context — not `requireContext()` — to ensure the dialog has a valid window token.

```kotlin
SpinSdkCore.show(
    context        = requireActivity(),
    config         = SpinConfig(accessToken = authRepo.currentToken),
    onTokenRefresh = { authRepo.refreshToken() },
)
```

**Q: The sheet doesn't open a second time on the Compose path. What's wrong?**  
A: Ensure your `onDismiss` sets your `visible` flag back to `false`. If it stays `true`, the SDK still considers the sheet open, so setting it `true` again has no effect.

```kotlin
SpinAndWin(
    visible   = showSpin,
    onDismiss = { showSpin = false }, // required
)
```

**Q: Can I observe events from multiple screens simultaneously?**  
A: Yes. `SpinSdkCore.events` is a `SharedFlow` and supports multiple concurrent collectors. Each collector receives every event independently.

**Q: What happens if `onTokenRefresh` returns `null`?**  
A: The SDK surfaces an in-sheet auth-error state and emits `SpinEvent.AuthFailed`. Listen for this event and redirect the user to your login screen.

**Q: Does the SDK request notification permission or send push notifications?**  
A: No. The "Notify Me" / "Turn on Notification" prompt inside the sheet records the user's opt-in locally and emits `SpinEvent.NotificationScheduleRequested(nextSpinAtEpochMs)`. The SDK does not request `POST_NOTIFICATIONS` and does not schedule or post any notifications — your app schedules the reminder from that event. See [Scheduling the reminder notification](#scheduling-the-reminder-notification).

**Q: I upgraded to `0.0.4` and my `when (event)` no longer compiles. Why?**  
A: `0.0.4` adds `SpinEvent.NotificationScheduleRequested` to the sealed `SpinEvent` interface, so an exhaustive `when` without an `else` branch now has an unhandled case. Either handle the new event — see [Scheduling the reminder notification](#scheduling-the-reminder-notification) — or add `else -> Unit`. This is a source-level change only; nothing about the existing events changed.

**Q: Can I change the UI language?**  
A: All SDK UI strings are Android string resources prefixed with `spin_sdk_`. You can override any of them by declaring a string with the same name in your app's `res/values*/strings.xml`. Server-provided text, such as reward names and terms, is shown as returned by the API.

**Q: Where do I report bugs or request features?**  
A: Contact the SpinMyGP SDK team at **ahsan@cloud7bd.com**.
