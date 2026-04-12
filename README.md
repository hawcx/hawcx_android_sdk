# Hawcx Android SDK V6.0.2

Hawcx delivers passwordless authentication for Android with an adaptive V6 flow for new integrations, while keeping existing V4/V5 APIs available for apps that are already in production.

[![Kotlin Version](https://img.shields.io/badge/Kotlin-2.0+-blue.svg)](https://kotlinlang.org)
[![Platform](https://img.shields.io/badge/platform-Android_8.0%2B-green.svg)](https://developer.android.com/about)
[![License](https://img.shields.io/badge/license-MIT-red.svg)](LICENSE)

## What This Release Includes

- V6 adaptive authentication via `buildProtocolV1Sdk()` and `HawcxV1SDK`
- Trusted-device handling and secure device credentials managed by the SDK
- Optional OAuth redirect and exchange helpers for V6 flows
- Existing `HawcxSDK` V4/V5 APIs for current integrations and incremental migration

## Requirements

- Android API 26+
- A Kotlin- or Java-based Android app

## Installation

Add the Hawcx Maven repository:

```gradle
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
        maven {
            url = uri("https://raw.githubusercontent.com/hawcx/hawcx_android_sdk/main/maven")
            metadataSources {
                mavenPom()
                artifact()
            }
        }
    }
}
```

Then add the dependency:

```gradle
dependencies {
    implementation "api.hawcx:hawcx:6.0.2"
}
```

Runtime dependencies such as OkHttp, Retrofit, Coroutines, Gson, and AndroidX helpers are declared in the published POM and resolve automatically.

If your project still bundles a local `hawcx-*.aar` or uses `flatDir { dirs("libs") }`, remove that and let Gradle resolve the SDK from Maven.

## V6 Quick Start

V6 is the public SDK release name. The current Android API surface uses `HawcxV1SDK` and related `HawcxV1*` types because the SDK is built on Hawcx protocol v1.

```kotlin
import com.hawcx.internal.HawcxSDK

val hawcxCore = HawcxSDK(
    context = applicationContext,
    projectApiKey = "<YOUR_CONFIG_ID>",
    baseUrl = "https://your-hawcx-host.example.com"
)

val hawcx = hawcxCore.buildProtocolV1Sdk(
    configId = "<YOUR_CONFIG_ID>",
    autoPollApprovals = true
)
```

### Initialization Notes

- `HawcxSDK` remains the Android entry point because it wires Android context, secure storage, device info, and trusted-device helpers used by the V6 flow.
- `buildProtocolV1Sdk(...)` returns the `HawcxV1SDK` wrapper that powers the V6 prompt-driven flow.
- `configId` is the Hawcx value provisioned for this integration. In current public releases, this is the same value you may receive as your project API key / Config ID.
- `baseUrl` should point to your Hawcx tenant host. Pass the tenant host root when possible. If you already have a `/v1` URL, the SDK normalizes it automatically. Do not append `/v1`, `/auth`, or `/hc_auth` yourself.
- `autoPollApprovals` defaults to `true`, which is the right choice for most apps.
- `relyingParty` is optional and should only be set when your backend expects the `X-Relying-Party` header.
- The recommended public integration path is `HawcxSDK(...).buildProtocolV1Sdk(...)`. If you instantiate `HawcxV1SDK` directly as an advanced setup, you must also provide the Android device-info and device-crypto wiring yourself.

## How V6 Works

1. Initialize `HawcxSDK` with your `configId` and tenant base URL.
2. Build the V6 wrapper with `buildProtocolV1Sdk()`.
3. Start a flow, usually `SIGNIN`, with the user's identifier.
4. Render the prompt Hawcx returns.
5. Send the user's input back to the SDK.
6. Exchange the resulting authorization code on your backend.

The SDK handles protocol requests, required headers, PKCE when needed, trusted-device storage, internal device-trust processing, and approval polling.

## Starting Authentication

`HawcxV1FlowType` supports:

- `SIGNIN`
- `SIGNUP`
- `ACCOUNT_MANAGE`

Most apps should start with `SIGNIN`:

```kotlin
import com.hawcx.protocol.v1.HawcxV1FlowType

hawcx.start(
    flowType = HawcxV1FlowType.SIGNIN,
    identifier = "user@example.com"
)
```

Additional fields are available when your tenant policy requires them:

```kotlin
hawcx.start(
    flowType = HawcxV1FlowType.SIGNIN,
    identifier = "user@example.com",
    startToken = "<optional-start-token>",
    inviteCode = "12345678"
)
```

`startToken` is optional and is only needed when your backend or hosted flow gives your app a pre-issued token to resume or continue a specific start path.

If you omit `codeChallenge`, the SDK generates PKCE automatically and returns `codeVerifier` in the completed update.

## Handling Flow Updates

Use `flow.onUpdate` to drive your UI:

```kotlin
import com.hawcx.protocol.v1.HawcxV1FlowUpdate
import com.hawcx.protocol.v1.HawcxV1UserPrompt

hawcx.flow.onUpdate = { update ->
    when (update) {
        is HawcxV1FlowUpdate.Loading -> {
            // Show progress.
        }

        is HawcxV1FlowUpdate.Prompt -> when (val prompt = update.prompt) {
            is HawcxV1UserPrompt.SelectMethod -> {
                print(prompt.methods.map { it.id })
            }

            is HawcxV1UserPrompt.EnterCode -> {
                print(prompt.destination)
            }

            is HawcxV1UserPrompt.SetupSms -> {
                print(prompt.existingPhone ?: "")
            }

            is HawcxV1UserPrompt.SetupTotp -> {
                print(prompt.secret)
                print(prompt.otpauthUrl)
            }

            is HawcxV1UserPrompt.EnterTotp -> {
                // Render authenticator-code UI.
            }

            is HawcxV1UserPrompt.Redirect -> {
                print(prompt.url)
                print(prompt.returnScheme ?: "")
            }

            is HawcxV1UserPrompt.AwaitApproval -> {
                print(prompt.expiresAt)
                print(prompt.pollInterval)
            }
        }

        is HawcxV1FlowUpdate.Completed -> {
            print(update.session)
            print(update.authCode)
            print(update.expiresAt)
            print(update.codeVerifier ?: "")
        }

        is HawcxV1FlowUpdate.Error -> {
            print(update.code)
            print(update.message)
            print(update.meta?.traceId ?: "")
        }

        HawcxV1FlowUpdate.Idle -> Unit
    }
}
```

Internal device-trust prompts are handled by the SDK automatically and should not be rendered in your UI.

## Redirect Handling

If the backend returns a `Redirect` prompt, open the provided URL in a browser or custom tab, register a callback intent filter, parse the callback, and resume the Hawcx flow.

```xml
<activity android:name=".MainActivity" android:exported="true">
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />

        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />

        <data android:scheme="yourapp" />
    </intent-filter>
</activity>
```

```kotlin
import android.content.Intent
import android.net.Uri
import androidx.fragment.app.FragmentActivity
import com.hawcx.protocol.v1.HawcxV1OAuthCallbackParser

class MainActivity : FragmentActivity() {
    private lateinit var hawcx: com.hawcx.protocol.v1.HawcxV1SDK

    private fun openRedirect(url: String) {
        startActivity(Intent(Intent.ACTION_VIEW, Uri.parse(url)))
    }

    override fun onNewIntent(intent: Intent) {
        super.onNewIntent(intent)

        if (intent.action != Intent.ACTION_VIEW) return

        val callback = HawcxV1OAuthCallbackParser.parse(intent.dataString) ?: return
        hawcx.flow.oauthCallback(callback.code, callback.state)
    }
}
```

## Backend Exchange

When the flow completes, the SDK returns:

- `session`
- `authCode`
- `expiresAt`
- `codeVerifier` when PKCE was generated by the SDK
- `meta.traceId` for support and correlation

For most apps, send `authCode` and `codeVerifier` to your backend immediately over HTTPS and perform the exchange there. Keep OAuth client credentials and token verification on the server, not in the Android app.

Include `codeVerifier` whenever the SDK returns it. If your app supplied a custom `codeChallenge`, send the matching verifier from that flow to your backend as part of the exchange.

Your backend should:

1. exchange `authCode` and `codeVerifier` with the Hawcx backend SDK
2. verify the returned claims
3. create your app session or tokens
4. return the app auth result your Android app needs

If you also want the SDK to persist backend-issued tokens for existing utilities, you can store them with:

```kotlin
hawcxCore.storeBackendOAuthTokens(
    accessToken = accessToken,
    refreshToken = refreshToken,
    userId = "user@example.com"
)
```

If your app truly needs an SDK-managed exchange path, `HawcxV1OAuthExchangeService` is available as an advanced option. When configured with a verifier or a non-empty `publicKeyPem`, it can verify the returned ID token and returns a `HawcxV1OAuthToken` (`idToken`, `tokenType`, `expiresIn`) instead of app-specific access / refresh tokens.

For backend implementation examples, see:

- [Node.js backend quickstart](https://docs.hawcx.com/docs/v1/sdk-reference/backend/nodejs/quickstart)
- [Python backend quickstart](https://docs.hawcx.com/docs/v1/sdk-reference/backend/python/quickstart)

## Trusted Devices and Secure Storage

The SDK manages trusted-device material for you:

- device-trust steps are handled internally
- device credentials are stored per app, per user, and per Hawcx project / Config ID
- secure storage is isolated so the same user in different apps does not share credentials
- invalid local trusted-device records are cleared and re-enrolled automatically when needed

That automatic trusted-device wiring is part of the recommended `HawcxSDK(...).buildProtocolV1Sdk(...)` path. If you instantiate `HawcxV1SDK` directly, you must provide the Android device-info and device-crypto pieces yourself for trusted-device steps to behave the same way.

## Existing V4/V5 APIs

This package still includes `HawcxSDK` for apps that already use the existing V4/V5 integration surface, including:

- V4/V5 authentication flows
- push approval handling
- device session APIs
- existing token-storage helpers

That makes incremental migration possible: new V6 flows can live on `HawcxV1SDK` while existing utilities remain on `HawcxSDK`.

## Documentation

- [V6 Android guide](https://docs.hawcx.com/docs/v1/sdk-reference/frontend/android/sdk-v6)
- [V5 Android guide](https://docs.hawcx.com/docs/v1/sdk-reference/frontend/android/sdk)
- [Node.js backend quickstart](https://docs.hawcx.com/docs/v1/sdk-reference/backend/nodejs/quickstart)
- [Python backend quickstart](https://docs.hawcx.com/docs/v1/sdk-reference/backend/python/quickstart)
- [Documentation home](https://docs.hawcx.com)

## Support

- [Website](https://www.hawcx.com)
- [Support Email](mailto:info@hawcx.com)
