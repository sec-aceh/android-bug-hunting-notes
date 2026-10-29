<p align="center">
  <img src="./bughunter.jpg" width="250" alt="SEC-ACEH Logo"/>
</p>

# APK BUG BOUNTY HUNTING - WORKFLOW ANTI-DUPLICATE
# Lab: InsecureBankv2 (Practice) -> Real Target (HackerOne/Bugcrowd)
# Author: Debker Journey - Kali Linux
# Date: $(date)

---

## 0. LEGAL & SCOPE
- HANYA test APK yang ada di scope bug bounty program
- Pastikan program mengizinkan Mobile / Android testing
- Jangan test APK Bank asli tanpa izin, pakai APK lab dulu
- Simpan bukti scope: screenshot halaman program

## 1. SETUP TOOLS (Sekali aja)
sudo apt update && sudo apt install -y apktool jadx dex2jar sqlite3
# apktool = bongkar resource & smali
# jadx = decompile dex -> java (paling enak dibaca)
# jadx-gui = versi GUI kalau mau klik-klik

## 2. DAPETIN APK TARGET
# Opsi A: Dari Play Store (pakai Aurora Store / APKMirror dengan izin)
# Opsi B: Lab (buat latihan)
cd /tmp
wget https://github.com/dineshshetty/Android-InsecureBankv2/raw/master/InsecureBankv2.apk -O InsecureBankv2.apk

## 3. RECON - BONGKAR APK (Static Analysis Step 1)

# 3.1 Bongkar dengan apktool (dapat AndroidManifest.xml + smali)
apktool d /tmp/InsecureBankv2.apk -o /tmp/bongkaran_bank

# 3.2 Cek permission & exported component (CARI YANG EXPORTED=TRUE TANPA PERMISSION)
cat /tmp/bongkaran_bank/AndroidManifest.xml | grep -E "exported|permission" -i
# BUG HUNT: Kalau ada android:exported="true" tanpa android:permission -> POTENSI BUG

# 3.3 Decompile ke Java (biar bisa baca kayak PHP)
rm -rf /tmp/output_jadx && mkdir /tmp/output_jadx
jadx -d /tmp/output_jadx /tmp/InsecureBankv2.apk
ls /tmp/output_jadx/sources/com/android/insecurebankv2/

## 4. HUNTING - CARI BUG (Static Analysis Step 2)

# KITA PAKAI FILTER FOLDER APLIKASI, BUKAN LIBRARY GOOGLE
TARGET_SMALI="/tmp/bongkaran_bank/smali/com/android/insecurebankv2"
TARGET_JAVA="/tmp/output_jadx/sources/com/android/insecurebankv2"

# 4.1 [CRITICAL] Hardcoded Secret & API Key
echo "[*] Hunting Hardcoded Key"
grep -R "api_key\|secret\|token\|key.*123" $TARGET_SMALI -i | head -20
grep -R "This is the super secret" $TARGET_SMALI -i
# Contoh Real Bug: CryptoClass.java -> String key = "This is the super secret key 123"
# Impact: Bisa decrypt semua data user
# CWE: CWE-798

# 4.2 [HIGH] Hardcoded Password & Broadcast Leak
echo "[*] Hunting Password Leak"
grep -R "password\|superSecurePassword" $TARGET_SMALI -i | head -20
cat $TARGET_JAVA/MyBroadCastReceiver.java
# Contoh Real Bug: Password di-decrypt dan dikirim via SMS ke nomor penyerang
# PoC: am broadcast -a <action> --es phonenumber <nomor_kita>

# 4.3 [HIGH] Insecure Crypto - IV Zero & Weak Algorithm
cat $TARGET_JAVA/CryptoClass.java
# Cek: ivBytes = {0,0,0...} -> CWE-329 (IV tidak random)
# Cek: AES/ECB atau AES/CBC/PKCS5Padding dengan key hardcode

# 4.4 [MEDIUM-HIGH] Exported Component + Content Provider SQLi
cat $TARGET_JAVA/TrackUserContentProvider.java
cat /tmp/bongkaran_bank/AndroidManifest.xml | grep -A2 provider
# Test: content://com.android.insecurebankv2.TrackUserContentProvider/trackerusers/

# 4.5 [MEDIUM] HTTP (bukan HTTPS) & Hardcoded IP
grep -R "http://" $TARGET_SMALI -i | grep -v "http://schemas" | head -20
# Real bug: DoTransfer.java masih pakai http:// -> bisa di-MITM

# 4.6 [MEDIUM] SharedPreferences & SQLite Leak
grep -R "SharedPreferences\|getString.*password\|MODE_WORLD_READABLE" $TARGET_JAVA -i

## 5. REPORT TEMPLATE (Biar di-accept HackerOne)

# Title: [CWE-ID] Hardcoded AES Encryption Key in CryptoClass.java Leads to Sensitive Data Decryption
# Severity: Critical / High
# Steps To Reproduce:
# 1. Download APK vX.X dari Play Store (link program)
# 2. Decompile dengan: jadx -d output app.apk
# 3. Buka sources/com/target/CryptoClass.java line 18
# 4. Terdapat: String key = "This is the super secret key 123"
# 5. Key ini dipakai di aes256decrypt() untuk decrypt password user di MyBroadCastReceiver.java
# 6. Penyerang bisa decrypt data lokal / trigger broadcast untuk leak via SMS
# Impact: Confidentiality breach, attacker can decrypt all encrypted data offline
# Remediation: Gunakan Android Keystore, jangan hardcode key di APK, gunakan random IV
# PoC: Screenshot CryptoClass.java + MyBroadCastReceiver.java

## 6. NEXT LEVEL - DYNAMIC (Kalau Static udah habis)
# - Setup Emulator + Burp (Proxy 127.0.0.1:8080)
# - Frida bypass SSL pinning: frida --codeshare akabe1/ssl-pinning-bypass -f com.target.app
# - Check logcat: adb logcat | grep -i password

## 7. COMMAND CEPAT BUAT REAL TARGET
# Ganti InsecureBankv2.apk dengan target.apk
# jadx -d /tmp/output_jadx /tmp/target.apk && grep -R "secret\|api_key\|password" /tmp/output_jadx/sources --include="*.java" -i -n | head -30

