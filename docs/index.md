# SpinMyGP SDK — Partner Integration Guide

**SDK version:** `0.0.1`  
**Min Android SDK:** 21 (Android 5.0)  
**Kotlin:** 2.1+  
**Compose BOM:** 2024.09.00+

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
| Jetpack Compose | BOM 2024.09.00+ |
| `compileSdk` | 35+ |
| `minSdk` | 21 |

The SDK ships as an AAR and bundles all its own transitive dependencies (Compose Material3, Ktor, Coil, Lottie, Haze). You do not need to declare any of those yourself.

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
                username = providers.gradleProperty("WIN_JFROG_USERNAME").orNull
                    ?: System.getenv("WIN_JFROG_USERNAME")
                password = providers.gradleProperty("WIN_JFROG_PASSWORD").orNull
                    ?: System.getenv("WIN_JFROG_PASSWORD")
            }
        }
    }
}
```

Store credentials either in your project's `local.properties` (never commit this file):

```properties
# local.properties  — add to .gitignore
WIN_JFROG_USERNAME=your_username
WIN_JFROG_PASSWORD=your_password
```

…or as CI environment variables `WIN_JFROG_USERNAME` / `WIN_JFROG_PASSWORD`.

### 2b. Declare the dependency

In your **app module** `build.gradle.kts`:

```kotlin
dependencies {
    implementation("com.gakk.spin:mygp:0.0.1")
}
```

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

The flow has a **replay cache of 1**, so you will always receive the most recent event even if you subscribe slightly late.

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

By default the SDK scopes its internal `ViewModel` to `LocalViewModelStoreOwner.current` — usually
your host Activity, which is the right choice for almost every integration. Override it only when
you need the SDK's state to follow a narrower lifecycle, e.g. a navigation destination so it is
cleared automatically when the user navigates away:

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
| `accessToken` | `String`  | `""`           | A valid Bearer token used to authenticate all SDK API calls. The subscriber's msisdn and segment are extracted from the token server-side. Pass an empty string only for unauthenticated testing — all API calls will return `401`. |

### `onTokenRefresh` callback

Both `SpinSdkCore.show()` and `SpinAndWin` accept an optional `onTokenRefresh: suspend () -> String?` callback:

| Return value | SDK behaviour |
|---|---|
| Non-null `String` | SDK retries the failed request once with the new token. |
| `null` | SDK surfaces an auth-error state and emits `SpinEvent.AuthFailed`. |

If you do not supply `onTokenRefresh`, the default is `{ null }` — any 401 immediately results in `SpinEvent.AuthFailed`.

---

## 8. Event Reference

```kotlin
sealed interface SpinEvent
```

| Event | Payload | When fired |
|---|---|---|
| `SheetOpened` | — | Immediately after `SpinSdkCore.show()` is called. **Not fired** on the Compose path. |
| `SheetDismissed` | — | When the user swipes down or presses Back. **Not fired** on the Compose path. |
| `SpinTriggered` | — | The user tapped the Spin button; wheel animation begins. |
| `SpinCompleted` | `reward: Reward` | Wheel animation finished with a successful API response. |
| `SpinFailed` | `error: String` | API call failed or daily spin limit exceeded. |
| `AuthFailed` | — | A 401 was received and `onTokenRefresh` returned `null`. Redirect the user to your login screen. |
| `TabChanged` | `tab: SpinTab` | User switched to a different tab. |
| `GoDetailsClicked` | `rewardType: RewardType` | User tapped the "Go to Details" CTA on the result screen. |

### `Reward` properties

```kotlin
data class Reward(
    val productId  : Int,
    val type       : RewardType,
    val icon       : String,      // URL — SVG or raster
    val title      : String,      // Full localised display name
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
| `UNKNOWN` | Unrecognised type returned by the server. Treat as no prize. |
| `NOTHING` | The user did not win anything this spin. |

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

When `GoDetailsClicked` fires, the user expects to land on your in-app product details page:

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
is SpinEvent.SheetDismissed -> viewModel.refresh()
```

### Check if the user won (not just spun)

```kotlin
is SpinEvent.SpinCompleted -> {
    val reward = event.reward
    if (reward.type != RewardType.NOTHING && reward.type != RewardType.UNKNOWN) {
        celebrateWin(reward)
    }
}
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

**Q: Where do I report bugs or request features?**  
A: Contact the SpinMyGP SDK team at **mushfiq.gakk@gmail.com**.
