# BrownfieldPoc - React Native Integration Monorepo

A comprehensive guide for integrating React Native 0.84 into existing native Android (Java) and iOS (Swift) apps using a monorepo structure. This POC demonstrates native header preservation with React Native content rendering below it, bidirectional communication between native and RN, and state management with Redux Toolkit.

## 📋 Project Structure

```
BrownfieldPoc/
├── architecture-samples/        # Native Android app (Java + Compose)
│   ├── app/
│   │   ├── src/main/
│   │   │   ├── java/.../RNContainerActivity.kt    # RN host activity with native Toolbar
│   │   │   ├── java/.../bridge/NavigationBridgeModule.kt  # Native bridge module
│   │   │   └── res/layout/activity_rn_container.xml
│   │   └── build.gradle.kts
│   └── settings.gradle.kts      # Includes FlyDubai as module
│
├── sample-food-truck/           # Native iOS app (SwiftUI + UIKit)
│   ├── App/
│   │   ├── App.swift            # Main app entry
│   │   ├── RNHostViewController.swift  # RN host view controller
│   │   ├── RNBridgeDelegate.swift     # iOS bridge implementation
│   │   └── RNAppDelegate.swift        # React Native app delegate
│   ├── ios/
│   │   └── Podfile              # React Native pod dependencies
│   └── Food Truck.xcodeproj/
│
└── FlyDubai/                    # React Native app (0.84)
    ├── App.tsx                  # Main app with Redux provider
    ├── src/
    │   ├── store/
    │   │   ├── index.ts         # Redux store configuration
    │   │   ├── slices/
    │   │   │   ├── authSlice.ts
    │   │   │   └── navigationSlice.ts
    │   ├── navigation/
    │   │   └── NavigationStack.tsx  # React Navigation setup (no header)
    │   ├── screens/
    │   │   ├── DashboardScreen.tsx
    │   │   ├── ProfileScreen.tsx
    │   │   └── TransactionsScreen.tsx
    │   ├── bridge/
    │   │   └── bridgeUtils.ts   # Native-to-RN and RN-to-native bridge
    │   ├── i18n/
    │   │   ├── config.ts        # i18next configuration
    │   │   ├── en.json, ar.json, ru.json
    │   └── index.ts             # Entry point
    ├── design-tokens/
    │   └── theme.json           # Design tokens for theming
    ├── package.json
    └── index.js                 # React Native entry
```

## 🚀 Quick Start

### Prerequisites

- **Android**: Android Studio, JDK 17+, compileSdk 35+, minSdk 24+
- **iOS**: Xcode 15+, iOS 15.0+, CocoaPods
- **Node.js**: 22.11.0 or higher
- **React Native CLI**: `npm install -g @react-native-community/cli`

### Initial Setup

#### 1. **Install React Native Dependencies**

```bash
cd FlyDubai
npm install
npm install @react-navigation/native @react-navigation/native-stack \
  @reduxjs/toolkit react-redux redux-thunk \
  i18next react-i18next react-native-localize
```

#### 2. **Android Setup**

```bash
# Update build.gradle.kts and settings.gradle.kts (already configured)
cd architecture-samples

# Sync Gradle
./gradlew sync

# Run on Android device/emulator
./gradlew run
```

**Key Components:**
- **RNContainerActivity.kt**: Hosts React Native content below a native Toolbar
- **NavigationBridgeModule.kt**: Exposes native APIs to React Native
- **NavigationBridgePackage.kt**: Registers the bridge module

#### 3. **iOS Setup**

```bash
# Install pods
cd sample-food-truck/ios
pod install --repo-update
cd ..

# Open workspace
open FoodTruck.xcworkspace

# Run on iPhone simulator (Xcode)
```

**Key Components:**
- **RNHostViewController.swift**: Hosts React Native with UINavigationBar
- **RNBridgeDelegate.swift**: iOS bridge for native-RN communication
- **RNAppDelegate.swift**: React Native app delegate integration

---

## 🏗️ Architecture

### Native Header Strategy

Both platforms maintain a **native header** with React Native content rendering below:

**Android:**
```
┌─────────────────────────┐
│  Native Toolbar         │  ← RNContainerActivity
├─────────────────────────┤
│                         │
│  React Native Content   │  ← RCTRootView
│  (Dashboard, etc.)      │
│                         │
└─────────────────────────┘
```

**iOS:**
```
┌─────────────────────────┐
│  UINavigationBar        │  ← RNHostViewController
├─────────────────────────┤
│                         │
│  React Native Content   │  ← RCTRootView
│  (Dashboard, etc.)      │
│                         │
└─────────────────────────┘
```

### State Management

**Redux Toolkit** with organized slices:

```
Redux Store
├── auth (userId, authToken, isAuthenticated)
├── navigation (headerTitle, headerRightButton, currentScreen)
└── middleware: redux-thunk for async actions
```

