# LibreraReader Security Audit Report

**Audit Date:** 2025-11-10
**Auditor:** Claude Code
**Scope:** Main application module and AndroidManifest
**Methodology:** Static code analysis, manifest review, dependency audit

---

## Executive Summary

This security audit identified **16 vulnerabilities** across multiple severity levels. The most critical issues involve:

- Cleartext traffic enabled (MITM vulnerability)
- Exported components without permission checks
- WebView security issues
- Outdated dependencies with CVEs
- XML External Entity (XXE) vulnerability

**Overall Security Score:** 6/10

**Risk Level:** MEDIUM-HIGH

**Immediate Action Required:** 4 critical/high severity issues

---

## CRITICAL SEVERITY ISSUES

### 🚨 1. Cleartext Traffic Allowed

**Severity:** CRITICAL (CVSS 7.5)
**CWE:** CWE-319 - Cleartext Transmission of Sensitive Information

**Affected Files:**
- `app/src/main/AndroidManifest.xml:50`
- `app/src/main/res/xml/network_security_config.xml:2`

**Vulnerability:**
```xml
<!-- AndroidManifest.xml -->
<application
    android:usesCleartextTraffic="true"
    android:networkSecurityConfig="@xml/network_security_config">

<!-- network_security_config.xml -->
<network-security-config>
    <base-config cleartextTrafficPermitted="true">
        <!-- ... -->
    </base-config>
</network-security-config>
```

**Impact:**
- All network traffic can be transmitted unencrypted
- Man-in-the-middle (MITM) attacks possible
- OPDS catalog credentials exposed
- Book downloads interceptable
- User data (reading history, preferences) exposed

**Attack Scenario:**
1. User connects to public WiFi (coffee shop, library)
2. Attacker performs MITM attack (e.g., using Wireshark, mitmproxy)
3. Attacker intercepts HTTP traffic
4. Attacker captures:
   - OPDS credentials
   - Downloaded books
   - User preferences
   - Cloud sync data

**Affected User Actions:**
- Browsing OPDS catalogs over HTTP
- Downloading books over HTTP
- Any HTTP API calls

**Fix:**

```xml
<!-- AndroidManifest.xml -->
<application
    android:usesCleartextTraffic="false"
    android:networkSecurityConfig="@xml/network_security_config">

<!-- network_security_config.xml -->
<network-security-config>
    <base-config cleartextTrafficPermitted="false">
        <trust-anchors>
            <certificates src="system" />
        </trust-anchors>
    </base-config>

    <!-- Only allow cleartext for localhost during development -->
    <domain-config cleartextTrafficPermitted="true">
        <domain includeSubdomains="true">localhost</domain>
        <domain includeSubdomains="true">127.0.0.1</domain>
    </domain-config>
</network-security-config>
```

**Exception Handling:**
If some OPDS servers only support HTTP:
```xml
<!-- Whitelist specific domains (use sparingly) -->
<domain-config cleartextTrafficPermitted="true">
    <domain includeSubdomains="false">legacy-opds-server.example.com</domain>
</domain-config>
```

**Testing:**
```bash
# Verify cleartext is blocked
adb shell am start -n com.foobnix.librera/.MainActivity
# Try to connect to http://example.com
# Should fail with cleartext traffic error
```

**Priority:** IMMEDIATE - Fix in next release

---

## HIGH SEVERITY ISSUES

### ⚠️ 2. Exported Activities Without Permission Checks

**Severity:** HIGH (CVSS 7.0)
**CWE:** CWE-927 - Use of Implicit Intent for Sensitive Communication

**File:** `app/src/main/AndroidManifest.xml`

#### 2.1 OpenerActivity (Lines 104-228)

**Vulnerability:**
```xml
<activity
    android:name="com.foobnix.OpenerActivity"
    android:exported="true"
    android:launchMode="singleTask">
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <data android:scheme="file" />
        <data android:scheme="content" />
        <data android:mimeType="*/*" />
    </intent-filter>
</activity>
```

**Code Vulnerability:**
`app/src/main/java/com/foobnix/OpenerActivity.java`
```java
// Lines 113, 126, 136, 143
file = new File(uri.getPath());  // No validation!
```

