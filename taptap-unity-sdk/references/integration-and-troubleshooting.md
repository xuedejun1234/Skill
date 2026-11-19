# TapTap Unity SDK: SanXiao Integration Notes

This file separates confirmed project facts from likely causes and unresolved incidents. It is a practical record, not a replacement for TapTap's current versioned documentation.

## Known project context

- Unity Editor shown in project history: `2022.3.62f3c1`.
- TapSDK Core, Login, and Compliance were installed from Git at `4.11.1` (Package Manager screenshots showed all three).
- Android Package Name / application ID shown in Player Settings: `com.tq.bfxyx`.
- The project has `Assets/Scripts/GameSDKManager.cs` and `Assets/Scripts/Login/LoginSceneController.cs`.
- The integration intent is TapTap login and compliance; the storefront is simulated, not a real payment processor. TapTap's compliance service is expected to enforce age-based purchase limits.

The official Unity Login guide documents Core + Login at 4.11.1. The Compliance guide documents Core + Login + Compliance at 4.11.1 and the Git package URLs. Re-check the selected version's docs before applying these versions to a different project.

## Incidents and resolutions

### 1. `TapSDK`, `TapTapLogin`, and Compliance symbols could not be resolved

**Observed:** The SDK wrapper had been written before the SDK packages were imported. Compile diagnostics listed unresolved names such as `TapSDK`, `TapTapLogin`, `TapTapSdkOptions`, and `TapTapComplianceOption`.

**Cause:** C# source references SDK assemblies that are not installed, or the package version/namespaces do not match the source.

**Resolution/check:** Install the official Core, Login, and (for age/compliance) Compliance packages through Package Manager using one consistent version. Wait for Unity's package import and script compilation to finish. Confirm all expected packages appear in Package Manager and resolve using the namespaces documented for that installed version. Do not treat IDE style inspections (naming, unused parameters, spelling) as compiler errors.

**Project outcome:** Package Manager later showed Core, Login, and Compliance 4.11.1 installed. If symbols remain unresolved after that, inspect the first Unity compiler error, asmdef references, duplicate/manual package copies, and package import/compilation state.

### 2. Build stopped because External Dependency Manager was missing

**Observed:** TapSDK's build precheck said it could not detect `com.google.external-dependency-manager`.

**Cause:** Android dependency resolution support required by the TapSDK integration was absent or not imported correctly.

**Resolution/check:** Install the External Dependency Manager for Unity (EDM4U) required by the selected TapSDK version. The official TapTap practice page currently lists `com.google.external-dependency-manager` 1.2.179 and NuGetForUnity alongside the SDK dependencies. Then run the Android Resolver's Force Resolve command from Unity's External Dependency Manager menu and inspect the resolver output. Avoid keeping overlapping copies of EDM4U in both Package Manager and `Assets/` unless the version's instructions require it.

If Unity asks whether to enable Android Gradle templates, decide based on the SDK/resolver setup and inspect the generated templates after enabling; do not assume that dialog itself fixes dependency resolution.

### 3. Google iOS Resolver failed to load `UnityEditor.iOS.Extensions.Xcode`

**Observed:** Unity warned that `Google.IOSResolver.dll` could not resolve `UnityEditor.iOS.Extensions.Xcode`.

**Meaning:** This is an Editor-side iOS resolver assembly/platform reference warning. By itself it does not prove that an Android player or an already-built iOS app cannot run. It can indicate missing iOS Build Support in the Unity installation, an incompatible plugin platform configuration, or opening an iOS-only resolver on a machine without the iOS tooling.

**Resolution/check:** If building iOS, install the matching Unity iOS Build Support module and use macOS with Xcode for the iOS build/sign process. Check the resolver DLL's Plugin Inspector platform settings and the full Console/compiler output. If the target is Android, confirm whether this warning is isolated; do not disable reference validation indiscriminately to hide a real iOS build problem.

### 4. Compliance API reported “SDK 正在初始化中，请稍后再试”

**Observed:** Stack trace went through `TapTapCompliance.RegisterComplianceCallback`, then `GameSDKManager.InitSDK()` (historically around line 113) and `LoginSceneController.Start()` (around line 31).

**Meaning:** A compliance API was reached while the installed SDK's Core initialization state was not ready. The stack trace identifies the call site, not the underlying reason. Possible causes to check include initialization not running on the actual device/platform, duplicate or racing initialization, a prior startup error, and calling an API before that SDK version's readiness point.

**Resolution/check:**

- Ensure `TapTapSDK.Init(...)` is called exactly once with the Core options and Compliance options before dependent calls.
- Compare call ordering with the exact installed 4.11.1 API/sample. The official guide's sample initializes and then registers the callback; if that still yields “initializing” on-device, inspect package source/logs for the supported readiness callback/state rather than inserting a fixed `WaitForSeconds` delay.
- Log initialization start/result and the first SDK error, but redact credentials.
- Ensure `LoginSceneController` does not trigger a second initialization or call `Startup`/`RegisterComplianceCallback` before the manager is ready.
- Confirm the package build includes the platform implementation and that the app credentials/config correspond to the selected app.

