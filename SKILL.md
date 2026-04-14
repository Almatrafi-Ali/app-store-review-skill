---
name: app-store-review
description: |
  فحص شامل لكود تطبيقات iOS/macOS في Xcode قبل رفعها على App Store. يفحص الكود والإعدادات ويكشف أسباب الرفض المحتملة.
  استخدم هذه المهارة عندما يطلب المستخدم: فحص التطبيق للمتجر، مراجعة قبل الرفع، تجهيز التطبيق للنشر، App Store review، أسباب رفض التطبيق.
  يجب استخدامها أيضاً عندما يذكر المستخدم أن Apple رفضت تطبيقه أو يريد معرفة لماذا تم الرفض.
  هذه المهارة تفحص الكود فقط - لا تفحص إعدادات App Store Connect (مثل Support URL وPrivacy Policy URL) لأن المطوّر لن يستطيع الإرسال بدونها أصلاً.
---

# مهارة فحص كود التطبيق لمتجر Apple App Store

أنت مراجع خبير بقواعد Apple App Store Review Guidelines. مهمتك فحص **كود المشروع وإعداداته في Xcode** وتقديم تقرير مفصّل بالعربية مع المصطلحات التقنية بالإنجليزية.

> **ملاحظة:** هذه المهارة تفحص ما يمكن فحصه من الكود. أشياء مثل Support URL، Privacy Policy URL، Screenshots، Keywords - هذه يعبّئها المطوّر في App Store Connect ولن يستطيع الإرسال بدونها، فلا داعي للتنبيه عليها.

## آلية العمل

### الخطوة 1: اكتشاف المشروع

ابحث عن ملفات المشروع في المسار المحدد أو working directory:
```
*.xcodeproj/project.pbxproj
Info.plist
*.swift (ملفات المصدر الرئيسية، استثنِ Pods و LocalPackages و DerivedData)
*.entitlements
PrivacyInfo.xcprivacy
```

### الخطوة 2: الفحوصات

---

## 1. الخصوصية والأذونات (أكثر سبب رفض)

### تطابق APIs مع الأذونات
اقرأ pbxproj وابحث عن `INFOPLIST_KEY_NS*`، ثم ابحث بملفات .swift عن استخدام كل API:

| API بالكود | المفتاح المطلوب |
|-----------|----------------|
| `AVCaptureSession`, `UIImagePickerController(.camera)` | `NSCameraUsageDescription` |
| `PHPhotoLibrary`, `PhotosPicker`, `UIImagePickerController` | `NSPhotoLibraryUsageDescription` |
| `UIImageWriteToSavedPhotosAlbum`, `PHAssetChangeRequest` | `NSPhotoLibraryAddUsageDescription` |
| `AVAudioEngine`, `AVAudioRecorder`, `AVAudioSession(.record)` | `NSMicrophoneUsageDescription` |
| `CLLocationManager` | `NSLocationWhenInUseUsageDescription` |
| `CLLocationManager` + `allowsBackgroundLocationUpdates` | `NSLocationAlwaysUsageDescription` |
| `SFSpeechRecognizer` | `NSSpeechRecognitionUsageDescription` |
| `LAContext` (Face ID) | `NSFaceIDUsageDescription` |
| `CNContactStore` | `NSContactsUsageDescription` |
| `EKEventStore` | `NSCalendarsUsageDescription` |
| `CoreBluetooth` | `NSBluetoothAlwaysUsageDescription` |
| `CMMotionManager` | `NSMotionUsageDescription` |
| `HKHealthStore` | `NSHealthShareUsageDescription` |
| `ATTrackingManager` | `NSUserTrackingUsageDescription` |

**قواعد الفحص:**
- ❌ API مستخدم بدون إذن مقابل = **رفض مضمون**
- ❌ إذن موجود بدون API مستخدم = **رفض محتمل** (إذن غير مبرر)
- ❌ نص الإذن عام جداً ("This app needs access") = **رفض محتمل**
- ✅ النص يشرح **لماذا** التطبيق يحتاج الإذن بالتحديد

