# Hawcx Android SDK V5

Hawcx provides enterprise-grade passwordless authentication for Android applications, delivering a secure and frictionless login experience across all user devices with unified V5 authentication that mirrors the latest iOS bridge.

[![Kotlin Version](https://img.shields.io/badge/Kotlin-2.0+-blue.svg)](https://kotlinlang.org)
[![Platform](https://img.shields.io/badge/platform-Android_7.0+-green.svg)](https://developer.android.com/about)
[![License](https://img.shields.io/badge/license-MIT-red.svg)](LICENSE)

## What's New in V5

- **Hardware-backed security** – provisioning now derives Ed25519 keys with HKDF and stores secrets via the Android Keystore.
- **OAuth-backed login** – one flow (`authenticateV5`/`submitOtpV5`) produces OAuth JWTs that align with the Hawcx server stack.
- **Unified cross-platform API** – method signatures and callbacks match the iOS SDK, easing multi-platform integrations.

## Installation

### 1. Add the Hawcx Maven repository

```gradle
repositories {
    maven {
        url = uri("https://raw.githubusercontent.com/hawcx/hawcx_android_sdk/main/maven")
        // For private access, configure a GitHub token with read permissions.
        credentials {
            username = System.getenv("GITHUB_USER") ?: "<github-username>"
            password = System.getenv("GITHUB_TOKEN") ?: "<github-token>"
        }
        metadataSources {
            mavenPom()
            artifact()
        }
    }
    google()
    mavenCentral()
}
```

> **Private repo tip:** when using a GitHub personal access token, supply it as the password and any non-empty string (or your username) as the user.

### 2. Depend on the SDK

```gradle
dependencies {
    implementation "api.hawcx:hawcx:5.1.1"
}
```

Runtime dependencies (Retrofit, OkHttp, Coroutines, Biometric, BouncyCastle, Play Services, etc.) are declared in the published POM and are pulled in automatically.

## Getting Started (V5)

```kotlin
val oauthConfig = HawcxOAuthConfig(
    tokenEndpoint = "https://dev-api.hawcx.com/hc_auth/v5/oauth/token",
    clientId = "hawcx-mobile",
    publicKeyPem = """
        -----BEGIN PUBLIC KEY-----
        ...
        -----END PUBLIC KEY-----
    """.trimIndent()
)

val hawcxSdk = HawcxSDK(
    context = applicationContext,
    projectApiKey = BuildConfig.HAWCX_PROJECT_KEY,
    baseUrl = BuildConfig.HAWCX_BASE_URL, // host only; SDK appends /hc_auth
    oauthConfig = oauthConfig // optional when exchanging auth codes for tokens
)

hawcxSdk.authenticateV5("user@example.com", object : AuthV5Callback {
    override fun onOtpRequired() {
        // Prompt the user for their OTP and forward it to submitOtpV5(...)
    }

    override fun onAuthSuccess(accessToken: String, refreshToken: String, isLoginFlow: Boolean) {
        // Tokens are also persisted in the credential store for later use.
    }

    override fun onError(errorCode: AuthV5ErrorCode, errorMessage: String) {
        // Present UI feedback or log analytics.
    }
})

// Later, submit the OTP once the user provides it.
hawcxSdk.submitOtpV5(otpFromUser)
```

`baseUrl` should be the customer-specific host Hawcx provisioned for your tenant (for example `https://hawcx-api.hawcx.com`). Pass only the scheme + host; the SDK derives `/hc_auth`, `/ha_login`, `/hc_reg`, and other endpoints from it so the same binary works for every dedicated environment.

- Device registration provisions HKDF salts, wraps Ed25519 keys with Android Keystore, and records metadata via `V5CredentialStore`.
- Login flows reuse stored state and only request OAuth authorization codes when required.
- Tokens are saved to the secure store and the last logged-in user is recorded for push support.

## Backwards Compatibility

Legacy V4 APIs (`authenticateV4`, `submitOtpV4`, etc.) remain available for existing deployments. They continue to use the Paillier challenge/response pipeline and can coexist with V5 flows. The remainder of this document preserves the V4 reference material for teams still migrating.

---

## Legacy V4 Reference

### Table of Contents

- [Installation](#installation)
- [Getting Started](#getting-started)
- [Core Features](#core-features)
  - [Unified Authentication](#unified-authentication)
  - [Biometric Authentication](#biometric-authentication)
  - [Session Management](#session-management)
- [Advanced Features](#advanced-features)
  - [Web Authentication](#web-authentication)
  - [Error Handling](#error-handling)
- [Migration from V3](#migration-from-v3)
- [API Reference](#api-reference)
- [Samples & Resources](#samples--resources)

## Installation

### Requirements
- Android 7.0+ (API level 24+)
- Kotlin 1.9+
- Android Gradle Plugin 8.x

### Installation

1. Download the latest [hawcx-4.0.0.aar](https://github.com/hawcx/hawcx_android_sdk/releases/latest)
2. Place the AAR file in your project's `libs` directory
3. Add to your module's `build.gradle`:

```gradle
dependencies {
    implementation files('libs/hawcx-4.0.0.aar')
    
    // Required dependencies
    implementation 'androidx.biometric:biometric-ktx:1.2.0-alpha05'
    implementation 'com.google.code.gson:gson:2.11.0'
    implementation 'org.jetbrains.kotlinx:kotlinx-coroutines-core:1.7.3'
    implementation 'org.jetbrains.kotlinx:kotlinx-coroutines-android:1.7.3'
    implementation 'com.squareup.retrofit2:retrofit:2.11.0'
    implementation 'com.squareup.retrofit2:converter-gson:2.11.0'
    implementation 'com.squareup.okhttp3:okhttp:4.12.0'
    implementation 'com.squareup.okhttp3:logging-interceptor:4.12.0'
}
```

## Getting Started

### Initialize the SDK

```kotlin
import com.hawcx.sdk.HawcxSDK

// Initialize the SDK (no global initialization required)
val sdk = HawcxSDK("<PROJECT_API_KEY>")
```

### Basic Authentication Flow

```kotlin
class AuthenticationManager : AuthV4Callback {
    private val sdk = HawcxSDK("<PROJECT_API_KEY>")
    
    fun authenticateUser(email: String) {
        // Single method handles all authentication scenarios
        sdk.authenticateV4(userid = email, callback = this)
    }
    
    // MARK: - AuthV4Callback
    override fun onOtpRequired() {
        // Show OTP input UI - user needs to verify email
        showOTPInput()
    }
    
    override fun onAuthSuccess(
        accessToken: String?, 
        refreshToken: String?, 
        isLoginFlow: Boolean
    ) {
        if (isLoginFlow) {
            // User successfully logged in
            navigateToHomeScreen()
        } else {
            // Device registration completed, login will happen automatically
            showMessage("Device registered successfully")
        }
    }
    
    override fun onError(errorCode: AuthV4ErrorCode, errorMessage: String) {
        // Handle authentication errors
        showError(message = errorMessage)
    }
    
    fun submitOTP(otp: String) {
        sdk.submitOtpV4(otp)
    }
}
```

## Core Features

### Unified Authentication

V4 uses a single method for all authentication scenarios:

```kotlin
// Handles all cases automatically:
// - New user registration
// - Existing user login  
// - New device registration
sdk.authenticateV4(userid = "user@example.com", callback = this)

// Submit OTP when required
sdk.submitOtpV4("123456")
```

**Authentication Flow Logic:**
- **New user**: `onOtpRequired()` → Registration → Auto-login
- **Existing user, known device**: Direct `onAuthSuccess()` 
- **Existing user, new device**: `onOtpRequired()` → Device registration → Auto-login

### Biometric Authentication

Integrate fingerprint, face unlock, and other biometric authentication:

```kotlin
import androidx.biometric.BiometricPrompt
import androidx.core.content.ContextCompat
import androidx.fragment.app.FragmentActivity

class BiometricAuthHelper(private val activity: FragmentActivity) {
    private val sdk = HawcxSDK("<PROJECT_API_KEY>")
    
    fun authenticateWithBiometrics(
        username: String,
        onSuccess: () -> Unit,
        onError: (String) -> Unit
    ) {
        val executor = ContextCompat.getMainExecutor(activity)
        
        val biometricPrompt = BiometricPrompt(activity, executor,
            object : BiometricPrompt.AuthenticationCallback() {
                override fun onAuthenticationSucceeded(result: BiometricPrompt.AuthenticationResult) {
                    // Proceed with Hawcx authentication after biometric success
                    sdk.authenticateV4(userid = username, callback = authCallback)
                    onSuccess()
                }
                
                override fun onAuthenticationError(errorCode: Int, errString: CharSequence) {
                    onError(errString.toString())
                }
            })
        
        val promptInfo = BiometricPrompt.PromptInfo.Builder()
            .setTitle("Authenticate")
            .setSubtitle("Use your biometric to log in")
            .setNegativeButtonText("Cancel")
            .build()
        
        biometricPrompt.authenticate(promptInfo)
    }
}
```

**Required Permissions:**
Add to your `AndroidManifest.xml`:
```xml
<uses-permission android:name="android.permission.USE_BIOMETRIC" />
<uses-permission android:name="android.permission.USE_FINGERPRINT" />
```

### Session Management

V4 provides enhanced session management with granular control:

```kotlin
// Get last logged in user for UI pre-filling
val lastUser = sdk.getLastLoggedInUsername()
if (lastUser.isNotEmpty()) {
    emailField.setText(lastUser)
}

// Standard logout - keeps device registration, clears session tokens
sdk.clearSessionTokens("user@example.com")

// Full device removal - user must re-register this device
sdk.clearUserKeychainData("user@example.com")

// Clear last logged in user marker (for UI pre-fill)
sdk.clearLastLoggedInUser()
```

## Advanced Features

### Web Authentication

Support QR code and PIN-based web authentication:

```kotlin
class WebAuthManager : WebLoginCallback {
    private val sdk = HawcxSDK("<PROJECT_API_KEY>")
    
    // Step 1: Submit PIN from web login
    fun submitWebLoginPIN(pin: String) {
        sdk.webLogin(pin = pin, callback = this)
    }
    
    // Step 2: Approve the web login request
    fun approveWebLogin(webToken: String) {
        sdk.webApprove(token = webToken, callback = this)
    }
    
    // MARK: - WebLoginCallback
    override fun onWebLoginSuccess() {
        // Web login PIN submitted successfully
        showMessage("Web login initiated")
    }
    
    override fun onWebApproveSuccess() {
        // Web login approved successfully
        showMessage("Web login approved")
    }
    
    override fun onWebLoginError(errorCode: WebLoginErrorCode, errorMessage: String) {
        // Handle web authentication errors
        showError(message = errorMessage)
    }
}
```

### Error Handling

V4 provides comprehensive error codes for better user experience:

```kotlin
override fun onError(errorCode: AuthV4ErrorCode, errorMessage: String) {
    when (errorCode) {
        AuthV4ErrorCode.NETWORK_ERROR -> {
            showRetryableError("Please check your internet connection")
        }
        
        AuthV4ErrorCode.OTP_VERIFICATION_FAILED -> {
            showError("Invalid verification code. Please try again.")
        }
        
        AuthV4ErrorCode.DEVICE_VERIFICATION_FAILED -> {
            showError("Device verification failed. Please try again.")
        }
        
        AuthV4ErrorCode.FINGERPRINT_ERROR -> {
            showError("Biometric authentication is not available")
        }
        
        AuthV4ErrorCode.KEYCHAIN_SAVE_FAILED -> {
            showError("Failed to save authentication data securely")
        }
        
        AuthV4ErrorCode.AUTH_INIT_FAILED -> {
            showError("Authentication initialization failed")
        }
        
        AuthV4ErrorCode.CIPHER_VERIFICATION_FAILED -> {
            showError("Login verification failed")
        }
        
        AuthV4ErrorCode.MISSING_DEVICE_TOKEN_SESSION -> {
            showError("Session expired. Please try again.")
        }
        
        AuthV4ErrorCode.INTERNAL_STATE_ERROR -> {
            showError("An unexpected error occurred. Please update the app.")
        }
        
        else -> {
            showError("Authentication failed: $errorMessage")
        }
    }
}
```

## Migration from V3

### Before (V3):
```kotlin
// V3 required global initialization and multiple managers
HawcxInitializer.getInstance().init(applicationContext, "YOUR_API_KEY")

val signUpManager = HawcxInitializer.getInstance().signUp
val signInManager = HawcxInitializer.getInstance().signIn
val addDeviceManager = HawcxInitializer.getInstance().addDeviceManager

// Different methods for different scenarios
signUpManager.signUp(userid = email, callback = this)
signInManager.signIn(userid = email, callback = this)
addDeviceManager.startAddDeviceFlow(userid = email, callback = this)
```

### After (V4):
```kotlin
// V4 uses single SDK instance and method
val sdk = HawcxSDK("<PROJECT_API_KEY>")

// One method handles all scenarios
sdk.authenticateV4(userid = email, callback = this)
```

### Callback Migration

**V3 had multiple callback interfaces:**
- `SignUpCallback`
- `SignInCallback` 
- `AddDeviceCallback`

**V4 uses unified `AuthV4Callback`:**
```kotlin
interface AuthV4Callback {
    fun onOtpRequired()
    fun onAuthSuccess(accessToken: String?, refreshToken: String?, isLoginFlow: Boolean)
    fun onError(errorCode: AuthV4ErrorCode, errorMessage: String)
}
```

## API Reference

### Core Classes

#### HawcxSDK
| Method | Description |
| ------ | ----------- |
| `HawcxSDK(projectApiKey: String)` | Initialize SDK with your API key |
| `authenticateV4(userid: String, callback: AuthV4Callback)` | Unified authentication method |
| `submitOtpV4(otp: String)` | Submit OTP during verification |
| `getLastLoggedInUsername(): String` | Get last logged in user for UI pre-fill |
| `clearSessionTokens(userid: String)` | Standard logout (keeps device registration) |
| `clearUserKeychainData(userid: String)` | Full device removal |
| `clearLastLoggedInUser()` | Clear last user marker |
| `webLogin(pin: String, callback: WebLoginCallback)` | Submit web login PIN |
| `webApprove(token: String, callback: WebLoginCallback)` | Approve web login |

### Callback Interfaces

#### AuthV4Callback
```kotlin
interface AuthV4Callback {
    fun onOtpRequired()
    fun onAuthSuccess(accessToken: String?, refreshToken: String?, isLoginFlow: Boolean)
    fun onError(errorCode: AuthV4ErrorCode, errorMessage: String)
}
```

#### WebLoginCallback
```kotlin
interface WebLoginCallback {
    fun onWebLoginSuccess()
    fun onWebApproveSuccess() 
    fun onWebLoginError(errorCode: WebLoginErrorCode, errorMessage: String)
}
```

### Error Codes

#### AuthV4ErrorCode
- `NETWORK_ERROR` - Connectivity issues
- `OTP_VERIFICATION_FAILED` - Invalid OTP
- `DEVICE_VERIFICATION_FAILED` - Device registration failed
- `FINGERPRINT_ERROR` - Biometric authentication unavailable
- `KEYCHAIN_SAVE_FAILED` - Secure storage failed
- `AUTH_INIT_FAILED` - Authentication initialization failed
- `CIPHER_VERIFICATION_FAILED` - Login verification failed
- `MISSING_DEVICE_TOKEN_SESSION` - Session token lost
- `INTERNAL_STATE_ERROR` - Unexpected SDK state

## ProGuard/R8 Configuration

Add these rules to your `proguard-rules.pro`:

```proguard
# Hawcx SDK V4
-keep class com.hawcx.sdk.** { *; }
-keep interface com.hawcx.sdk.** { *; }
-dontwarn com.hawcx.sdk.**

# Keep callback interfaces
-keep interface com.hawcx.sdk.AuthV4Callback { *; }
-keep interface com.hawcx.sdk.DevSessionCallback { *; }
-keep interface com.hawcx.sdk.WebLoginCallback { *; }
```

## Samples & Resources

### Demo Application

Explore our [V4 demo application](https://github.com/hawcx/hawcx_android_demo) featuring:
- Unified authentication implementation
- Biometric integration
- Session management  
- Web authentication
- Modern Jetpack Compose architecture
- Complete error handling

### Best Practices

#### MVVM with StateFlow
```kotlin
class AuthViewModel : ViewModel(), AuthV4Callback {
    private val sdk = HawcxSDK("<PROJECT_API_KEY>")
    
    private val _authState = MutableStateFlow<AuthState>(AuthState.Idle)
    val authState = _authState.asStateFlow()
    
    fun authenticate(email: String) {
        _authState.value = AuthState.Loading
        sdk.authenticateV4(userid = email, callback = this)
    }
    
    override fun onAuthSuccess(accessToken: String?, refreshToken: String?, isLoginFlow: Boolean) {
        _authState.value = if (isLoginFlow) {
            AuthState.Success
        } else {
            AuthState.DeviceRegistered
        }
    }
}
```

#### Biometric Authentication Flow
```kotlin
fun attemptBiometricLogin() {
    val lastUser = sdk.getLastLoggedInUsername()
    if (lastUser.isNotEmpty() && biometricsEnabled(lastUser)) {
        biometricHelper.authenticateWithBiometrics(
            username = lastUser,
            onSuccess = { 
                // Biometric success, proceed with Hawcx auth
            },
            onError = { error ->
                // Handle biometric failure
            }
        )
    }
}
```

#### Session State Management
```kotlin
fun handleAuthSuccess(isLoginFlow: Boolean) {
    if (isLoginFlow) {
        // User is now logged in
        updateUIForLoggedInState()
        navigateToHomeScreen()
    } else {
        // Device was registered, login will happen automatically
        showTemporaryMessage("Device registered successfully")
    }
}
```

## Support

- [Documentation](https://docs.hawcx.com)
- [API Reference](https://docs.hawcx.com/ios/quickstart)
- [Website](https://www.hawcx.com)
- [Support Email](mailto:info@hawcx.com)