**Project outcome:** This issue was reported with a stack trace, but the conversation did not establish a verified final fix. Keep it marked unresolved until a device log shows initialization success followed by a compliance callback.

### 5. Login seemed successful but the game did not enter HomeScene

**Observed:** Login UI displayed “登录成功，正在进行身份验证…” and stayed there; another log showed the compliance initialization-state error.

**Likely flow issue:** The scene transition is gated by compliance success, but the callback that unlocks the flow is not arriving. A TapTap login result alone is not the same as a successful anti-addiction callback.

**Resolution/check:** Log the TapTap login result and every compliance `(code, extra)` callback. Only enter the game after the allowed callback (`500`) for the installed API's documented flow. Handle `1200` as app configuration/network failure with a retry path; handle time/age restrictions as a blocked state; handle exit/account switch by returning to login; handle `9002` as a user-closed real-name flow. Confirm the `HomeScene` is in Build Settings, but do not change scene wiring to mask a missing compliance callback.

### 6. Toast said “TapSDK: 包名、签名错误” even though login returned success

**Observed:** The app displayed the package/signature error; the user confirmed the intended package name was `com.tq.bfxyx`, and Player Settings showed that value.

**Meaning:** The Unity setting alone is not proof that the final installed APK uses the same application ID and signing certificate registered for the TapTap app. Login success also does not establish that compliance configuration is valid.

**Resolution/check:**

1. Check the final Gradle/merged manifest `applicationId` for the exact build variant, including any application-ID suffix.
2. Check whether the APK was signed with the expected debug or release keystore. Compare the APK's signing-certificate fingerprint with the certificate configured for the same TapTap app. Uninstall an old build if switching signing keys.
3. Confirm the Client ID/Token belong to that same TapTap application and that required login/compliance services and developer-center setup are enabled.
4. Rebuild and test the exact variant; inspect the SDK's detailed ErrLog, not just its toast.

The package value `com.tq.bfxyx` matched the user's intended value, but the full package/signature configuration was not independently verified in the conversation.

### 7. Android build hung during asset import or failed resolving `launcher:releaseRuntimeClasspath`

**Observed:** Unity showed long “Importing assets” / “Building Player” activity, then a Gradle error: `Could not resolve all files for configuration ':launcher:releaseRuntimeClasspath'`.

**Meaning:** The Gradle message is a dependency-resolution failure. Entering the keystore password only addresses signing when the failure is in a signing task; it does not resolve missing Maven artifacts or repository/network errors.

**Resolution/check:**

- Wait while Unity's import/build progress is changing; do not repeatedly force-close during active import. If it is genuinely stalled, inspect `Editor.log` and the generated Gradle build log for the first `Could not find`, HTTP, TLS, or repository error.
- Check network/proxy access to the repositories, EDM4U Force Resolve output, generated Gradle templates, Android SDK/NDK/JDK configuration, and dependency versions.
- Export a Gradle project or build with detailed logging only when needed to expose the first unresolved artifact. Fix that dependency/repository cause, then build again.

**Project outcome:** The user provided the generic Gradle configuration failure, but no full first-cause Gradle log or verified final resolution. Do not claim the keystore was the fix.

### 8. Package import takes a long time

**Observed:** Unity stayed on “Importing assets” for several minutes after package changes.

**Resolution/check:** Check whether the progress bar, CPU/disk activity, or `Editor.log` continues to advance. A large first import can take time. Avoid repeated package reimports or force-closing while the Asset Database is actively working; if progress is stationary, inspect the last Editor.log lines and Package Manager errors before retrying.

## Login and compliance flow reference

For this project shape, keep initialization/callback routing in `GameSDKManager`, and UI/scene routing in the login controller. The expected sequence is:

1. Initialize TapSDK with the target app's credentials and the Compliance options.
2. Register one persistent compliance callback.
3. Get or perform TapTap login.
4. Pass the signed-in TapTap account's `openId` to Compliance startup.
5. Wait for and process the compliance callback before entering gameplay.

Use the exact API spelling and async behavior from the package installed in the project. The official v4 Unity compliance guide lists `500` as allowed, `1000` as exit/invalid credential, `1001` as account switch, `1030` and `1050` as play-time restrictions, `1100` as age limit, `1200` as configuration/network error, and `9002` as the real-name flow being closed. Re-check the versioned docs if these codes or flows differ in a later SDK.

For the simulated store, keep the game-side purchase record separate from the compliance check. Check the age-based payment limit before granting currency. The current official Unity guide documents the compliance payment amount in fen and `SubmitPayment` only after a completed purchase; do not submit canceled or failed actions. Confirm whether the project's simulated purchase semantics should count as a completed consumption under the active TapTap compliance setup before wiring the report call.

## Official references

- [TapTap Unity Login SDK guide](https://developer.taptap.cn/docs/sdk/taptap-login/development-guide/unity/)
- [TapTap Unity Compliance SDK guide](https://developer.taptap.cn/docs/sdk/anti-addiction/development-guide/unity/)
- [TapTap anti-addiction integration practices](https://developer.taptap.cn/docs/sdk/anti-addiction/practice/)

Always re-check current versioned documentation for package dependencies, API signatures, platform support, callback behavior, and required developer-center setup before changing an active project.
