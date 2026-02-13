# Architecture & Integration Details

## System Architecture

### Monorepo Layout
```
BrownfieldPoc (Root)
├── FlyDubai/               React Native 0.84 app
├── architecture-samples/   Android native app
├── sample-food-truck/      iOS native app
├── README.md              Complete project documentation
├── SETUP.md               Quick setup guide
└── ARCHITECTURE.md        This file
```

---

## Component Interactions

### 1. **Application Startup Flow**

#### Android Path
```
1. User launches Android app
   ↓
2. MainActivity or other native activity
   ↓
3. Call intent to open RNContainerActivity
   Intent(context, RNContainerActivity::class.java)
   .putExtra(KEY_USER_ID, userId)
   .putExtra(KEY_AUTH_TOKEN, authToken)
   ↓
4. RNContainerActivity.onCreate()
   ├─ setupToolbar()           // Native header
   ├─ setupReactNativeContainer()
   └─ launchReactNativeApp()
   ↓
5. ReactInstanceManager initialized
   ├─ Create ReactRootView
   ├─ Pass initialProps
   └─ Add to FrameLayout container
   ↓
6. React Native app loads
   ├─ Redux provider wraps app
   ├─ i18n initialized
   └─ Navigation stack ready
```

#### iOS Path
```
1. User navigates in native app
   ↓
2. Some native action triggers RN
   ↓
3. Create RNHostViewController
   let rnVC = RNHostViewController(
       userId: userId,
       authToken: authToken,
       userName: userName
   )
   ↓
4. Push to navigation stack
   navigationController?.pushViewController(rnVC, animated: true)
   ↓
5. RNHostViewController.viewDidLoad()
   ├─ setupNavigationBar()
   ├─ setupReactNativeContainer()
   └─ Create RCTRootView with initialProps
   ↓
6. React Native app loads (same as Android)
```

---

### 2. **React Navigation Flow (No RN Header)**

```
App (Redux Provider)
  ↓
NavigationStack (NavigationContainer)
  ├─ screenOptions: { headerShown: false }  // ← Key: hiding RN header
  ↓
Stack.Navigator
  ├─ DashboardScreen
  │  └─ useEffect: bridgeUtils.updateHeader('Dashboard')
  ├─ ProfileScreen
  │  └─ useEffect: bridgeUtils.updateHeader('Profile')
  └─ TransactionsScreen
     └─ useEffect: bridgeUtils.updateHeader('Transactions')
```

**Why `headerShown: false`?**
- Native app owns the header (Toolbar/UINavigationBar)
- RN header would overlap or duplicate native header
- Updates coordinated via bridge when RN screen changes

---

### 3. **Redux State Management**

#### Store Structure
```typescript
store = configureStore({
  reducer: {
    auth: authReducer,
    navigation: navigationReducer
  },
  middleware: [thunk, logger] // Redux-thunk for async actions
})
```

#### Redux Slices

**Auth Slice:** Manages authentication state
```typescript
state = {
  isAuthenticated: boolean,
  userId: string | null,
  authToken: string | null,    // Never logged!
  userName: string | null,
  error: string | null,
  loading: boolean
}

actions = {
  setAuthenticated({ userId, authToken, userName }),
  logout(),
  setError(message),
  setLoading(boolean)
}
```

**Navigation Slice:** Manages header state (synced with native)
```typescript
state = {
  headerTitle: string,
  headerRightButton: string | null,
  currentScreen: string
}

actions = {
  setHeaderTitle(title),
  setHeaderRightButton(label),
  setCurrentScreen(name)
}
```

#### Data Flow Example
```
Screen renders
  ↓
useEffect triggers
  ↓
dispatch(setHeaderTitle('New Title'))  // Redux action
  ↓
Redux state updated
  ↓
Subscription triggers
  ↓
Component re-renders (if subscribed)
  ↓
bridgeUtils.updateHeader('New Title')  // Also call bridge
  ↓
Native.NativeModules.NavigationBridge.updateHeader()
  ↓
RNContainerActivity.updateHeaderTitle()
  ↓
Native Toolbar title updates ✓
```

---

### 4. **Bridge Communication Layer**

#### Type 1: RN → Native (Call & Return)

**Process:**
```
1. RN code calls NativeModules.NavigationBridge.methodName()
2. Marshalling: JavaScript values → Native types (Bundle, ReadableMap)
3. Native module method executes
4. Promise.resolve() returns to JavaScript
5. RN code receives result/error

Example: Navigate to native screen
```