### Privacy Manifest
- ابحث عن `PrivacyInfo.xcprivacy` بمجلد المشروع
- ❌ إذا مفقود = **رفض** (مطلوب من Apple لكل التطبيقات الجديدة)
- إذا موجود، تحقق إنه يغطي البيانات المستخدمة فعلاً

### سياسة الخصوصية داخل التطبيق
- ابحث عن: "PrivacyPolicy", "سياسة الخصوصية", "privacy" في ملفات Swift
- ❌ إذا ما فيه صفحة أو رابط خصوصية **داخل التطبيق** = رفض محتمل

### حذف الحساب
- ابحث عن: `SignIn`, `Login`, `Register`, `createAccount`, `authentication`, `AuthenticationServices`
- إذا فيه نظام حسابات:
  - ❌ لازم فيه خيار حذف الحساب داخل التطبيق
  - ابحث عن: "delete account", "حذف الحساب"

---

## 2. جودة الكود واستقرار التطبيق

### نصوص اختبارية
ابحث في ملفات .swift الرئيسية (استثنِ LocalPackages/Pods):
- `"TODO"`, `"FIXME"`, `"HACK"` - تعليقات تطوير متبقية
- `"placeholder"`, `"lorem"`, `"coming soon"`, `"تجريبي"` - محتوى مؤقت
- `"YOUR_API_KEY"`, `"YOUR_KEY"`, `"xxx"` - مفاتيح غير مكتملة

### كراشات محتملة
- `fatalError()` - كراش متعمد
- `preconditionFailure()` - كراش متعمد
- `try!` - كراش عند فشل
- `as!` - force cast كراش عند نوع خاطئ
- Force unwrap على متغيرات غير مضمونة (استثنِ IBOutlet patterns)

### print statements
- ابحث عن `print(` في كود الإنتاج
- ⚠️ يُفضل إزالتها أو استبدالها بـ `os_log` للإنتاج

---

## 3. الأمان

### App Transport Security
- ابحث عن `NSAllowsArbitraryLoads` في pbxproj أو Info.plist
- ❌ إذا `true` بدون مبرر = رفض محتمل
- ابحث عن `http://` بدل `https://` في الكود

### بيانات حساسة مكشوفة
- ابحث عن: API keys, passwords, secrets, tokens مكتوبة مباشرة بالكود
- أنماط البحث: `"sk-"`, `"api_key"`, `"password"`, `"secret"`, `"Bearer "`, `apiKey =`
- ❌ أي بيانات حساسة مكشوفة بالكود = خطر أمني

### عناوين IP مشفرة
- ابحث عن أنماط IP: `\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}`
- ⚠️ يُفضل استخدام domain names بدل IPs

---

## 4. المتطلبات التقنية

### Deployment Target
- اقرأ `IPHONEOS_DEPLOYMENT_TARGET` من pbxproj
- ⚠️ تحقق من وجود قيم متناقضة (project-level vs target-level)

### Background Modes
- ابحث عن `UIBackgroundModes` في pbxproj
- لكل mode موجود، تحقق إنه مستخدم فعلاً بالكود:
  - `audio` → ابحث عن `AVAudioSession`, `AVPlayer`
  - `location` → ابحث عن `allowsBackgroundLocationUpdates`
  - `fetch` → ابحث عن `BGTaskScheduler`, `performFetchWithCompletionHandler`
  - `remote-notification` → ابحث عن `UNUserNotificationCenter`
- ❌ mode مفعّل بدون استخدام فعلي = رفض

### WebView Wrapper
- ابحث عن `WKWebView`, `SFSafariViewController`
- إذا التطبيق **كله** WKWebView بدون ميزات native:
  - ❌ رفض - التطبيق لازم يقدم قيمة أكثر من موقع ويب

### Bundle ID
- اقرأ `PRODUCT_BUNDLE_IDENTIFIER` من pbxproj
- ✅ تحقق من صيغة reverse-DNS صحيحة

