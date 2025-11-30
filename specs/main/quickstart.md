# SMARGE Quickstart Guide

**Platform**: iOS (Flutter)  
**Prerequisites**: macOS with Xcode, Flutter SDK  
**Time to First Build**: ~30 minutes

---

## Prerequisites

### Required

1. **macOS** (for iOS development)
   - macOS Sequoia 15.x or later recommended
   - 8GB RAM minimum, 16GB recommended

2. **Xcode**
   - Version 16.0+ (for iOS 17-26 support)
   - Install from Mac App Store
   - Run `xcode-select --install` to install command line tools

3. **Flutter SDK**
   - Version 3.24 or later
   - Install via: https://docs.flutter.dev/get-started/install/macos

4. **Git**
   - Pre-installed on macOS or via Xcode CLI tools

### Optional

- **VS Code** with Flutter extension (recommended IDE)
- **Android Studio** (alternative IDE)
- **iOS Simulator** (included with Xcode)
- **Physical iPhone** (iOS 17+) for testing

---

## Setup Instructions

### 1. Install Flutter

```powershell
# Download Flutter SDK
git clone https://github.com/flutter/flutter.git -b stable ~/flutter

# Add to PATH (add to ~/.zshrc or ~/.bashrc)
export PATH="$HOME/flutter/bin:$PATH"

# Verify installation
flutter doctor
```

**Expected Output:**
```
Doctor summary (to see all details, run flutter doctor -v):
[✓] Flutter (Channel stable, 3.24.x)
[✓] Xcode - develop for iOS
[✓] Chrome - develop for the web
[✓] VS Code (version x.x.x)
```

### 2. Clone Repository

```powershell
git clone https://github.com/schdief/smarge.git
cd smarge
```

### 3. Install Dependencies

```powershell
# Get all Flutter packages
flutter pub get

# Generate code (for Isar, JSON serialization, etc.)
flutter pub run build_runner build --delete-conflicting-outputs
```

### 4. Configure iOS

```powershell
# Open iOS project in Xcode
open ios/Runner.xcworkspace
```

**In Xcode:**
1. Select `Runner` in left sidebar
2. Go to "Signing & Capabilities" tab
3. Select your Apple ID team
4. Change Bundle Identifier to unique value (e.g., `com.yourname.smarge`)
5. Enable capabilities (will be added during development):
   - Background Modes → Background fetch, Remote notifications
   - Push Notifications

### 5. Run on Simulator

```powershell
# List available simulators
flutter emulators

# Launch iPhone simulator
flutter emulators --launch apple_ios_simulator

# Run app
flutter run
```

**Or in VS Code:**
- Press `F5` (Run → Start Debugging)
- Select "iPhone 15 Pro" or similar from device picker

### 6. Run on Physical Device

```powershell
# Connect iPhone via USB
# Unlock iPhone and trust computer if prompted

# List connected devices
flutter devices

# Run on device
flutter run -d <device-id>
```

---

## Project Structure Overview

```
smarge/
├── lib/
│   ├── main.dart              # App entry point
│   ├── models/                # Data models (Isar)
│   ├── services/              # Business logic
│   │   ├── optimization/      # Optimization engine
│   │   ├── api/               # API clients
│   │   └── background/        # Background tasks
│   ├── ui/                    # Flutter widgets
│   │   ├── dashboard/
│   │   ├── bev/
│   │   ├── analytics/
│   │   └── settings/
│   └── utils/                 # Helpers
│
├── test/                      # Unit & widget tests
├── integration_test/          # E2E tests
├── ios/                       # iOS-specific code
├── pubspec.yaml               # Dependencies
└── README.md
```

---

## Development Workflow

### Running Tests

```powershell
# Run all unit tests
flutter test

# Run specific test file
flutter test test/services/optimization_engine_test.dart

# Run with coverage
flutter test --coverage
genhtml coverage/lcov.info -o coverage/html
open coverage/html/index.html
```

### Code Generation

**When to run:**
- After adding/modifying Isar models
- After adding/modifying JSON serialization
- After adding/modifying Riverpod providers with `@riverpod` annotation

```powershell
# Watch mode (auto-generates on file changes)
flutter pub run build_runner watch

# One-time build
flutter pub run build_runner build --delete-conflicting-outputs
```

### Linting & Formatting

```powershell
# Analyze code for issues
flutter analyze

# Format all Dart files
dart format .

# Fix auto-fixable issues
dart fix --apply
```

### Building Release

```powershell
# iOS Release Build
flutter build ios --release

# Or build in Xcode:
# Product → Archive → Distribute App → App Store Connect
```

---

## Configuration

### API Credentials (Development)

Create `.env` file in project root (not committed to git):

```env
TIBBER_ACCESS_TOKEN=your_tibber_token_here
MYSKODA_EMAIL=your_email@example.com
MYSKODA_PASSWORD=your_password_here
MYSKODA_SPIN=1234
```

Load in app using `flutter_dotenv`:

```dart
import 'package:flutter_dotenv/flutter_dotenv.dart';

Future<void> main() async {
  await dotenv.load(fileName: ".env");
  runApp(SmargeApp());
}

// Usage
final tibberToken = dotenv.env['TIBBER_ACCESS_TOKEN']!;
```