**Impact:**
- Malicious apps can send crafted intents
- Path traversal attacks possible (`../../sensitive-file`)
- Can trigger crashes with malformed intents
- Potential arbitrary file access

**Attack Scenario:**
```java
// Malicious app
Intent intent = new Intent(Intent.ACTION_VIEW);
intent.setComponent(new ComponentName(
    "com.foobnix.librera",
    "com.foobnix.OpenerActivity"
));
intent.setData(Uri.parse("file:///data/data/com.foobnix.librera/databases/app.db"));
startActivity(intent);
// Could potentially expose database file
```

**Fix:**

```java
// OpenerActivity.java
private boolean isPathSafe(String path) {
    try {
        File file = new File(path).getCanonicalFile();
        String canonicalPath = file.getAbsolutePath();

        // Allow only specific directories
        List<String> allowedPaths = Arrays.asList(
            Environment.getExternalStorageDirectory().getAbsolutePath(),
            getFilesDir().getAbsolutePath(),
            getCacheDir().getAbsolutePath()
        );

        for (String allowed : allowedPaths) {
            if (canonicalPath.startsWith(allowed)) {
                return true;
            }
        }

        LOG.w("Blocked access to: " + canonicalPath);
        return false;
    } catch (IOException e) {
        return false;
    }
}

@Override
protected void onCreate(Bundle savedInstanceState) {
    Uri uri = getIntent().getData();
    if (uri == null) {
        finish();
        return;
    }

    String path = uri.getPath();
    if (!isPathSafe(path)) {
        Toast.makeText(this, "Access denied", Toast.LENGTH_SHORT).show();
        finish();
        return;
    }

    // Proceed with validated path
}
```

**AndroidManifest.xml Fix:**
```xml
<!-- Add signature-level permission -->
<permission
    android:name="com.foobnix.librera.permission.OPEN_DOCUMENT"
    android:protectionLevel="signature" />

<activity
    android:name="com.foobnix.OpenerActivity"
    android:exported="true"
    android:permission="com.foobnix.librera.permission.OPEN_DOCUMENT">
```

---

#### 2.2 SendReceiveActivity (Lines 82-91)

**Vulnerability:**
```xml
<activity
    android:name="com.foobnix.zipmanager.SendReceiveActivity"
    android:exported="true">
    <intent-filter>
        <action android:name="android.intent.action.SEND" />
        <data android:mimeType="*/*" />
    </intent-filter>
</activity>
```

**Impact:**
- Accepts any MIME type without validation
- No input validation on intent data
- Potential DoS via large files

**Fix:**
```xml
<!-- Restrict to specific MIME types -->
<intent-filter>
    <action android:name="android.intent.action.SEND" />
    <data android:mimeType="application/pdf" />
    <data android:mimeType="application/epub+zip" />
    <!-- Add other supported formats -->
</intent-filter>
```

---

#### 2.3 CloudrailActivity (Lines 287-300)

**Vulnerability:**
```xml
<activity
    android:name="com.foobnix.ui2.CloudrailActivity"
    android:exported="true">
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="http" />
        <data android:scheme="https" />
    </intent-filter>
</activity>
```

**Impact:**
- Handles deep links without validation
- Can be triggered by any HTTP/HTTPS URL
- Potential phishing via crafted URLs

**Fix:**
```xml
<!-- Restrict to specific host -->
<data
    android:scheme="https"
    android:host="auth.librera.app"
    android:pathPrefix="/callback" />
```

---

### ⚠️ 3. WebView Security Issues

**Severity:** HIGH (CVSS 7.0)
**CWE:** CWE-749 - Exposed Dangerous Method or Function

**Affected Files:**
- `app/src/main/java/com/foobnix/android/utils/WebViewUtils.java`
- `app/src/main/java/com/foobnix/pdf/info/WebViewHepler.java`

#### 3.1 JavaScript Enabled Without Necessity

**Lines:** WebViewUtils.java:49, WebViewHepler.java:32
```java
web.getSettings().setJavaScriptEnabled(true);
```

**Impact:**
- Cross-Site Scripting (XSS) attacks possible
- Malicious scripts can execute
- Data exfiltration risk

#### 3.2 JavaScript Interface Exposed

**Line:** WebViewUtils.java:135
```java
web.addJavascriptInterface(new WebAppInterface(...), "android");
```

