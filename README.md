# SecureAttend Android Application

SecureAttend Android is the mobile frontend for the SecureAttend attendance management system. This application implements a secure multi-factor authentication approach to prevent proxy attendance by combining QR code scanning, face recognition, and Bluetooth proximity detection.

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Project Structure](#project-structure)
- [Getting Started](#getting-started)
  - [Prerequisites](#prerequisites)
  - [Setup](#setup)
  - [Configuration](#configuration)
- [Implementation Details](#implementation-details)
  - [Authentication](#authentication)
  - [QR Code System](#qr-code-system)
  - [Face Verification](#face-verification)
  - [Bluetooth Proximity](#bluetooth-proximity)
- [Usage Guide](#usage-guide)
  - [Faculty Usage](#faculty-usage)
  - [Student Usage](#student-usage)
- [Technology Stack](#technology-stack)
- [Troubleshooting](#troubleshooting)
- [Contributing](#contributing)
- [License](#license)

## Overview

The SecureAttend Android application is the mobile client component of the SecureAttend system, designed to provide a seamless experience for both faculty and students. The app ensures attendance integrity through a robust multi-factor authentication approach:

1. **QR Code Verification**: Faculty generates a unique QR code for each session
2. **Face Recognition**: Students verify their identity using device biometrics
3. **Bluetooth Proximity**: The system confirms the student's physical presence near the faculty device

This combination of factors ensures that students must be physically present in the classroom to mark attendance, effectively preventing proxy attendance.

## Features

### Faculty Features
- Secure authentication with role-based access
- Session creation for courses and rooms
- QR code generation for attendance sessions
- Bluetooth broadcasting for proximity verification
- Real-time attendance monitoring
- Historical session and attendance data
- Session management controls (start/end sessions)

### Student Features
- Secure authentication with role-based access
- QR code scanning for session verification
- Face verification using device biometrics
- Bluetooth proximity detection
- Multi-factor attendance marking
- Personal attendance history view
- Verification status feedback

### Core Components
- JWT-based authentication system
- Role-specific user interfaces
- Camera integration for QR scanning
- BiometricPrompt integration for face verification
- BLE (Bluetooth Low Energy) for proximity detection
- Comprehensive error handling and feedback
- Offline capability for recent attendance data

## Project Structure

The app follows an MVVM architecture with clean separation of concerns:

```
com.secureattend/
├── config/
│   └── ApiConfig.kt              # API configuration
├── core/
│   ├── biometric/
│   │   └── BiometricHelper.kt    # Face authentication helper
│   ├── bluetooth/
│   │   ├── BleAdvertiser.kt      # Faculty BLE broadcasting
│   │   └── BleScanner.kt         # Student BLE scanning
│   └── camera/
│       └── CameraHelper.kt       # Camera utilities
├── data/
│   ├── model/                    # Data models
│   │   ├── LoginResponse.kt      # Authentication models
│   │   ├── QRCodeResponse.kt     # QR code models
│   │   ├── SessionResponse.kt    # Session models
│   │   └── AttendanceRecord.kt   # Attendance models
│   ├── network/                  # Network components
│   │   ├── ApiService.kt         # API endpoints interface
│   │   ├── RetrofitClient.kt     # Network client
│   │   └── AuthInterceptor.kt    # Auth token interceptor
│   └── SessionManager.kt         # User session management
├── ui/
│   ├── auth/                     # Authentication UI
│   │   ├── LoginActivity.kt      # Login screen
│   │   └── LoginViewModel.kt     # Login logic
│   ├── faculty/                  # Faculty screens
│   │   ├── FacultyMainActivity.kt # Faculty dashboard
│   │   ├── QRCodeActivity.kt     # QR code display
│   │   ├── SessionHistoryActivity.kt # Session history
│   │   └── SessionDetailActivity.kt # Session details
│   └── student/                  # Student screens
│       ├── StudentMainActivity.kt # Student dashboard
│       ├── QRScannerActivity.kt  # QR code scanner
│       ├── AttendanceHistoryActivity.kt # Attendance history
│       └── FaceVerificationActivity.kt # Face verification
└── SecureAttendApp.kt           # Application class
```

## Getting Started

### Prerequisites

- Android Studio Iguana (2023.2.1) or newer
- Minimum SDK: Android 7.0 (API level 24)
- Target SDK: Android 13 (API level 33)
- JDK 17
- Device with camera and Bluetooth capabilities

### Setup

1. Clone the repository:
   ```bash
   git clone https://github.com/yourusername/secure_attend_android.git
   cd secure_attend_android
   ```

2. Open the project in Android Studio:
   - Start Android Studio
   - Select "Open an existing Android Studio project"
   - Navigate to the cloned repository folder and click "OK"

3. Build the project:
   - Click "Build" > "Rebuild Project"
   - Wait for the Gradle build to complete

### Configuration

1. Update the API configuration in `ApiConfig.kt`:
   ```kotlin
   // app/src/main/java/com/secureattend/config/ApiConfig.kt
   
   // Replace with your server's IP address and port
   const val SERVER_IP = "192.168.1.100"  // Your server's IP
   const val SERVER_PORT = "8000"         // Default FastAPI port
   ```

2. Check Bluetooth and camera permissions in `AndroidManifest.xml`:
   ```xml
   <uses-permission android:name="android.permission.CAMERA" />
   <uses-permission android:name="android.permission.BLUETOOTH" />
   <uses-permission android:name="android.permission.BLUETOOTH_ADMIN" />
   
   <!-- For Android 12+ -->
   <uses-permission android:name="android.permission.BLUETOOTH_CONNECT" />
   <uses-permission android:name="android.permission.BLUETOOTH_SCAN" />
   <uses-permission android:name="android.permission.BLUETOOTH_ADVERTISE" />
   ```

## Implementation Details

### Authentication

The authentication system uses JWT tokens and includes:
- User login with email/password
- Token storage in encrypted SharedPreferences
- Automatic token inclusion in API requests
- Role-based navigation to appropriate screens

```kotlin
// Example login flow
val loginResponse = apiService.login(email, password)
if (loginResponse.isSuccessful) {
    SessionManager.saveSession(
        token = loginResponse.body()?.accessToken,
        userId = loginResponse.body()?.userId,
        userRole = loginResponse.body()?.role,
        userName = loginResponse.body()?.fullName
    )
    
    // Navigate based on role
    if (SessionManager.isFaculty()) {
        startActivity(Intent(this, FacultyMainActivity::class.java))
    } else {
        startActivity(Intent(this, StudentMainActivity::class.java))
    }
}
```

### QR Code System

The QR code system handles:
- Generation of QR codes for faculty sessions
- Scanning QR codes by students
- Verification of session information
- Extraction of proximity UUID for Bluetooth verification

```kotlin
// Faculty: Display QR code
val qrResponse = apiService.getSessionQR(sessionId)
val imageUrl = qrResponse.image_url
Glide.with(context).load(imageUrl).into(imageView)

// Student: Scan QR code
scanLauncher.launch(ScanOptions())
// Then process the result
scanLauncher = registerForActivityResult(ScanContract()) { result ->
    result.contents?.let { qrData ->
        // Process the QR code data
        viewModel.processQrCode(qrData)
    }
}
```

### Face Verification

Face verification uses Android's BiometricPrompt for secure, on-device biometric authentication:

```kotlin
val biometricPrompt = BiometricPrompt(this, executor,
    object : BiometricPrompt.AuthenticationCallback() {
        override fun onAuthenticationSucceeded(result: BiometricPrompt.AuthenticationResult) {
            // Face verification succeeded
            isFaceVerified = true
            updateVerificationStatus()
        }
        
        override fun onAuthenticationError(errorCode: Int, errString: CharSequence) {
            // Handle authentication error
        }
    })

val promptInfo = BiometricPrompt.PromptInfo.Builder()
    .setTitle("Verify your identity")
    .setSubtitle("Use your face to verify your identity")
    .setNegativeButtonText("Cancel")
    .build()

biometricPrompt.authenticate(promptInfo)
```

### Bluetooth Proximity

Bluetooth proximity uses BLE (Bluetooth Low Energy) for:
- Faculty broadcasting a unique session identifier
- Student scanning for this identifier
- Verification of physical proximity based on signal strength

```kotlin
// Faculty: Broadcast proximity UUID
bleAdvertiser.startAdvertising(proximityUuid) { success, message ->
    if (success) {
        // Broadcasting started successfully
    } else {
        // Handle broadcasting error
    }
}

// Student: Scan for faculty device
bleScanner.startScanning(proximityUuid) { isNearby, rssi, message ->
    if (isNearby) {
        // Proximity verified
        isProximityVerified = true
        updateVerificationStatus()
    } else {
        // Not in proximity or error
    }
}
```

## Usage Guide

### Faculty Usage

1. **Login**: Enter your faculty credentials to access the faculty dashboard.
2. **Create Session**: Tap "Create Session" and select the course and room.
3. **Display QR Code**: The app will generate and display a QR code for students to scan.
4. **Monitor Attendance**: View real-time updates of students marking attendance.
5. **End Session**: Tap "End Session" when the class is over.
6. **View History**: Access past sessions and attendance records through "View Past Sessions".

### Student Usage

1. **Login**: Enter your student credentials to access the student dashboard.
2. **Scan QR Code**: Tap "Scan QR" and scan the faculty's QR code.
3. **Verify Face**: Complete face verification when prompted.
4. **Verify Proximity**: The app will automatically check proximity to the faculty device.
5. **Mark Attendance**: Once all verifications are complete, attendance is marked.
6. **View History**: Access your attendance history through "View Attendance History".

## Technology Stack

- **Language**: Kotlin
- **UI Framework**: Android Views and Material Components
- **Architecture**: MVVM with LiveData and ViewModels
- **Networking**: Retrofit with OkHttp
- **JSON Parsing**: Gson
- **Image Loading**: Glide
- **QR Code**: ZXing
- **Face Recognition**: Android BiometricPrompt
- **Bluetooth**: Android BLE APIs
- **Local Storage**: SharedPreferences with encryption

## Troubleshooting

### Common Issues

1. **API Connection Issues**:
   - Verify server is running
   - Check IP address in ApiConfig.kt
   - Ensure device is on the same network as server

2. **QR Code Scanning Problems**:
   - Ensure proper lighting
   - Hold device steady
   - Verify camera permissions are granted

3. **Face Verification Issues**:
   - Ensure device supports biometric authentication
   - Check adequate lighting for face recognition
   - Verify biometric settings are enabled on device

4. **Bluetooth Connection Issues**:
   - Ensure Bluetooth is enabled
   - Verify Bluetooth permissions are granted
   - Check proximity to faculty device

## Contributing

Contributions to the SecureAttend Android application are welcome! To contribute:

1. Fork the repository
2. Create your feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add some amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

## License

This project is licensed under the MIT License - see the LICENSE file for details.
