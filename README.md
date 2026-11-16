# NetAudit

تطبيق أندرويد مفتوح المصدر لفحص أمان **شبكتك الخاصة** وكشف الإعدادات الضعيفة فيها.
للاستخدام الدفاعي على شبكة تملكها أو لديك إذن صريح بفحصها فقط.

## ماذا يفحص الإصدار V1؟
- نوع تشفير الواي فاي (WPA3 / WPA2 / WPA / WEP / مفتوح)
- المنافذ الخطرة على الراوتر (Telnet وFTP وSMB وRDP وADB وTR-069)
- لوحة إدارة الراوتر هل تعمل عبر HTTP فقط
- قائمة الأجهزة الحيّة في الشبكة المحلية
- درجة أمان من 100 مع توصيات

لا يتضمن: كسر كلمات المرور، قطع اتصال الأجهزة، استغلال الثغرات.

## الإعداد خطوة بخطوة
1. في Android Studio: **New Project ← Empty Activity** (Kotlin + Compose)، واجعل اسم الحزمة `com.netaudit`.
2. استبدل `MainActivity.kt` بالملف المرفق، وأضف `NetAudit.kt` في المجلد نفسه.
3. في `app/build.gradle.kts` أضف داخل `dependencies`:
```kotlin
implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.8.1")
```
4. في `AndroidManifest.xml` أضف قبل وسم `<application>`:
```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.ACCESS_WIFI_STATE" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
```
5. وصّل هاتفك واتصل بشبكتك ثم اضغط Run.
6. لإنتاج APK: **Build ← Build Bundle(s) / APK(s) ← Build APK(s)**.

## النشر على GitHub
```bash
git init
git add .
git commit -m "NetAudit V1"
git branch -M main
git remote add origin https://github.com/USERNAME/netaudit.git
git push -u origin main
```
ثم أنشئ Release وارفع ملف الـ APK الموقّع إليه.

## الترخيص
يُقترح **Apache-2.0** (مرن ويمنح حماية براءات الاختراع) أو **MIT** (أبسط).
أضف ملف `LICENSE` من خيار *Add file ← Choose a license* في GitHub.

## إخلاء المسؤولية
يُستخدم هذا التطبيق على الشبكات التي تملكها أو لديك إذن مكتوب بفحصها فقط.
المطوّر غير مسؤول عن أي استخدام غير مشروع.