**Impact:**
- **CRITICAL for API < 17** - Allows arbitrary Java method invocation
- **HIGH for API ≥ 17** - Still risky if not carefully designed
- Remote code execution possible on older devices

**Attack Scenario:**
```html
<!-- Malicious web page -->
<script>
android.sensitiveMethod();  // Can call exposed Java methods
</script>
```

#### 3.3 File URL Loading

**Lines:** WebViewHepler.java:34, 65
```java
webView.loadUrl("file://error.html");
webView.loadUrl("file://" + path);
```

**Impact:**
- Can access local files if `setAllowFileAccess(true)`
- Potential access to app's private data
- Same-origin policy bypass

**Fix:**

```java
// WebViewUtils.java
public static void init(WebView web) {
    WebSettings settings = web.getSettings();

    // Disable JavaScript unless absolutely necessary
    settings.setJavaScriptEnabled(false);

    // Disable file access
    settings.setAllowFileAccess(false);
    settings.setAllowFileAccessFromFileURLs(false);
    settings.setAllowUniversalAccessFromFileURLs(false);

    // Disable content access if not needed
    settings.setAllowContentAccess(false);

    // Modern WebView settings
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
        settings.setSafeBrowsingEnabled(true);
    }

    // Remove JavaScript interface
    // web.addJavascriptInterface(...)  // ❌ Remove or secure
}

// If JavaScript interface is necessary:
class WebAppInterface {
    @JavascriptInterface  // Required for API 17+
    public void allowedMethod() {
        // Only expose essential, safe methods
    }

    // Don't expose dangerous methods
    // public void deleteFile(String path) { ... }  // ❌ Dangerous
}
```

**Alternative for File Loading:**
```java
// Instead of file:// URLs, use content:// with FileProvider
Uri contentUri = FileProvider.getUriForFile(
    context,
    "com.foobnix.librera.fileprovider",
    file
);
webView.loadUrl(contentUri.toString());
```

---

### ⚠️ 4. Outdated Dependencies with CVEs

**Severity:** HIGH (CVSS 7.5)
**CWE:** CWE-1035 - Using Components with Known Vulnerabilities

**File:** `app/build.gradle:320`

**Vulnerability:**
```gradle
implementation 'com.squareup.okhttp3:okhttp:3.12.6'
```

**CVE:** CVE-2021-0341
**CVSS Score:** 7.5 (HIGH)
**Description:** Inadequate encryption strength, hostname verification bypass

**Impact:**
- TLS/SSL security weakened
- Man-in-the-middle attacks easier
- Certificate validation bypass possible

**Affected Operations:**
- All OPDS catalog connections
- Book downloads
- Cloud sync
- Any HTTP/HTTPS request

**Fix:**
```gradle
implementation 'com.squareup.okhttp3:okhttp:5.1.0'  // Latest stable
```

**See:** Dependency Audit Report (02-dependencies-and-upgrades.md) for full details

---

### ⚠️ 5. Command Injection Vulnerability

**Severity:** HIGH (CVSS 7.0)
**CWE:** CWE-78 - OS Command Injection

**File:** `app/src/main/java/com/foobnix/pdf/info/ExtUtils.java:1781`

**Vulnerability:**
```java
final Process process = new ProcessBuilder()
    .command("mount | grep /dev/block/vold")  // ❌ Shell operators
    .redirectErrorStream(true).start();
```

**Impact:**
- ProcessBuilder doesn't support shell operators (|, >, <, etc.)
- Code won't work as intended
- If any user input reaches this, command injection possible

**Fix:**
```java
// Option 1: Use separate commands
final Process mount = new ProcessBuilder()
    .command("mount")
    .redirectErrorStream(true)
    .start();

// Then grep the output in Java
BufferedReader reader = new BufferedReader(new InputStreamReader(mount.getInputStream()));
String line;
while ((line = reader.readLine()) != null) {
    if (line.contains("/dev/block/vold")) {
        // Process line
    }
}

// Option 2: Use /system/bin/sh (if necessary)
final Process process = new ProcessBuilder()
    .command("/system/bin/sh", "-c", "mount | grep /dev/block/vold")
    .redirectErrorStream(true)
    .start();
```