Example reducer:
```typescript
// src/store/slices/authSlice.ts
const authSlice = createSlice({
  name: 'auth',
  initialState,
  reducers: {
    setAuthenticated: (state, action) => { ... },
    logout: (state) => { ... }
  }
});
```

### Bidirectional Communication

**Native → React Native** (via DeviceEventEmitter):
```typescript
// React Native listener
bridgeUtils.onNativeEvent('user_authenticated', (data) => {
  dispatch(setAuthenticated(data));
});
```

**React Native → Native** (via NativeModules):
```typescript
// React Native
bridgeUtils.openNativeScreen('ProfileActivity', { userId: '123' });
bridgeUtils.updateHeader('New Title', 'Settings');
```

---

## 📱 Navigation Flow

### 1. Initial Launch (Native)

**Android:**
```kotlin
val intent = Intent(context, RNContainerActivity::class.java).apply {
    putExtra(RNContainerActivity.KEY_USER_ID, userId)
    putExtra(RNContainerActivity.KEY_AUTH_TOKEN, authToken)
    putExtra(RNContainerActivity.KEY_USER_NAME, userName)
}
startActivity(intent)
```

**iOS:**
```swift
let rnVC = RNHostViewController(userId: userId, authToken: authToken, userName: userName)
let navVC = UINavigationController(rootViewController: rnVC)
navigationController?.pushViewController(navVC, animated: true)
```

### 2. In React Native

```typescript
// DashboardScreen.tsx
const DashboardScreen = ({ navigation }) => {
  // Navigate within RN screens
  const handleViewProfile = () => {
    navigation?.navigate('ProfileScreen');  // React Navigation
  };

  // Navigate back to native
  const handleOpenNativeProfile = () => {
    bridgeUtils.openNativeScreen('ProfileActivity', { userId });
  };

  // Update native header
  useEffect(() => {
    bridgeUtils.updateHeader('Dashboard', 'Settings');
  }, []);
};
```

### 3. Dynamic Header Updates

```typescript
// From any RN screen
dispatch(setHeaderTitle('My Screen'));
bridgeUtils.updateHeader('My Screen');  // Updates native header
```

---

## 🔄 Bridge Communication

### Bridge Utilities (React Native)

**File:** `src/bridge/bridgeUtils.ts`

```typescript
// Navigate to native screen
bridgeUtils.openNativeScreen('ProfileActivity', { userId: '123' });

// Update native header
bridgeUtils.updateHeader('New Title', 'ButtonLabel');

// Close RN and go back to native
bridgeUtils.closeRNView();

// Listen for native events
const unsubscribe = bridgeUtils.onNativeEvent('eventName', (data) => {
  console.log('Event from native:', data);
});

// Emit to native
bridgeUtils.emitToNative('eventName', { key: 'value' });
```

### Native Modules

**Android:** `NavigationBridgeModule.kt`
```kotlin
@ReactMethod
fun openNativeScreen(screenName: String, params: ReadableMap?, promise: Promise)

@ReactMethod
fun updateHeader(title: String, rightButtonLabel: String?, promise: Promise)

@ReactMethod
fun closeRNView(promise: Promise)
```

**iOS:** `RNNavigationBridge` (Swift)
```swift
@objc(openNativeScreen:params:resolver:rejecter:)
func openNativeScreen(screenName: String, params: [String: Any]?, ...)

@objc(updateHeader:rightButtonLabel:resolver:rejecter:)
func updateHeader(title: String, rightButtonLabel: String?, ...)
```

---

## 🌍 Internationalization (i18n)

**Supported Languages:**
- English (en)
- Arabic (ar) - RTL support enabled
- Russian (ru)

**Setup:**
```typescript
// src/i18n/config.ts
i18n.use(initReactI18next).init({
  resources: { en, ar, ru },
  lng: detectDeviceLanguage(),
  fallbackLng: 'en'
});
```

**Usage in Components:**
```typescript
import { useTranslation } from 'react-i18next';

const { t } = useTranslation();
<Text>{t('dashboard')}</Text>
```

**Translation Files:**
- `src/i18n/en.json`
- `src/i18n/ar.json`
- `src/i18n/ru.json`

---

## 🎨 Theming

**Design Tokens:** `design-tokens/theme.json`

Contains:
```json
{
  "colors": { primary, secondary, accent, background, ... },
  "spacing": { xs, sm, md, lg, xl },
  "typography": { headingLarge, bodyMedium, ... },
  "borderRadius": { sm, md, lg, xl, full }
}
```

**Usage:**
```typescript
import theme from '../../design-tokens/theme.json';

<View style={{ backgroundColor: theme.colors.primary }} />
```

**Future:** Native will consume the same JSON for platform consistency.

---

## 🔐 Security Best Practices

### PII Handling