**Android Implementation:**
```kotlin
@ReactMethod
fun openNativeScreen(screenName: String, params: ReadableMap?, promise: Promise) {
    try {
        val intent = Intent(currentActivity, getActivityClass(screenName))
        intent.putExtras(params.toBundle())  // Safe conversion
        currentActivity.startActivity(intent)
        promise.resolve(true)
    } catch (e: Exception) {
        promise.reject("ERROR", e.message)
    }
}
```

**iOS Implementation:**
```swift
@objc(openNativeScreen:params:resolver:rejecter:)
func openNativeScreen(
    screenName: String,
    params: [String: Any]?,
    resolver: @escaping RCTPromiseResolveBlock,
    rejecter: @escaping RCTPromiseRejectBlock
) {
    RNBridgeDelegate.shared.openNativeScreen(screenName: screenName, params: params)
    resolver(true)
}
```

**React Native Usage:**
```typescript
import { NativeModules } from 'react-native';

const { NavigationBridge } = NativeModules;

NavigationBridge.openNativeScreen('ProfileActivity', { userId: '123' })
    .then(() => console.log('Success'))
    .catch(err => console.error('Failed', err));
```

#### Type 2: Native → RN (Events)

**Process:**
```
1. Native code emits event via DeviceEventEmitter
2. Event carries JSON payload
3. RN subscribes to event via DeviceEventEmitter listener
4. Listener callback fires with event data
5. RN can dispatch Redux action with data

Example: User logged in from native, notify RN
```

**Android Emit:**
```kotlin
DeviceEventManagerModule.RCTDeviceEventEmitter eventEmitter = 
    reactInstanceManager.getCurrentReactContext()
        .getJSModule(DeviceEventManagerModule.RCTDeviceEventEmitter.class);

eventEmitter.emit("user_authenticated", mapOf(
    "userId" to userId,
    "authToken" to authToken  // Don't log this!
));
```

**iOS Emit:**
```swift
let bridge = RCTBridge(delegate: self, launchOptions: nil)
let eventEmitter = bridge.module(for: RCTDeviceEventEmitter.self) as? RCTDeviceEventEmitter
eventEmitter?.emit("user_authenticated", withBody: [
    "userId": userId,
    "authToken": authToken
])
```

**React Native Listen:**
```typescript
import { DeviceEventEmitter } from 'react-native';

useEffect(() => {
    const unsubscribe = DeviceEventEmitter.addListener(
        'user_authenticated',
        (data) => {
            dispatch(setAuthenticated({
                userId: data.userId,
                authToken: data.authToken
            }));
        }
    );

    return unsubscribe;
}, [dispatch]);
```

---

## Security Considerations

### PII Data Handling

**❌ UNSAFE PRACTICES:**
```typescript
// Never log auth tokens
console.log('Token:', authToken);

// Never pass in URL
navigation.navigate('Screen?token=' + authToken);

// Never store in Redux without encryption
state.sensitiveToken = rawToken;

// Never embed in error messages
throw new Error(`Auth failed: ${token}`);
```

**✅ SAFE PRACTICES:**
```typescript
// Pass via secure bridge
bridgeUtils.openNativeScreen('ProfileActivity', { userId });

// Log safely
console.log('Auth state:', isAuthenticated);

// Store in secure storage
secureStore.set('authToken', authToken);

// Use indices for data
const userIndex = 0;  // Instead of userId in logs
```

### Data in Transit

**Bridge Call Guarantees:**
- ✅ Passed via method calls (not URL)
- ✅ Bundle/ReadableMap marshalling (JSON under hood)
- ✅ Not exposed to JavaScript debugger by default
- ✅ Platform-native memory (not easily introspectable)

**Recommendations:**
1. Keep Redux middleware clean (strip sensitive fields)
2. Use redux-persist only with encryption
3. Clear auth state on app close
4. Never serialize tokens to disk unencrypted

---

## Extension Points

### Adding New Native Screens

**Step 1: Create native activity/VC**
```kotlin
// Android
class MyNewActivity : AppCompatActivity() { ... }
```

```swift
// iOS
class MyNewViewController: UIViewController { ... }
```

**Step 2: Register in bridge**
```kotlin
// Android: NavigationBridgeModule.kt
val intent = when (screenName) {
    "MyNewScreen" -> Intent(currentActivity, MyNewActivity::class.java)
    else -> { ... }
}
```

```swift
// iOS: RNBridgeDelegate.swift
switch screenName {
case "MyNewScreen":
    openMyNewScreen(with: params)
default: { ... }
}
```