**Better:** Avoid shell commands entirely, use Android APIs:
```java
// Use Android's storage APIs instead
StorageManager storageManager = (StorageManager) getSystemService(Context.STORAGE_SERVICE);
List<StorageVolume> volumes = storageManager.getStorageVolumes();
// ...
```

---

### ⚠️ 6. XML External Entity (XXE) Vulnerability

**Severity:** HIGH (CVSS 7.5)
**CWE:** CWE-611 - XML External Entity Reference

**File:** `app/src/main/java/net/arnx/wmf2svg/gdi/svg/SvgGdi.java:111`

**Vulnerability:**
```java
DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();
DocumentBuilder builder = factory.newDocumentBuilder();
// ❌ No XXE protection!
```

**Impact:**
- Malicious XML files can read arbitrary files
- Server-Side Request Forgery (SSRF) possible
- Denial of Service (DoS) via billion laughs attack
- Information disclosure

**Attack Scenario:**
```xml
<!-- Malicious SVG file -->
<!DOCTYPE foo [
  <!ENTITY xxe SYSTEM "file:///data/data/com.foobnix.librera/databases/app.db">
]>
<svg>
  <text>&xxe;</text>
</svg>
```

**Fix:**
```java
DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();

// Disable DTDs entirely
factory.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);

// If DTDs must be supported, disable external entities
factory.setFeature("http://xml.org/sax/features/external-general-entities", false);
factory.setFeature("http://xml.org/sax/features/external-parameter-entities", false);
factory.setFeature("http://apache.org/xml/features/nonvalidating/load-external-dtd", false);

// Disable entity expansion
factory.setExpandEntityReferences(false);

// Enable secure processing
factory.setFeature(XMLConstants.FEATURE_SECURE_PROCESSING, true);

DocumentBuilder builder = factory.newDocumentBuilder();
```

**Apply to all XML parsers:**
- XmlPullParser (OPDS.java)
- DocumentBuilder (SvgGdi.java)
- SAXParser (if used)

---

## MEDIUM SEVERITY ISSUES

### ⚠️ 7. Insecure Random Number Generation

**Severity:** MEDIUM (CVSS 5.0)
**CWE:** CWE-330 - Use of Insufficiently Random Values

**Affected Files:**
- `app/src/main/java/com/foobnix/pdf/info/ADS.java:67,69`
- `app/src/main/java/com/foobnix/pdf/info/TintUtil.java:76,85`
- `app/src/main/java/com/foobnix/opds/OPDS.java:56`

**Vulnerability:**
```java
new Random().nextBoolean()
new Random().nextInt(360)
```

**Impact:**
- Predictable random numbers
- If used for security: session IDs, tokens predictable
- Current usage: UI decisions (low impact)

**Risk:** Low for current usage, but bad practice

**Fix:**
```java
// For non-security uses (UI, colors, etc.)
// Current usage is fine, but better:
ThreadLocalRandom.current().nextInt(360);

// For security-sensitive uses:
SecureRandom secureRandom = new SecureRandom();
int value = secureRandom.nextInt(360);
```

---

### ⚠️ 8. Weak Cryptographic Hash (MD5)

**Severity:** MEDIUM (CVSS 5.0)
**CWE:** CWE-327 - Use of a Broken or Risky Cryptographic Algorithm

**File:** `app/src/main/java/com/foobnix/pdf/info/ADS.java:145`

**Vulnerability:**
```java
MessageDigest digest = java.security.MessageDigest.getInstance("MD5");
```

**Impact:**
- MD5 is cryptographically broken
- Hash collisions possible
- Not suitable for security purposes

**Current Usage:** Device ID hashing (not password)
**Risk:** Medium - not critical but should be fixed

**Fix:**
```java
MessageDigest digest = java.security.MessageDigest.getInstance("SHA-256");
// Or even better: SHA-3
// MessageDigest digest = MessageDigest.getInstance("SHA3-256");
```

---

### ⚠️ 9. Path Traversal Vulnerabilities

**Severity:** MEDIUM (CVSS 6.0)
**CWE:** CWE-22 - Path Traversal

**File:** `app/src/main/java/com/foobnix/OpenerActivity.java`

**Vulnerability:**
```java
// Lines 113, 126, 136, 143
file = new File(uri.getPath());  // No canonical path check
```