✅ **DO:**
- Pass userId and authToken via initialProps (not logged)
- Use secure bridges for sensitive data
- Store tokens in native Keychain/KeyStore
- Emit events without logging sensitive keys

❌ **DON'T:**
- Log authToken or password in console
- Pass raw payment data through bridge
- Store sensitive data in Redux without encryption
- Expose PII in error messages

### Example - Safe Data Passing

**Android:**
```kotlin
val initialProps = Bundle().apply {
    // Never log these
    putString("userId", userId)
    putString("authToken", authToken)
}
reactRootView.startReactApplication(reactInstanceManager, "FlyDubai", initialProps)
```

---

## 📋 Development Workflow

### 1. **Metro Bundler (Development)**

```bash
cd FlyDubai
npm start

# In another terminal
npm run android  # or npm run ios
```

### 2. **Hot Reload**

- Changes in RN code reload instantly
- Native changes require rebuild

### 3. **Debugging**

**Android:**
```bash
adb shell input keyevent 82  # Menu button
# Select "Debug JS Remotely" → Chrome DevTools
```

**iOS:**
```
Cmd + D in simulator → Debug JS Remotely
```

### 4. **Testing**

```bash
cd FlyDubai
npm test

# For specific file
npm test -- DashboardScreen.tsx
```

---

## 🛠️ Common Tasks

### Add a New Redux Slice

```typescript
// src/store/slices/yourSlice.ts
const yourSlice = createSlice({
  name: 'yourSlice',
  initialState,
  reducers: { ... }
});

// Add to store
// src/store/index.ts
configureStore({
  reducer: {
    auth, navigation, yourSlice  // ← Add here
  }
});
```

### Add a New Screen

1. Create `src/screens/NewScreen.tsx`
2. Add to `NavigationStack.tsx`
3. Navigate via `navigation.navigate('NewScreen')`

### Update Native Header Dynamically

```typescript
useEffect(() => {
  dispatch(setHeaderTitle('My New Title'));
  bridgeUtils.updateHeader('My New Title', 'Action');
}, [dispatch]);
```

### Emit Native Event from React Native

```typescript
bridgeUtils.emitToNative('payment_completed', { 
  transactionId: '12345',
  amount: 100
});

// Listen in native (Android)
BroadcastReceiver or event listener
```

---

## 📚 Deliverables Checklist

- ✅ **Android Integration**: RNContainerActivity with native Toolbar
- ✅ **iOS Integration**: RNHostViewController with UINavigationBar
- ✅ **Redux Toolkit**: Configured with thunk middleware
- ✅ **Bidirectional Bridge**: NativeModules & DeviceEventEmitter
- ✅ **Sample Navigation**: Dashboard → Profile → Transactions → Native
- ✅ **i18n Scaffold**: English, Arabic, Russian translations
- ✅ **Theme Design Tokens**: Centralized JSON for RN (native later)
- ✅ **Security Guards**: PII handling best practices
- ✅ **Documentation**: This README with setup and architecture

---

## 🚫 Out of Scope (For Future)

- **Deep Linking**: Will require URL routing in React Navigation and native intent filters
- **CI/CD**: Will add GitHub Actions for build and test automation
- **OTA Updates**: Will integrate CodePush or Hermes for over-the-air updates
- **Encrypted Storage**: Will add redux-persist with encryption for sensitive data
- **Analytics**: Will integrate event tracking for native and RN
- **Push Notifications**: Will integrate FCM and APNs

---

## 🐛 Troubleshooting

### Metro Bundler Not Starting

```bash
npm start --reset-cache
```

### Gradle Build Failures (Android)

```bash
cd architecture-samples
./gradlew clean
./gradlew sync
```

### Pod Installation Issues (iOS)

```bash
cd sample-food-truck/ios
pod cache clean --all
pod install --repo-update
```

### RN Module Not Found

Ensure `NavigationBridgePackage` is registered in `MainApplication.kt` (Android) or `RNAppDelegate` (iOS).

### Header Not Updating

- Verify `bridgeUtils.updateHeader()` is called
- Check Redux dispatch is firing
- Confirm native module is linked

---

## 📖 References

- [React Native Official Docs](https://reactnative.dev)
- [React Navigation](https://reactnavigation.org)
- [Redux Toolkit](https://redux-toolkit.js.org)
- [i18next Documentation](https://www.i18next.com)
- [Android Embedding RN](https://reactnative.dev/docs/integration-with-existing-apps)
- [iOS Embedding RN](https://reactnative.dev/docs/integration-with-existing-apps)

---

## 📝 License

Licensed under the Apache License, Version 2.0. See individual files for license headers.

---

## 👥 Contributors

- Architecture design and implementation for brownfield RN integration
- Security-first approach with PII handling
- Comprehensive bridge implementation for native↔RN communication
- i18n and theming scaffold for future platform consistency

**Questions or Issues?** Review architecture samples and bridge implementations in source files.
