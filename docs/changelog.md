# Changelog

All notable changes to the SpinMyGP SDK will be documented in this file.

### 0.0.5 (2026-09-21)
- Remove the `replay = 1` cache from the public `events` shared flow.
- Make `accessToken` a required `SpinConfig` parameter without a default value.
- Migrate list-based UI states and models to `ImmutableList` for improved stability and performance.
- Update the wheel API endpoint from `GetWheelByMsisdn` to `GetWheelByToken` and refine the fallback server error response structure.
- Some minor UI issues fixed according to the feedback

### 0.0.4-dev (2026-09-17)
- Add `SpinEvent.NotificationScheduleRequested(nextSpinAtEpochMs)`. It is emitted when the user opts into spin reminders, and again after each later spin while the opt-in is on, so your app can schedule the reminder. The SDK does not post notifications itself. An exhaustive `when (event)` needs a new branch or an `else`.
- Report `VOUCHER` prizes as `RewardType.VOUCHER`. Earlier versions reported them as `UNKNOWN`.
- `RewardType.UNKNOWN` is a win: a prize whose type this SDK version does not recognise.
- Fix the Compose path (`SpinAndWin`) failing every request with "Something went wrong" after the sheet was closed and reopened, or after a rotation.
- Versions ending in `-dev` connect to the dev server and show a test label with the subscriber's MSISDN and segment. Production versions have no suffix and no label.
- Clarify event timing, `show()` and `dismiss()` behaviour, the event replay cache, and keeping `accessToken` stable on the Compose path.

### 0.0.3 (2026-09-13)
- Downgrade haze from 1.6.10 to 1.6.0

### 0.0.2 (2026-09-13)
- Downgrade the compile SDK to 35 and update related dependencies to ensure compatibility with SDK 35.

### 0.0.1 (2026-06-18)
- Initial release of the SpinMyGP SDK.
- Support for View-based and Compose-based integrations.
- SharedFlow event system for tracking user interactions.
- Built-in authentication refresh mechanism.