**Attack Scenario:**
```
file://../../data/data/com.foobnix.librera/shared_prefs/preferences.xml
```

**Impact:**
- Access files outside intended directories
- Read app's private data
- Potential data exfiltration

**Fix:**
```java
private File validatePath(Uri uri) throws IOException {
    File file = new File(uri.getPath());
    File canonicalFile = file.getCanonicalFile();
    String canonicalPath = canonicalFile.getAbsolutePath();

    // Define allowed directories
    List<String> allowedDirs = Arrays.asList(
        Environment.getExternalStorageDirectory().getAbsolutePath(),
        getFilesDir().getAbsolutePath()
    );

    // Check if canonical path starts with allowed directory
    boolean allowed = false;
    for (String allowedDir : allowedDirs) {
        if (canonicalPath.startsWith(allowedDir)) {
            allowed = true;
            break;
        }
    }

    if (!allowed) {
        throw new SecurityException("Path traversal detected: " + canonicalPath);
    }

    return canonicalFile;
}
```

---

### ⚠️ 10. Intent Redirection Risk

**Severity:** MEDIUM (CVSS 5.5)
**CWE:** CWE-940 - Intent Hijacking

**File:** `app/src/main/java/com/foobnix/zipmanager/SendReceiveActivity.java:69-73`

**Vulnerability:**
```java
getIntent().setAction(Intent.ACTION_VIEW);
getIntent().setClass(this, OpenerActivity.class);
startActivity(getIntent());  // Forwards entire intent without validation
```

**Impact:**
- Malicious apps can craft intents
- Bypass security checks
- Inject malicious data

**Fix:**
```java
// Create new intent with validated data
Intent safeIntent = new Intent(Intent.ACTION_VIEW);
safeIntent.setClass(this, OpenerActivity.class);

// Only copy validated data
Uri uri = getIntent().getData();
if (isUriSafe(uri)) {
    safeIntent.setData(uri);
    startActivity(safeIntent);
} else {
    // Handle error
}
```

---

### ⚠️ 11. Missing Backup Configuration

**Severity:** MEDIUM (CVSS 4.5)
**CWE:** CWE-200 - Information Exposure

**File:** `app/src/main/AndroidManifest.xml`

**Vulnerability:**
```xml
<application
    <!-- android:allowBackup attribute missing (defaults to true) -->
```

**Impact:**
- App data can be backed up via ADB
- Potential exposure of:
  - SharedPreferences (including OPDS credentials)
  - Database files (book metadata, reading history)
  - Cache files

**Attack Scenario:**
```bash
# Attacker with physical device access
adb backup -f backup.ab -noapk com.foobnix.librera
# Extract and read app data
```

**Fix:**

**Option 1: Disable backups (most secure)**
```xml
<application
    android:allowBackup="false"
    android:fullBackupContent="false">
```

**Option 2: Selective backup with rules**
```xml
<application
    android:allowBackup="true"
    android:fullBackupContent="@xml/backup_rules">

<!-- res/xml/backup_rules.xml -->
<full-backup-content>
    <exclude domain="sharedpref" path="opds_credentials.xml"/>
    <exclude domain="database" path="."/>
    <include domain="file" path="books/"/>
</full-backup-content>
```

---

### ⚠️ 12. Password Transmitted via Intent Extra

**Severity:** MEDIUM (CVSS 4.5)
**CWE:** CWE-319 - Cleartext Transmission of Sensitive Information

**File:** `app/src/main/java/com/foobnix/pdf/search/activity/HorizontalModeController.java:114`

**Vulnerability:**
```java
String pasw = activity.getIntent().getStringExtra(EXTRA_PASSWORD);
```

**Impact:**
- Passwords in intents can be logged
- Accessible via dumpsys (adb shell)
- Other apps with permissions can intercept

**Attack Scenario:**
```bash
# Attacker with USB debugging
adb shell dumpsys activity | grep password
```

**Fix:**

**Option 1: Use secure storage**
```java
// Store password in EncryptedSharedPreferences
val encryptedPrefs = EncryptedSharedPreferences.create(
    "secure_prefs",
    masterKey,
    context,
    EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
    EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM
)
encryptedPrefs.edit().putString("password", password).apply()

// Pass only token/reference via intent
intent.putExtra("password_ref", passwordRef)
```