**⚠️ Production:**
- User enters credentials during onboarding
- Stored in Flutter Secure Storage
- Never commit credentials to git

### System Configuration

During onboarding, user configures:

```dart
final config = SystemConfig(
  // Solar
  solarCapacityKW: 13.0,
  solarOrientation: 'southwest',
  solarTiltDegrees: 45.0,
  latitude: 52.5200,
  longitude: 13.4050,
  
  // Battery
  batteryCapacityKWh: 12.0,
  batteryMaxChargeRateKW: 5.0,
  batteryMaxDischargeRateKW: 5.0,
  
  // BEV
  bevCapacityKWh: 82.0,
  bevMaxChargeRateKW: 11.0,
  bevDefaultMinimumSoC: 0.40,
  bevVIN: 'TMBJR1NV9N1234567',
  bevModel: 'Enyaq iV 80',
);
```

---

## Debugging

### Common Issues

**Issue**: "No devices found"
```powershell
# Kill ADB
adb kill-server

# Restart simulator
xcrun simctl shutdown all
xcrun simctl boot <simulator-udid>
```

**Issue**: "Pod install failed"
```powershell
cd ios
pod deintegrate
pod install
cd ..
flutter clean
flutter pub get
```

**Issue**: "Build failed: provisioning profile"
- Open `ios/Runner.xcworkspace` in Xcode
- Select your team in Signing & Capabilities
- Clean build folder: Product → Clean Build Folder

### Flutter DevTools

```powershell
# Launch DevTools
flutter pub global activate devtools
flutter pub global run devtools

# Run app in debug mode
flutter run --observatory-port=9200

# Open DevTools at http://127.0.0.1:9100
```

**Features:**
- Widget inspector
- Performance profiling
- Network monitoring
- Logging
- Memory usage

---

## Testing Strategy

### 1. Unit Tests

**Location**: `test/unit/`

```dart
// test/unit/services/optimization_engine_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:smarge/services/optimization/optimization_engine.dart';

void main() {
  group('OptimizationEngine', () {
    late OptimizationEngine engine;
    
    setUp(() {
      engine = OptimizationEngine();
    });
    
    test('should prioritize solar over grid', () {
      final result = engine.optimize(
        bevGoal: mockGoal,
        prices: mockPrices,
        solarForecast: mockSolarForecast,
      );
      
      expect(result.first.source, EnergySource.solar);
    });
  });
}
```

### 2. Widget Tests

**Location**: `test/widget/`

```dart
// test/widget/dashboard_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:smarge/ui/dashboard/dashboard_screen.dart';

void main() {
  testWidgets('Dashboard shows BEV status', (tester) async {
    await tester.pumpWidget(
      MaterialApp(home: DashboardScreen()),
    );
    
    expect(find.text('BEV Status'), findsOneWidget);
    expect(find.byType(BevStatusCard), findsOneWidget);
  });
}
```

### 3. Integration Tests

**Location**: `integration_test/`

```dart
// integration_test/app_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:integration_test/integration_test.dart';
import 'package:smarge/main.dart' as app;

void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();
  
  testWidgets('Complete optimization flow', (tester) async {
    app.main();
    await tester.pumpAndSettle();
    
    // Tap optimize button
    await tester.tap(find.text('Optimize Now'));
    await tester.pumpAndSettle();
    
    // Verify schedule appears
    expect(find.text('Schedule'), findsOneWidget);
  });
}
```

**Run integration tests:**
```powershell
flutter test integration_test/app_test.dart
```

---

## Useful Commands Cheat Sheet

```powershell
# Development
flutter run                    # Run in debug mode
flutter run --release          # Run in release mode
flutter run -d <device>        # Run on specific device
flutter hot-reload             # Press 'r' in running app
flutter hot-restart            # Press 'R' in running app

# Testing
flutter test                   # Run unit tests
flutter test --coverage        # With coverage report
flutter drive --target=integration_test/app_test.dart

# Code Quality
flutter analyze                # Lint
dart format .                  # Format
dart fix --apply               # Auto-fix

# Build
flutter build ios              # iOS build
flutter build apk              # Android APK
flutter build appbundle        # Android App Bundle

# Clean
flutter clean                  # Clean build artifacts
flutter pub cache clean        # Clear package cache
rm -rf ios/Pods ios/Podfile.lock
pod install --repo-update      # Reinstall iOS pods

# Debugging
flutter doctor                 # Check setup
flutter doctor -v              # Verbose output
flutter devices                # List devices
flutter logs                   # View device logs
```

---

## Next Steps

After successful setup:

1. **Review Architecture**: Read `specs/main/data-model.md`
2. **Understand APIs**: Read contract files in `specs/main/contracts/`
3. **Explore Codebase**: Start with `lib/main.dart`
4. **Run Tests**: `flutter test` to verify everything works
5. **Start Development**: Pick a task from `specs/main/tasks.md` (after `/speckit.tasks` creates it)

---

## Getting Help

- **Flutter Docs**: https://docs.flutter.dev/
- **Isar Database**: https://isar.dev/
- **Riverpod**: https://riverpod.dev/
- **Project Issues**: GitHub Issues in this repo
- **Tibber API**: https://developer.tibber.com/docs/overview

---

## Contributing

See `CONTRIBUTING.md` for:
- Code style guidelines
- PR process
- Testing requirements
- Commit message format