**Step 3: Call from RN**
```typescript
bridgeUtils.openNativeScreen('MyNewScreen', { /* params */ });
```

### Adding New Redux Slices

**Step 1: Create slice**
```typescript
// src/store/slices/mySlice.ts
const mySlice = createSlice({...});
```

**Step 2: Add to store**
```typescript
// src/store/index.ts
configureStore({
  reducer: {
    auth,
    navigation,
    mySlice  // ← Add here
  }
});
```

**Step 3: Use in components**
```typescript
const myData = useSelector((state) => state.mySlice);
dispatch(mySliceAction(payload));
```

### Adding New Languages

**Step 1: Create translation file**
```json
// src/i18n/fr.json
{
  "dashboard": "Tableau de bord",
  ...
}
```

**Step 2: Register in i18n**
```typescript
// src/i18n/config.ts
import fr from './fr.json';

const resources = {
  en, ar, ru, fr  // ← Add here
};
```

**Step 3: Use in components**
```typescript
const { t } = useTranslation();
<Text>{t('dashboard')}</Text>  // "Tableau de bord" in French
```

---

## Performance Optimization

### Native ↔ RN Bridge
- Calls are **synchronous when possible** (no bridge overhead)
- For heavy operations, use **async callbacks**
- Batch multiple state updates into single bridge call

### Redux State
- Selectors memoize computations
- Avoid creating new objects in reducers
- Use redux-thunk for async operations

### React Navigation
- Lazy load screens
- Use navigation.reset() instead of multiple pops
- Enable native header (reduces RN render overhead)

### Native Code
- Keep bridge calls lightweight
- Offload heavy work to background threads
- Cache frequently accessed data

---

## Testing Strategy

### Unit Tests
```bash
cd FlyDubai
npm test

# Test Redux reducers
npm test -- authSlice.test.ts

# Test bridge utilities
npm test -- bridgeUtils.test.ts
```

### Integration Tests
- Launch RNContainerActivity with mocked service
- Verify React Native renders
- Test bridge method calls
- Verify native header updates

### Manual QA Checklist
- [ ] App launches with correct initial state
- [ ] Navigation works (RN screens)
- [ ] Bridge calls work (open native screen)
- [ ] Header updates dynamically
- [ ] Back button works
- [ ] Logout/cleanup works
- [ ] Works in multiple languages
- [ ] No sensitive data in logs

---

## Debugging Tools

### Android
- **Android Studio Debugger**: Set breakpoints in Kotlin code
- **Logcat**: `adb logcat | grep RNContainerActivity`
- **React DevTools**: Chrome DevTools for JS

### iOS
- **Xcode Debugger**: Set breakpoints in Swift code
- **Xcode Console**: View NSLog output
- **React DevTools**: Safari Web Inspector for JS

### Both
- **React Native Debugger**: Standalone app for JS debugging
- **Redux DevTools**: Time-travel debugging
- **Network Monitor**: Check HTTP requests

---

## Migration Checklist

### Before Integrating into Production App

- [ ] Security audit on bridge calls
- [ ] PII data handling review
- [ ] Offline/online state management
- [ ] Error handling and fallbacks
- [ ] Permissions handling
- [ ] Testing on real devices
- [ ] Performance baseline
- [ ] Analytics integration
- [ ] Crash reporting setup
- [ ] A/B testing infrastructure

---

## Troubleshooting

### React Native Won't Load
```
1. Check Metro bundler running: npm start
2. Verify bundle path in native code
3. Check initialProps structure
4. Enable dev build in native
```

### Bridge Method Not Found
```
1. Verify native module registered
2. Check method name matches RN call
3. Rebuild native app
4. Check AndroidManifest.xml permissions
```

### Header Not Updating
```
1. Verify bridgeUtils.updateHeader() called
2. Check Redux dispatch
3. Verify activity/VC implements update method
4. Check native thread (use runOnUiThread/DispatchQueue.main)
```

---

## References & Resources

- React Native Embedding: https://reactnative.dev/docs/integration-with-existing-apps
- Redux Toolkit: https://redux-toolkit.js.org
- React Navigation: https://reactnavigation.org
- i18next: https://www.i18next.com
- Android Bridges: https://reactnative.dev/docs/native-modules-android
- iOS Bridges: https://reactnative.dev/docs/native-modules-ios

---

**Last Updated**: February 2026
**React Native Version**: 0.84
**Minimum Android SDK**: 24 (API 24)
**Minimum iOS Version**: 15.0