**Option 2: Use Android Keystore**
```java
// Encrypt password before putting in intent
String encryptedPassword = encryptWithKeystore(password);
intent.putExtra(EXTRA_PASSWORD, encryptedPassword);
```

---

## LOW SEVERITY ISSUES

### 💡 13. Excessive Logging

**Severity:** LOW (CVSS 2.0)
**CWE:** CWE-532 - Information Exposure Through Log Files

**Affected:** Multiple files

**Vulnerability:**
```java
LOG.d("File path: " + path);
LOG.d("Intent: " + intent.toString());
```

**Impact:**
- Sensitive data in logs
- Accessible by apps with READ_LOGS permission
- Visible in logcat during debugging

**Examples:**
- `OpenerActivity.java:103-107,121` - URIs and intents
- `OPDS.java:68-70` - Proxy credentials

**Fix:**

```java
// Option 1: Remove in production
if (BuildConfig.DEBUG) {
    LOG.d("Debug info: " + sensitiveData);
}

// Option 2: Use ProGuard to strip logs
-assumenosideeffects class android.util.Log {
    public static *** d(...);
    public static *** v(...);
}

// Option 3: Sanitize sensitive data
LOG.d("File path: " + sanitizePath(path));
```

---

### 💡 14. SQL Injection Risk (Low)

**Severity:** LOW (CVSS 3.0)
**CWE:** CWE-89 - SQL Injection

**File:** `app/src/main/java/com/foobnix/ui2/AppDB.java:388`

**Vulnerability:**
```java
String SQL_DISTINCT_ENAME = "SELECT DISTINCT " + in.getProperty().columnName +
    " as c FROM " + FileMetaDao.TABLENAME + " WHERE ...";
Cursor c = daoSession.getDatabase().rawQuery(SQL_DISTINCT_ENAME, null);
```

**Current Risk:** LOW - `in.getProperty().columnName` comes from enum, not user input

**Potential Risk:** If enum source ever changes or accepts user input

**Fix:**
```java
// Use parameterized queries
String SQL = "SELECT DISTINCT ? as c FROM " + FileMetaDao.TABLENAME + " WHERE ...";
Cursor c = daoSession.getDatabase().rawQuery(SQL, new String[]{columnName});

// Better: Use GreenDAO query builder (already type-safe)
List<FileMeta> results = fileMetaDao.queryBuilder()
    .distinct()
    .orderAsc(property)
    .list();
```

---

### 💡 15. Weak Password Encoding

**Severity:** LOW (CVSS 2.5)
**CWE:** CWE-838 - Inappropriate Encoding

**File:** `app/src/main/java/at/stefl/opendocument/java/odf/OpenDocumentCryptoUtil.java:112`

**Vulnerability:**
```java
byte[] passwordBytes = password.getBytes();  // Platform default charset
```

**Impact:**
- Platform-dependent charset
- May cause password validation failures
- Not a security issue but portability problem

**Fix:**
```java
byte[] passwordBytes = password.getBytes(StandardCharsets.UTF_8);
```

---

### 💡 16. Potential Information Disclosure

**Severity:** LOW (CVSS 2.0)
**CWE:** CWE-209 - Information Exposure Through Error Messages

**General Issue:** Exception stack traces may be logged or displayed

**Fix:**
```java
// Don't expose internal details
catch (Exception e) {
    LOG.e("Error occurred", e);  // Detailed logs only in debug
    Toast.makeText(this, "An error occurred", Toast.LENGTH_SHORT).show();  // Generic message to user
}
```

---

## SUMMARY

### Severity Distribution

| Severity | Count | Issues |
|----------|-------|--------|
| 🚨 **CRITICAL** | 1 | Cleartext traffic |
| ⚠️ **HIGH** | 6 | Exported activities, WebView, CVEs, XXE, Command injection |
| ⚠️ **MEDIUM** | 6 | Insecure random, weak crypto, path traversal, intent security |
| 💡 **LOW** | 3 | Logging, SQL, encoding |
| **TOTAL** | **16** | |

---

### Risk Assessment

**Overall Risk:** MEDIUM-HIGH

