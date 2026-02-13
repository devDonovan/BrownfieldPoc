# Quick Reference - React Native Integration

## 📁 Project Structure

```
BrownfieldPoc/
├── README.md                 ← Start here for overview
├── SETUP.md                  ← 30-min quick start
├── ARCHITECTURE.md           ← Deep dive into design
├── DELIVERABLES.md          ← Implementation checklist
│
├── FlyDubai/                 React Native 0.84 app
│   ├── src/
│   │   ├── store/            Redux Toolkit + Thunk
│   │   ├── navigation/       React Navigation (headerShown: false)
│   │   ├── screens/          3 sample screens
│   │   ├── bridge/           Native communication
│   │   └── i18n/             Translations (en, ar, ru)
│   ├── design-tokens/        Theming (theme.json)
│   ├── App.tsx              Redux + SafeArea + i18n
│   ├── package.json         Dependencies
│   └── index.js             Entry point
│
├── architecture-samples/     Android app (Kotlin + Compose)
│   ├── app/src/main/
│   │   ├── java/.../RNContainerActivity.kt        ← Main integration
│   │   ├── java/.../bridge/NavigationBridgeModule.kt
│   │   ├── java/.../bridge/NavigationBridgePackage.kt
│   │   ├── java/.../ProfileActivity.kt            ← Demo native screen
│   │   └── java/.../TodoApplication.kt
│   ├── app/src/main/res/layout/activity_rn_container.xml
│   ├── app/build.gradle.kts  (React Native deps added)
│   └── settings.gradle.kts   (FlyDubai module included)
│
└── sample-food-truck/        iOS app (Swift + SwiftUI)
    ├── App/
    │   ├── RNHostViewController.swift   ← Main integration
    │   ├── RNBridgeDelegate.swift       ← Bridge implementation
    │   └── RNAppDelegate.swift
    └── ios/
        └── Podfile                      (React Native pods)
```

---

## 🔑 Key Files You'll Work With

### React Native Development
```
FlyDubai/src/store/index.ts          Redux store
FlyDubai/src/screens/DashboardScreen.tsx  Example screen
FlyDubai/src/bridge/bridgeUtils.ts   Native calls API
FlyDubai/src/i18n/config.ts         Language setup
```

### Android Native Development
```
architecture-samples/app/src/main/java/.../RNContainerActivity.kt
architecture-samples/app/src/main/java/.../bridge/NavigationBridgeModule.kt
```

### iOS Native Development
```
sample-food-truck/App/RNHostViewController.swift
sample-food-truck/App/RNBridgeDelegate.swift
sample-food-truck/ios/Podfile
```

---

## 💻 Common Commands

### React Native
```bash
cd FlyDubai
npm start                    # Start Metro bundler
npm run android             # Run on Android
npm run ios                 # Run on iOS
npm test                    # Run tests
```

### Android
```bash
cd architecture-samples
./gradlew sync              # Sync dependencies
./gradlew build             # Build APK
./gradlew run               # Install and run
```

### iOS
```bash
cd sample-food-truck
cd ios && pod install       # Install pods
cd ..
open FoodTruck.xcworkspace # Open in Xcode
# Cmd + R to run
```

---

## 🎯 Core Concepts

### 1. Redux Store Structure
```typescript
store = {
  auth: {
    isAuthenticated: boolean
    userId: string
    authToken: string      // NEVER logged
    userName: string
  },
  navigation: {
    headerTitle: string
    headerRightButton: string | null
    currentScreen: string
  }
}
```

### 2. Navigation Flow
```
DashboardScreen → ProfileScreen → TransactionsScreen
   (React Navigation)
              ↓
        bridgeUtils.openNativeScreen('ProfileActivity')
              ↓
        Native Activity (Android/iOS)
              ↓
        Back → Return to React Native
```

### 3. Header Updates
```
Component mounts
  ↓
dispatch(setHeaderTitle('New Title'))
  ↓
Redux state updates
  ↓
useEffect triggers
  ↓
bridgeUtils.updateHeader('New Title')
  ↓
Native module called
  ↓
Toolbar/UINavigationBar updates ✓
```

### 4. Bridge Communication

**RN → Native (Promise-based):**
```typescript
const result = await bridgeUtils.openNativeScreen(name, params);
```

**Native → RN (Event-based):**
```typescript
bridgeUtils.onNativeEvent('eventName', data => { ... });
```

---

## 🔒 Security Rules

### ✅ DO:
```typescript
// Pass userId only
bridgeUtils.openNativeScreen('ProfileActivity', { userId })

// Log safely
console.log('Auth state:', isAuthenticated)

// Store in secure native storage
secureStore.set('authToken', token)

// Emit events without logging
bridgeUtils.emitToNative('userLoggedIn', { userId })
```

