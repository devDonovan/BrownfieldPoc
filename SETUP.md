# BrownfieldPoc - Quick Setup Guide

## 🚀 30-Minute Quick Start

### Prerequisites Check
```bash
# Node.js 22.11.0+
node -v

# JDK 17+
java -version

# Xcode + CocoaPods
xcode-select --install
sudo gem install cocoapods
```

### Step 1: Install React Native Dependencies (5 min)
```bash
cd FlyDubai
npm install
```

### Step 2: Android Setup (10 min)

```bash
cd ../architecture-samples

# Update gradle settings (already configured)
# Just sync gradle in Android Studio or:
./gradlew sync

# Run on emulator
./gradlew run
# OR in Android Studio: Run → Select FlyDubai as target
```

**Testing Android:**
- Open Settings app → Navigate using React Navigation (Dashboard → Profile → Transactions)
- Tap "Open Native Profile" to test bridge (opens native ProfileActivity)
- Tap back to return to RN

### Step 3: iOS Setup (10 min)

```bash
cd ../sample-food-truck/ios

# Install React Native pods
pod install --repo-update

cd ..

# Open and run in Xcode
open FoodTruck.xcworkspace
# Select FoodTruck target
# Build and Run (Cmd + R)
```

**Testing iOS:**
- Navigate through React Navigation screens (Dashboard → Profile → Transactions)
- Tap "Open Native Profile" to test bridge
- Observe native header updating as you navigate

### Step 4: Verify Integration

✅ **Android Checklist:**
- [ ] App launches with RN content below native Toolbar
- [ ] React Navigation works (Dashboard → Profile → Transactions)
- [ ] Native header updates when switching screens
- [ ] "Open Native Profile" button opens ProfileActivity
- [ ] Back button works from native activity
- [ ] Logout closes RN view

✅ **iOS Checklist:**
- [ ] App launches with RN content below UINavigationBar
- [ ] React Navigation works (Dashboard → Profile → Transactions)
- [ ] Native header updates when switching screens
- [ ] "Open Native Profile" button opens native screen
- [ ] Back button (swipe or native button) works
- [ ] Logout closes RN view

---

## 📦 What's Included

### React Native App (FlyDubai)
- ✅ Redux Toolkit store with auth & navigation slices
- ✅ React Navigation 7.x (no header - using native)
- ✅ 3 Sample screens: Dashboard, Profile, Transactions
- ✅ Bridge utils for native communication
- ✅ i18n config (en, ar, ru) with RTL support
- ✅ Design tokens (theme.json) for theming

### Android Integration (architecture-samples)
- ✅ RNContainerActivity.kt - Host activity with native Toolbar
- ✅ NavigationBridgeModule.kt - Native module for RN bridge
- ✅ NavigationBridgePackage.kt - Package provider
- ✅ ProfileActivity.kt - Demo native screen
- ✅ TodoApplication.kt - Updated with ReactApplication
- ✅ Updated settings.gradle.kts to include FlyDubai module

### iOS Integration (sample-food-truck)
- ✅ RNHostViewController.swift - Host VC with UINavigationBar
- ✅ RNBridgeDelegate.swift - iOS bridge implementation
- ✅ RNAppDelegate.swift - React Native app delegate
- ✅ Podfile - React Native and navigation pods
- ✅ Updated to support React Native embedding

---

## 🔧 Key Files to Understand

### For Developers Working on RN
- `FlyDubai/App.tsx` - Redux provider + main entry
- `FlyDubai/src/store/index.ts` - Redux store configuration
- `FlyDubai/src/bridge/bridgeUtils.ts` - Bridge communication API
- `FlyDubai/src/navigation/NavigationStack.tsx` - React Navigation setup
- `FlyDubai/src/screens/*.tsx` - Sample screens

### For Android Developers
- `architecture-samples/app/src/main/java/.../RNContainerActivity.kt` - Main integration point
- `architecture-samples/app/src/main/java/.../bridge/NavigationBridgeModule.kt` - Bridge module
- `architecture-samples/settings.gradle.kts` - FlyDubai module inclusion

### For iOS Developers
- `sample-food-truck/App/RNHostViewController.swift` - Main integration point
- `sample-food-truck/App/RNBridgeDelegate.swift` - Bridge implementation
- `sample-food-truck/ios/Podfile` - Pod dependencies

---

## 📱 Testing the Bridge

### Test Native→RN Communication (Passing Auth)
1. Launch RNContainerActivity/RNHostViewController with userId and authToken
2. DashboardScreen receives props and displays greeting with userName
3. Redux state is updated with auth info

### Test RN→Native Communication
1. In DashboardScreen, tap "Open Native Profile"
2. bridgeUtils.openNativeScreen() calls NavigationBridge native module
3. Native ProfileActivity opens with userId parameter
4. Verify back button returns to RN

### Test Header Updates
1. Navigate between screens
2. Each screen calls `bridgeUtils.updateHeader(title)`
3. Verify native toolbar/navigation bar title updates
4. Check Redux `navigation.headerTitle` is synced

---

## 🎯 Next Steps After Setup

### 1. **Connect Real Auth**
- Replace mock auth in DashboardScreen
- Integrate with your native auth system
- Pass real userId/authToken to RNContainerActivity

### 2. **Add More Native Screens**
- Create new activities (Android) or view controllers (iOS)
- Register in NavigationBridgeModule/RNBridgeDelegate
- Navigate from RN via bridgeUtils.openNativeScreen()

### 3. **Setup Data Persistence**
- Install redux-persist
- Configure serialization (for authToken)
- Consider encrypted storage for sensitive data

### 4. **Add Analytics**
- Emit navigation events from RN
- Log via bridge to native analytics systems
- Track screen views and user actions

### 5. **Implement Deep Linking**
- Setup URL schemes/routing
- Add navigation listeners in React Navigation
- Handle deep links from native notifications

---

## 🐛 Common Issues & Fixes

| Issue | Fix |
|-------|-----|
| Metro bundler won't start | `npm start --reset-cache` |
| Android build fails | `./gradlew clean && ./gradlew sync` |
| Pods won't install | `pod cache clean --all && pod install --repo-update` |
| Bridge module not found | Check package registration in native code |
| RN content not showing | Verify ReactRootView frame and hierarchy |
| Header not updating | Ensure bridgeUtils.updateHeader() is called |

---

## 📖 Documentation

For detailed architecture, security practices, and API reference:
→ See **[README.md](./README.md)**

For code examples and implementation details:
→ Review source files in respective folders

---

## ✨ Architecture Highlights

- **Monorepo Structure**: Android, iOS, and RN in single repo
- **Native Headers**: Preserve native UX at top of each screen
- **Bidirectional Bridge**: NativeModules + DeviceEventEmitter
- **Redux State**: Shared state for auth, navigation, and app data
- **i18n Ready**: Multi-language support (en, ar, ru) with RTL
- **Security First**: PII-safe data passing, no sensitive logging
- **Scalable**: Easy to add native screens and RN features

---

## 🎉 You're Ready!

Your monorepo is configured and ready for:
- Cross-platform native+RN development
- Secure data exchange between layers
- Internationalized content
- Consistent theming
- Scalable feature development

**Happy coding! 🚀**