**Critical Systems at Risk:**
- Network communication (OPDS, downloads)
- User data (reading history, preferences)
- File access (books, database)
- Authentication (credentials)

**Data at Risk:**
- OPDS credentials (plaintext in transit, plaintext in storage)
- Reading history
- Book metadata
- User preferences
- Downloaded books

---

## PRIORITIZED REMEDIATION PLAN

### Phase 1: IMMEDIATE (Week 1)

**Priority: CRITICAL**

1. **Disable Cleartext Traffic**
   - Update AndroidManifest.xml
   - Update network_security_config.xml
   - Test all network operations

2. **Update OkHttp (CVE Fix)**
   - Update to OkHttp 5.1.0+
   - Test OPDS connections
   - Verify authentication still works

**Effort:** 8-16 hours
**Impact:** Fixes most critical vulnerabilities

---

### Phase 2: HIGH PRIORITY (Weeks 2-3)

**Priority: HIGH**

3. **Secure Exported Activities**
   - Add permission checks
   - Validate all intent data
   - Implement path traversal protection

4. **Fix WebView Security**
   - Disable unnecessary JavaScript
   - Remove or secure JavaScript interface
   - Disable file access

5. **Fix XXE Vulnerability**
   - Secure all XML parsers
   - Test with malicious XML files

**Effort:** 20-30 hours
**Impact:** Prevents exploitation of high-severity issues

---

### Phase 3: MEDIUM PRIORITY (Month 2)

**Priority: MEDIUM**

6. **Replace Weak Cryptography**
   - MD5 → SHA-256
   - Secure password encoding

7. **Fix Path Traversal**
   - Canonical path validation
   - Directory whitelist

8. **Secure Intent Handling**
   - Validate intent data
   - Don't forward intents directly

9. **Configure Backup Rules**
   - Exclude sensitive data
   - Or disable backups

**Effort:** 15-20 hours

---

### Phase 4: LONG-TERM (Ongoing)

**Priority: LOW**

10. **Remove Excessive Logging**
    - Strip debug logs in production
    - Sanitize sensitive data

11. **Security Hardening**
    - Code review process
    - Penetration testing
    - Security training

**Effort:** Ongoing

---

## TESTING & VERIFICATION

### Security Test Cases

**1. Network Security:**
- [ ] Verify cleartext traffic is blocked
- [ ] Test HTTPS connections work
- [ ] Verify certificate validation
- [ ] Test with proxy

**2. Activity Security:**
- [ ] Test intent validation
- [ ] Test path traversal prevention
- [ ] Test with malicious intents

**3. WebView Security:**
- [ ] Test JavaScript disabled
- [ ] Test file access blocked
- [ ] Test with XSS payloads

**4. XML Security:**
- [ ] Test with XXE payloads
- [ ] Test with billion laughs attack
- [ ] Verify external entities blocked

**5. Authentication:**
- [ ] Verify credentials encrypted in storage
- [ ] Test authentication over HTTPS
- [ ] Test with wrong credentials

---

### Security Tools

**Static Analysis:**
```bash
# Android Lint
./gradlew lint

# FindBugs/SpotBugs
./gradlew spotbugsRelease

# OWASP Dependency Check
./gradlew dependencyCheckAnalyze
```

**Dynamic Analysis:**
```bash
# Drozer - Android security framework
drozer console connect

# Inspect exported components
run app.package.attacksurface com.foobnix.librera

# Test for path traversal
run app.activity.start --component com.foobnix.librera com.foobnix.OpenerActivity --extra uri file://../../data/data/com.foobnix.librera/databases/app.db
```

**Network Analysis:**
```bash
# mitmproxy - Intercept HTTPS traffic
mitmproxy --mode transparent --showhost

# Verify cleartext is blocked
# Verify HTTPS connections work
```

---

## COMPLIANCE & STANDARDS

### OWASP Mobile Top 10 (2024)