---

## 5. Sign in with Apple

- ابحث عن third-party login: `GoogleSignIn`, `GIDSignIn`, `FacebookLogin`, `FBSDKLoginButton`, `TwitterLogin`
- إذا موجود:
  - ابحث عن `ASAuthorizationController`, `ASAuthorizationAppleIDProvider`
  - ❌ third-party login بدون Sign in with Apple = **رفض مضمون**

---

## 6. الذكاء الاصطناعي (إذا التطبيق يستخدم AI)

ابحث عن: `llama`, `CoreML`, `MLModel`, `GenerativeModel`, `ChatGPT`, `OpenAI`, `LLM`, `GPT`

إذا التطبيق يستخدم AI:
- ⚠️ هل فيه إخلاء مسؤولية إن الردود قد لا تكون دقيقة؟
- ❌ هل الـ system prompt يتضمن تعليمات لرفض المحتوى الضار/العنيف/الجنسي؟
  - ابحث عن: "harmful", "offensive", "ضار", "مسيء", "refuse", "ارفض" بالـ system prompt
  - إذا مفقود = خطر رفض لأن Apple تتطلب فلترة المحتوى
- ⚠️ إذا يشارك بيانات مع AI خارجي (cloud API)، لازم يوضح للمستخدم

---

## 7. المشتريات والإعلانات

### In-App Purchase
- ابحث عن: `StoreKit`, `SKProduct`, `SKPayment`, `Product`, `purchase`
- إذا فيه محتوى رقمي مدفوع بدون StoreKit = ❌

### إعلانات
- ابحث عن: `GADMobileAds`, `AdMob`, `adUnitID`, `GADBannerView`, `interstitial`
- إذا موجودة:
  - ⚠️ لازم تكون مناسبة للفئة العمرية
  - ❌ ما تكون بـ extensions/widgets/notifications

---

## 8. أيقونات التطبيق

- تحقق من `Assets.xcassets/AppIcon.appiconset/Contents.json`
- ✅ تحقق من وجود أيقونة 1024x1024 (مطلوبة لـ App Store)
- ⚠️ تحقق من وجود جميع الأحجام المطلوبة

---

## تنسيق التقرير النهائي

```markdown
# 📋 تقرير فحص الكود لـ App Store

## معلومات المشروع
- **اسم المشروع:** [الاسم]
- **Bundle ID:** [الـ ID]
- **Deployment Target:** iOS [الإصدار]
- **تاريخ الفحص:** [التاريخ]

---

## النتيجة: [جاهز ✅ / يحتاج تعديلات ⚠️ / غير جاهز ❌]

| | العدد |
|--|-------|
| ✅ نجح | [عدد] |
| ⚠️ تحذير | [عدد] |
| ❌ حرج | [عدد] |

---

## التفاصيل

### 1. الخصوصية والأذونات
| الفحص | النتيجة | التفاصيل |
|-------|---------|----------|

### 2. جودة الكود
| الفحص | النتيجة | التفاصيل |
|-------|---------|----------|

[... باقي الأقسام ...]

---

## ❌ إجراءات مطلوبة (حسب الأولوية)
1. [المشكلة + الملف + الحل بالتحديد]

## ⚠️ توصيات
1. [التوصية]
```

---

## قواعد عامة

- افحص ملفات .swift **الرئيسية فقط** - استثنِ: `Pods/`, `LocalPackages/`, `DerivedData/`, `.build/`, `Tests/`
- كن **عملي ومحدد** - قول اسم الملف ورقم السطر
- إذا ما تقدر تفحص شي من الكود، **لا تذكره** (مثل Screenshots, Support URL, Keywords - هذي بـ App Store Connect)
- لو فيه مشكلة حرجة، اعرض **الحل بالكود** مباشرة
- الأولوية: ❌ حرج (رفض مضمون) > ⚠️ تحذير (رفض محتمل) > ✅ نجح
