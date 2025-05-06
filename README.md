# Hawcx Android SDK

Hawcx provides enterprise-grade passwordless authentication for Android applications, delivering a secure and frictionless login experience across all user devices.

[![Kotlin Version](https://img.shields.io/badge/Kotlin-1.8+-blue.svg)](https://kotlinlang.org)
[![Platform](https://img.shields.io/badge/platform-Android_8.0+-green.svg)](https://developer.android.com/about)
[![License](https://img.shields.io/badge/license-Custom-red.svg)](LICENSE)

## Table of Contents

- [Installation](#installation)
- [Getting Started](#getting-started)
- [Core Features](#core-features)
  - [User Registration](#user-registration)
  - [User Authentication](#user-authentication)
  - [Biometric Authentication](#biometric-authentication)
  - [Device Session Management](#device-session-management)
- [Advanced Features](#advanced-features)
  - [Multi-Device Support](#multi-device-support)
  - [Error Handling](#error-handling)
- [API Reference](#api-reference)
- [Samples & Resources](#samples--resources)

## Installation

### Requirements
- Android 8.0+ (API level 26+)
- Kotlin 1.8+

> **Note:** Android offers multiple integration methods:
> - **JitPack**: Similar to Swift Package Manager, using GitHub coordinates
> - **Direct Maven**: Pulls from our Maven repository structure
> - **Manual AAR**: Similar to embedding XCFramework in iOS


### Installation

1. Download the latest [hawcx-3.0.0.aar](https://github.com/hawcx/hawcx_android_sdk/releases/latest)
2. Place the AAR file in your project's `libs` directory
3. Add this to your module's build.gradle:

```gradle
dependencies {
    implementation files('libs/hawcx-3.0.0.aar')
    // Required dependencies
    implementation 'androidx.biometric:biometric-ktx:1.2.0-alpha05'
    implementation 'com.google.code.gson:gson:2.11.0'
    implementation 'org.jetbrains.kotlinx:kotlinx-coroutines-core:1.7.3'
    implementation 'org.jetbrains.kotlinx:kotlinx-coroutines-android:1.7.3'
    implementation 'com.squareup.retrofit2:retrofit:2.10.0'
    implementation 'com.squareup.retrofit2:converter-gson:2.10.0'
    implementation 'com.squareup.okhttp3:okhttp:4.12.0'
    implementation 'com.squareup.okhttp3:logging-interceptor:4.11.0'
    implementation 'com.google.android.gms:play-services-tasks:18.2.0'
    implementation 'com.google.android.gms:play-services-location:21.3.0'
}
```

## Getting Started

### Initialize the SDK

```kotlin
import com.hawcx.internal.HawcxInitializer

// In your Application class
class YourApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        
        // Initialize Hawcx SDK with your API key
        HawcxInitializer.getInstance().init(
            context = this,
            apiKey = "YOUR_API_KEY"
        )
    }
}
```

## Core Features

### User Registration

Register new users with a simple, secure process.

```kotlin
// Implement the SignUpCallback interface
class RegistrationManager(context: Context) : SignUpCallback {
    // Get SignUp manager from HawcxInitializer
    private val signUp = HawcxInitializer.getInstance().signUp
    
    fun registerUser(email: String) {
        signUp.signUp(userid = email, callback = this)
    }
    
    fun verifyOTP(otp: String) {
        signUp.handleVerifyOTP(otp = otp, callback = this)
    }
    
    // MARK: - SignUpCallback methods
    override fun onGenerateOTPSuccess() {
        // Show OTP verification UI
    }
    
    override fun onSuccessfulSignUpOrDeviceAdd() {
        // Registration complete, proceed to home screen
    }
    
    override fun onError(error: SignUpError, message: String) {
        // Handle specific error
    }
}
```

### User Authentication

Authenticate returning users with a passwordless flow.

```kotlin
// Implement the SignInCallback interface
class AuthenticationManager(context: Context) : SignInCallback {
    // Get SignIn manager from HawcxInitializer
    private val signIn = HawcxInitializer.getInstance().signIn
    
    fun authenticateUser(email: String) {
        signIn.signIn(userid = email, callback = this)
    }
    
    // MARK: - SignInCallback methods
    override fun onSuccessfulLogin(email: String, accessToken: String, refreshToken: String) {
        // User authenticated successfully
    }
    
    override fun navigateToRegistration(forUserId: String) {
        // User doesn't exist, show registration
    }
    
    override fun initiateAddDeviceRegistrationFlow(forUserId: String) {
        // This device needs to be added to the user's account
    }
    
    override fun onError(error: SignInError, message: String) {
        // Handle specific error
    }
    
    override fun requestUserAuthentication(
        onSuccess: () -> Unit,
        onFailure: (SignInError, String) -> Unit
    ) {
        // Handle biometric authentication request
    }
}
```

### Biometric Authentication

Enhance security with fingerprint or face authentication.

```kotlin
import androidx.biometric.BiometricPrompt
import androidx.core.content.ContextCompat
import androidx.fragment.app.FragmentActivity

class BiometricAuthHelper(private val activity: FragmentActivity) {
    
    fun authenticateWithBiometrics(
        onSuccess: () -> Unit,
        onError: (Int, String) -> Unit
    ) {
        val executor = ContextCompat.getMainExecutor(activity)
        
        val biometricPrompt = BiometricPrompt(activity, executor,
            object : BiometricPrompt.AuthenticationCallback() {
                override fun onAuthenticationSucceeded(result: BiometricPrompt.AuthenticationResult) {
                    // Proceed with Hawcx authentication
                    onSuccess()
                }
                
                override fun onAuthenticationError(errorCode: Int, errString: CharSequence) {
                    onError(errorCode, errString.toString())
                }
            })
        
        val promptInfo = BiometricPrompt.PromptInfo.Builder()
            .setTitle("Authenticate")
            .setSubtitle("Authenticate with biometrics")
            .setNegativeButtonText("Cancel")
            .build()
        
        biometricPrompt.authenticate(promptInfo)
    }
}
```

**Important**: Add `<uses-permission android:name="android.permission.USE_BIOMETRIC" />` to your AndroidManifest.xml.

### Device Session Management

Fetch and manage information about user sessions across devices.

```kotlin
// Implement the DevSessionCallback interface
class SessionManager(context: Context) : DevSessionCallback {
    // Get DevSession manager from HawcxInitializer
    private val devSession = HawcxInitializer.getInstance().devSession
    
    fun fetchDeviceDetails() {
        devSession.getDeviceDetails(callback = this)
    }
    
    // MARK: - DevSessionCallback methods
    override fun onSuccess() {
        // Access device information from SharedPreferences
        val devices = devSession.getStoredDeviceDetails()
        if (devices != null) {
            // Use devices data
        }
    }
    
    override fun onError() {
        // Handle error
    }
}
```

## Advanced Features

### Multi-Device Support

Enable users to securely access their accounts across multiple devices.

```kotlin
// Implement the AddDeviceCallback interface
class DeviceManager(context: Context) : AddDeviceCallback {
    // Get AddDeviceManager from HawcxInitializer
    private val addDeviceManager = HawcxInitializer.getInstance().addDeviceManager
    
    fun addDevice(email: String) {
        addDeviceManager.startAddDeviceFlow(userid = email, callback = this)
    }
    
    fun verifyOTP(otp: String) {
        addDeviceManager.handleVerifyOTP(otp = otp)
    }
    
    // MARK: - AddDeviceCallback methods
    override fun onGenerateOTPSuccess() {
        // Show OTP verification UI
    }
    
    override fun onAddDeviceSuccess() {
        // Device successfully added, proceed to home screen
    }
    
    override fun showError(addDeviceError: AddDeviceError, errorMessage: String) {
        // Handle specific error
    }
}
```

### Error Handling

Implement robust error handling for different scenarios.

```kotlin
override fun onError(error: SignInError, message: String) {
    when (error) {
        SignInError.USER_NOT_FOUND -> {
            // Show registration option
        }
        
        SignInError.ADD_DEVICE_REQUIRED, 
        SignInError.INVALID_DEVICE_TOKEN,
        SignInError.MISSING_KEY_DATA -> {
            // Redirect to Add Device flow
        }
        
        SignInError.NETWORK_ERROR -> {
            // Show connectivity error
        }
        
        else -> {
            // Handle other errors
        }
    }
}
```

## API Reference

### Core Classes

#### SignUp
| Method | Description |
|--------|-------------|
| `signUp(userid: String, callback: SignUpCallback)` | Start the registration process |
| `handleVerifyOTP(otp: String, callback: SignUpCallback)` | Verify the OTP during registration |

#### SignIn
| Method | Description |
|--------|-------------|
| `signIn(userid: String, callback: SignInCallback)` | Authenticate a user |

#### AddDeviceManager
| Method | Description |
|--------|-------------|
| `startAddDeviceFlow(userid: String, callback: AddDeviceCallback)` | Start the device addition flow |
| `handleVerifyOTP(otp: String)` | Verify the OTP during device addition |

#### DevSession
| Method | Description |
|--------|-------------|
| `getDeviceDetails(callback: DevSessionCallback)` | Fetch device session details |
| `getStoredDeviceDetails(): List<DeviceDetails>?` | Get cached device details |


### Callback Interfaces

#### SignUpCallback
```kotlin
public interface SignUpCallback {
    fun onError(error: SignUpError, message: String)
    fun onSuccessfulSignUpOrDeviceAdd()
    fun onGenerateOTPSuccess()
}
```

#### SignInCallback
```kotlin
public interface SignInCallback {
    fun onError(error: SignInError, message: String)
    fun onSuccessfulLogin(email: String, accessToken: String, refreshToken: String)
    fun navigateToRegistration(forUserId: String)
    fun initiateAddDeviceRegistrationFlow(forUserId: String)
    fun requestUserAuthentication(
        onSuccess: () -> Unit,
        onFailure: (SignInError, String) -> Unit
    )
}
```

#### AddDeviceCallback
```kotlin
public interface AddDeviceCallback {
    fun showError(addDeviceError: AddDeviceError, errorMessage: String)
    fun onAddDeviceSuccess()
    fun onGenerateOTPSuccess()
}
```

#### DevSessionCallback
```kotlin
public interface DevSessionCallback {
    fun onSuccess()
    fun onError()
}
```

## Samples & Resources

### Sample App

Explore our sample application for complete implementations of:

- User registration
- Authentication
- Multi-device support
- Session management
- Biometric integration

### Common Patterns

#### MVVM with Jetpack Compose

```kotlin
class LoginViewModel(
    private val appViewModel: AppViewModel
) : ViewModel(), SignInCallback {
    // StateFlow for UI state
    private val _isLoggingIn = MutableStateFlow(false)
    val isLoggingIn = _isLoggingIn.asStateFlow()
    
    // ...Login implementation...
} 

@Composable
fun LoginScreen(viewModel: LoginViewModel) {
    // Collect StateFlow values
    val isLoggingIn by viewModel.isLoggingIn.collectAsStateWithLifecycle()
    
    // UI implementation
}
```

#### Handling Device Addition During Login

```kotlin
override fun initiateAddDeviceRegistrationFlow(forUserId: String) {
    viewModel.navigateToAddDevice(forUserId)
}
```

### Support

- [Documentation](https://docs.hawcx.com)
- [API Reference](https://docs.hawcx.com/android/quickstart)
- [Website](https://hawcx.com)
- [Support Email](mailto:info@hawcx.com)
