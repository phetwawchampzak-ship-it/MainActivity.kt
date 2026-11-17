# MainActivity.kt
name: Build Android APK
​on:
push:
branches: [ "main", "master" ]
workflow_dispatch:
​jobs:
build:
runs-on: ubuntu-latest
​steps:
- name: 1. ดึงไฟล์โค้ดจาก Repository
uses: actions/checkout@v4
​- name: 2. ตั้งค่า Java 17 สำหรับ Android Gradle
uses: actions/setup-java@v4
with:
distribution: 'temurin'
java-version: '17'
​- name: 3. สร้างโครงสร้างโปรเจกต์ Android และไฟล์ตั้งค่า Gradle อัตโนมัติ
run: |
mkdir -p app/src/main/java/com/example/aztecdetector
mkdir -p app/src/main/assets
​# ย้ายไฟล์โค้ดหลักเข้าสู่โฟลเดอร์แพ็กเกจ (รองรับทั้งชื่อ MainActivity.kt และ AztecDetectorApp.kt)
if [ -f "MainActivity.kt" ]; then
cp MainActivity.kt app/src/main/java/com/example/aztecdetector/MainActivity.kt
elif [ -f "AztecDetectorApp.kt" ]; then
cp AztecDetectorApp.kt app/src/main/java/com/example/aztecdetector/MainActivity.kt
fi
​# หากมีการอัปโหลดไฟล์โมเดล symbol_model.tflite ไว้ที่หน้าแรก ให้ย้ายเข้า assets อัตโนมัติ
if [ -f "symbol_model.tflite" ]; then
cp symbol_model.tflite app/src/main/assets/symbol_model.tflite
fi
​# สร้างไฟล์ settings.gradle.kts
cat << 'EOF' > settings.gradle.kts
pluginManagement {
repositories {
google()
mavenCentral()
gradlePluginPortal()
}
}
dependencyResolutionManagement {
repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
repositories {
google()
mavenCentral()
}
}
rootProject.name = "AztecDetector"
include(":app")
EOF
​# สร้างไฟล์ gradle.properties
cat << 'EOF' > gradle.properties
org.gradle.jvmargs=-Xmx2048m -Dfile.encoding=UTF-8
android.useAndroidX=true
kotlin.code.style=official
android.nonTransitiveRClass=true
EOF
​# สร้างไฟล์ build.gradle.kts (ระดับโปรเจกต์)
cat << 'EOF' > build.gradle.kts
plugins {
id("com.android.application") version "8.5.0" apply false
id("org.jetbrains.kotlin.android") version "1.9.24" apply false
}
EOF
​# สร้างไฟล์ app/build.gradle.kts
cat << 'EOF' > app/build.gradle.kts
plugins {
id("com.android.application")
id("org.jetbrains.kotlin.android")
}
​android {
namespace = "com.example.aztecdetector"
compileSdk = 34
​defaultConfig {
applicationId = "com.example.aztecdetector"
minSdk = 26
targetSdk = 34
versionCode = 1
versionName = "1.0"
}
​buildTypes {
release {
isMinifyEnabled = false
}
}
​compileOptions {
sourceCompatibility = JavaVersion.VERSION_17
targetCompatibility = JavaVersion.VERSION_17
}
​kotlinOptions {
jvmTarget = "17"
}
​androidResources {
noCompress += "tflite"
}
}
​dependencies {
implementation("androidx.core:core-ktx:1.13.1")
implementation("androidx.appcompat:appcompat:1.7.0")
implementation("com.google.android.material:material:1.12.0")
implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.8.1")
implementation("org.tensorflow:tensorflow-lite:2.16.1")
implementation("org.tensorflow:tensorflow-lite-support:0.4.4")
}
EOF
​# สร้างไฟล์ AndroidManifest.xml
cat << 'EOF' > app/src/main/AndroidManifest.xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
​<uses-permission android:name="android.permission.SYSTEM_ALERT_WINDOW" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_MEDIA_PROJECTION" />
<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
​<application
android:allowBackup="true"
android:label="Aztec Detector"
android:supportsRtl="true"
android:theme="@style/Theme.MaterialComponents.DayNight.NoActionBar">
​<activity
android:name=".MainActivity"
android:exported="true">
<intent-filter>
<action android:name="android.intent.action.MAIN" />
<category android:name="android.intent.category.LAUNCHER" />
</intent-filter>
</activity>
​<service
android:name=".ScreenCaptureService"
android:enabled="true"
android:exported="false"
android:foregroundServiceType="mediaProjection" />
​<service
android:name=".FloatingWindowService"
android:enabled="true"
android:exported="false" />
​</application>
</manifest>
EOF
​- name: 4. คอมไพล์โค้ดเป็นไฟล์ APK
run: |
gradle wrapper --gradle-version 8.7
./gradlew assembleDebug
​- name: 5. อัปโหลดไฟล์ APK ให้ดาวน์โหลด
uses: actions/upload-artifact@v4
with:
name: AztecDetector-APK
path: app/build/outputs/apk/debug/app-debug.apk