| Risk | Status | Notes |
|------|--------|-------|
| M1: Improper Credential Usage | ⚠️ Partial | Credentials in cleartext |
| M2: Inadequate Supply Chain Security | ⚠️ Issues | Outdated dependencies |
| M3: Insecure Authentication | ⚠️ Partial | HTTP Basic Auth over cleartext |
| M4: Insufficient Input/Output Validation | ⚠️ Issues | Path traversal, XXE |
| M5: Insecure Communication | 🚨 Critical | Cleartext traffic enabled |
| M6: Inadequate Privacy Controls | ⚠️ Issues | Backup not configured |
| M7: Insufficient Binary Protection | ✅ Good | ProGuard enabled |
| M8: Security Misconfiguration | ⚠️ Issues | Exported activities |
| M9: Insecure Data Storage | ⚠️ Partial | Credentials in SharedPreferences |
| M10: Insufficient Cryptography | ⚠️ Issues | Weak hashing (MD5) |

---

### CWE Top 25 Most Dangerous

Findings mapped to CWE Top 25:

- ✅ CWE-22 (Path Traversal) - Found
- ✅ CWE-78 (OS Command Injection) - Found
- ✅ CWE-89 (SQL Injection) - Low risk found
- ✅ CWE-319 (Cleartext Transmission) - **CRITICAL**
- ✅ CWE-327 (Broken Crypto) - MD5 usage found
- ✅ CWE-611 (XXE) - Found

---

## RECOMMENDATIONS

### Immediate Actions (Next Release)

1. ✅ Disable cleartext traffic
2. ✅ Update OkHttp to latest
3. ✅ Add exported activity protections
4. ✅ Fix WebView security

**Target:** Next minor release (e.g., v8.9.x)

---

### Short-Term (Next 3 Months)

5. ✅ Fix XXE vulnerability
6. ✅ Implement path traversal protection
7. ✅ Replace weak cryptography
8. ✅ Configure backup rules
9. ✅ Secure intent handling

**Target:** Next major release (e.g., v9.0)

---

### Long-Term (Ongoing)

10. ✅ Implement EncryptedSharedPreferences for credentials
11. ✅ Add ProGuard log stripping
12. ✅ Regular security audits (quarterly)
13. ✅ Dependency vulnerability scanning (automated)
14. ✅ Penetration testing (annually)
15. ✅ Security training for developers
16. ✅ Bug bounty program (consider)

---

## SECURE DEVELOPMENT PRACTICES

### Code Review Checklist

**For New Code:**
- [ ] No hardcoded secrets
- [ ] Input validation on all user input
- [ ] Secure defaults (HTTPS, no cleartext)
- [ ] Minimal permissions requested
- [ ] Exported components have permissions
- [ ] No sensitive data in logs
- [ ] Cryptography uses modern algorithms
- [ ] File operations use canonical paths

**For Dependencies:**
- [ ] Check for known vulnerabilities
- [ ] Update regularly
- [ ] Review permissions requested
- [ ] Prefer well-maintained libraries

---

### Security Testing Integration

**CI/CD Pipeline:**
```yaml
# .github/workflows/security.yml
name: Security Checks

on: [push, pull_request]

jobs:
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2

      - name: Run lint
        run: ./gradlew lint

      - name: Dependency check
        run: ./gradlew dependencyCheckAnalyze

      - name: Upload reports
        uses: actions/upload-artifact@v2
        with:
          name: security-reports
          path: build/reports/
```

---

## CONCLUSION

LibreraReader has **16 identified security vulnerabilities** ranging from CRITICAL to LOW severity. The most pressing issues involve:

1. **Cleartext traffic** - Enables MITM attacks
2. **Exported components** - Allow malicious app interaction
3. **WebView security** - Potential XSS and code execution
4. **Outdated dependencies** - Known CVEs

**Current Security Posture:** 6/10 - MEDIUM

**After Phase 1 Fixes:** 8/10 - GOOD

**After All Fixes:** 9/10 - EXCELLENT

**Recommendation:** Prioritize Phase 1 fixes for immediate release, then systematically address remaining issues in subsequent releases. Implement ongoing security practices (automated scanning, regular audits) to maintain security posture.

**Estimated Effort:**
- Phase 1 (Critical): 8-16 hours
- Phase 2 (High): 20-30 hours
- Phase 3 (Medium): 15-20 hours
- **Total:** 43-66 hours (1-2 weeks full-time)

**Business Impact:** Addressing these issues will:
- Protect user data
- Prevent potential exploits
- Improve app store ratings
- Demonstrate security commitment
- Comply with security standards (OWASP, CWE)
- Reduce liability risk