### ❌ DON'T:
```typescript
// Log authToken
console.log('Token:', authToken)

// Pass in URL
navigate(`screen?token=${authToken}`)

// Store raw in Redux
state.token = rawToken

// Expose in error messages
catch(e) { console.log(`Failed: ${token}`) }
```

---

## 📱 Testing Checklist

### Launch Test
```
✓ App launches
✓ Redux state initialized
✓ i18n loaded
✓ RN content visible below native header
```

### Navigation Test
```
✓ Dashboard screen visible
✓ Tap Profile → Navigate to ProfileScreen
✓ Tap back → Return to Dashboard
✓ Tap Transactions → Navigate to TransactionsScreen
```

### Bridge Test
```
✓ Tap "Open Native Profile"
✓ Native screen opens with userId
✓ Tap back → Return to React Native
✓ RN screen preserved
```

### Header Test
```
✓ Header title matches current screen
✓ Each screen has correct title
✓ Dynamic updates work
```

---

## 🛠️ Adding Features

### New Redux State
```typescript
// 1. Create slice
const mySlice = createSlice({...});

// 2. Add to store
configureStore({ reducer: { mySlice } });

// 3. Use in component
const data = useSelector(s => s.mySlice);
```

### New Native Screen
```typescript
// 1. Create Activity (Android) or ViewController (iOS)
// 2. Register in NavigationBridgeModule/Delegate
// 3. Call from RN
bridgeUtils.openNativeScreen('MyScreen', params);
```

### New Language
```json
// 1. Create src/i18n/es.json
// 2. Register in i18n/config.ts
// 3. Use in components
const { t } = useTranslation();
<Text>{t('welcome')}</Text>
```

---

## 🐛 Common Issues

| Issue | Fix |
|-------|-----|
| Metro won't start | `npm start --reset-cache` |
| Android won't build | `./gradlew clean && ./gradlew sync` |
| Pods won't install | `pod cache clean --all && pod install --repo-update` |
| Bridge not found | Rebuild native app, check registration |
| Header not updating | Call bridgeUtils.updateHeader() |
| RN content not showing | Check FrameLayout/container hierarchy |

---

## 📚 Documentation Map

| Document | Content |
|----------|---------|
| [README.md](./README.md) | Full project overview & architecture |
| [SETUP.md](./SETUP.md) | Quick 30-minute setup guide |
| [ARCHITECTURE.md](./ARCHITECTURE.md) | Deep dive into design patterns |
| [DELIVERABLES.md](./DELIVERABLES.md) | Implementation checklist |
| **This file** | Quick reference |

---

## 🎓 Learning Path

1. **Start**: Read [SETUP.md](./SETUP.md) (15 min)
2. **Understand**: Read [README.md](./README.md) (30 min)
3. **Deep Dive**: Read [ARCHITECTURE.md](./ARCHITECTURE.md) (60 min)
4. **Build**: Follow code examples in source files
5. **Extend**: Use architecture patterns for new features

---

## 📞 Quick Answers

**Q: How do I navigate between RN screens?**
```typescript
navigation.navigate('ProfileScreen');
```

**Q: How do I open a native screen?**
```typescript
bridgeUtils.openNativeScreen('ProfileActivity', { userId });
```

**Q: How do I update the header?**
```typescript
bridgeUtils.updateHeader('New Title', 'ButtonLabel');
```

**Q: How do I change language?**
```typescript
i18n.changeLanguage('ar');
```

**Q: How do I access auth state?**
```typescript
const auth = useSelector(state => state.auth);
```

**Q: How do I dispatch Redux action?**
```typescript
dispatch(setAuthenticated({ userId, authToken }));
```

**Q: How do I add new language?**
```
1. Create src/i18n/es.json
2. Register in src/i18n/config.ts
3. Use: const { t } = useTranslation(); <Text>{t('key')}</Text>
```

**Q: How do I change colors?**
```
1. Edit design-tokens/theme.json
2. Use: backgroundColor: theme.colors.primary
```

---

## 🚀 Next Steps

1. **Setup** the project (SETUP.md)
2. **Run** on Android & iOS
3. **Test** all navigation flows
4. **Read** ARCHITECTURE.md for deep understanding
5. **Extend** with your own screens/features
6. **Deploy** to production

---

## 📝 Commit Message Template

```
feat: Add new RN feature
- Describe what was added
- Explain native changes if any
- Reference bridge changes if needed

perf: Optimize bridge communication
- Reduce number of bridge calls
- Batch state updates

fix: Fix header not updating
- Ensure bridgeUtils.updateHeader() called
- Verify Redux dispatch
```

---

## 🎉 You're All Set!

Everything is configured and ready to use.

**Start with [SETUP.md](./SETUP.md) for a quick 30-minute setup.**

Happy coding! 🚀
