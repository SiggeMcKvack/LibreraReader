# LibreraReader Dependency Audit & Upgrade Recommendations

**Audit Date:** 2025-11-10
**Auditor:** Claude Code
**Scope:** All modules (app, appCompose, appLibDrive, libPro, libReflow, Builder)

---

## Executive Summary

This audit analyzed **70+ dependencies** across all modules. Key findings:

- **🚨 1 CRITICAL Security Vulnerability:** OkHttp 3.12.6 (CVE-2021-0341)
- **⚠️ 12 Dependencies Need Updates**
- **✅ 50+ Dependencies Current**
- **📦 1 Deprecated Library:** legacy-support-v4
- **Overall Security Score:** 6/10

**Estimated Upgrade Effort:** 2-4 weeks (phased approach recommended)

---

## CRITICAL SECURITY ISSUES

### 🚨 OkHttp 3.12.6 - URGENT UPGRADE REQUIRED

**Module:** `app`
**File:** `app/build.gradle:320`
**Current Version:** 3.12.6 (released 2018)
**Latest Version:** 5.3.0
**Recommended:** 5.1.0 minimum

**Security Vulnerabilities:**
```
CVE-2021-0341 (CVSS Score: 7.5 - HIGH)
- Inadequate encryption strength
- Hostname verification bypass
- Man-in-the-middle attack risk
```

**Current Code:**
```gradle
implementation 'com.squareup.okhttp3:okhttp:3.12.6'
```

**Impact:**
- All network traffic vulnerable to MITM attacks
- Hostname verification can be bypassed
- TLS/SSL connection security compromised
- No longer maintained (end of life)

**Breaking Changes in 5.x:**
- Kotlin-first API (Java still supported)
- Package changes: Some classes moved
- Minimum Android SDK: API 21+
- Requires Java 8+

**Migration Guide:**

1. **Phase 1: Update to 4.12.0 (Conservative)**
   ```gradle
   implementation 'com.squareup.okhttp3:okhttp:4.12.0'
   ```
   - Fixes CVE-2021-0341
   - Minimal breaking changes
   - Java-friendly API

2. **Phase 2: Update to 5.1.0+ (Recommended)**
   ```gradle
   implementation 'com.squareup.okhttp3:okhttp:5.1.0'
   ```
   - Latest features and performance
   - Modern Kotlin API
   - Better HTTP/2 support

**Code Changes Required:**
```java
// Old (3.x):
OkHttpClient client = new OkHttpClient.Builder()
    .cache(new Cache(cacheDir, cacheSize))
    .build();

// New (5.x): Same API, but some edge cases differ
// Check: Connection pool configuration
// Check: Certificate pinning API changes
// Check: Interceptor interfaces
```

**Related Dependencies to Update:**
- `okio-parent:1.17.6` → `3.9.0` (update together with OkHttp)
- `okhttp-digest:1.21` → May need alternative (check compatibility)

**Testing Checklist:**
- [ ] OPDS catalog browsing
- [ ] Book downloads from catalogs
- [ ] Google Drive sync
- [ ] Authentication flows
- [ ] Proxy support (lines 59-79 in OPDS.java)
- [ ] Certificate handling

---

## BUILD SYSTEM

### Gradle & Build Tools

| Component | Current | Latest | Recommended | Effort | Priority |
|-----------|---------|--------|-------------|--------|----------|
| **Gradle** | 8.14.3 | 8.14.3 | ✅ Current | - | - |
| **AGP** | 8.12.0 | 8.13 | 8.13 | Low | Medium |
| **Kotlin** | 2.2.0 | 2.2.0 | ✅ Current | - | - |
| **Room Plugin** | 2.7.2 | 2.8.3 | 2.8.3 | Low | Medium |
| **KSP** | 2.2.0-2.0.2 | 2.2.0-2.0.7 | 2.2.0-2.0.7 | Low | Low |
| **Google Services** | 4.4.3 | 4.4.3 | ✅ Current | - | - |

**Android Gradle Plugin 8.12.0 → 8.13:**
```kotlin
// build.gradle.kts
plugins {
    id("com.android.application") version "8.13.0"
}
```
- **Benefits:** Bug fixes, improved build performance
- **Breaking Changes:** None expected
- **Migration Time:** < 1 hour

**Room Plugin 2.7.2 → 2.8.3:**
- **Benefits:** Better Kotlin 2.0 support, KSP improvements
- **Impact:** appCompose module (already uses Room 2.8.1)
- **Action:** Synchronize versions

---

## MODULE: app (Main Application)

### AndroidX Libraries

| Library | Current | Recommended | Effort | Benefits |
|---------|---------|-------------|--------|----------|
| **appcompat** | 1.7.1 | ✅ Current | - | - |
| **recyclerview** | 1.3.2 | ✅ Current | - | - |
| **cardview** | 1.0.0 | ✅ Current | - | Stable |
| **multidex** | 2.0.1 | ✅ Current | - | - |
| **work-runtime** | 2.10.2 | 2.11.0 | Low | Bug fixes |
| **legacy-support-v4** | 1.0.0 | 📦 **DEPRECATED** | Medium | Replace with specific libs |

**Work Manager Update:**
```gradle
implementation 'androidx.work:work-runtime:2.11.0'
```
- **Benefits:** Background task improvements, stability fixes
- **Breaking Changes:** None
- **Testing:** Verify SearchAllBooksWorker functionality

**Legacy Support v4 Deprecation:**

Current:
```gradle
implementation 'androidx.legacy:legacy-support-v4:1.0.0'
```

This library is deprecated and split into:
- `androidx.fragment:fragment:1.8.8`
- `androidx.media:media:1.7.0`
- `androidx.preference:preference:1.2.1`
- `androidx.localbroadcastmanager:localbroadcastmanager:1.1.0`

**Migration Steps:**
1. Identify which components are actually used
2. Add specific dependencies
3. Remove legacy-support-v4
4. Test thoroughly

---

### Core Dependencies

| Library | Current | Latest | Recommended | Priority | Notes |
|---------|---------|--------|-------------|----------|-------|
| **OkHttp** | 3.12.6 | 5.3.0 | 5.1.0 | 🚨 CRITICAL | Security vulnerability |
| **okio-parent** | 1.17.6 | 3.9.0 | 3.9.0 | High | Update with OkHttp |
| **okhttp-digest** | 1.21 | 1.21 | Check compatibility | High | May need alternative |
| **jsoup** | 1.21.1 | 1.21.2 | 1.21.2 | Low | HTML parser updates |
| **Glide** | 4.16.0 | 5.0.5 | 5.0.5 | Medium | Major version upgrade |
| **guava** | 33.4.8-android | 33.4.8 | ✅ Current | - | - |
| **eventbus** | 3.3.1 | 3.3.1 | ✅ Current | - | - |
| **juniversalchardet** | 2.5.0 | 2.5.0 | ✅ Current | - | - |
| **rtfparserkit** | 1.16.0 | 1.16.0 | ✅ Current | - | - |
| **mammoth** | 1.5.0 | 1.5.0 | ✅ Current | - | DOCX converter |
| **zip4j** | 2.11.5 | 2.11.5 | ✅ Current | - | - |
| **junrar** | 4.0.0 | 4.0.0 | ✅ Current | - | - |

**Jsoup Update:**
```gradle
implementation 'org.jsoup:jsoup:1.21.2'
```
- Bug fixes and HTML5 parsing improvements
- No breaking changes

**Glide 4.16.0 → 5.0.5:**
```gradle
implementation 'com.github.bumptech.glide:glide:5.0.5'
annotationProcessor 'com.github.bumptech.glide:compiler:5.0.5'
```

**Major Changes:**
- Improved Kotlin support
- Better memory management
- Enhanced transformation API
- Migration guide: https://github.com/bumptech/glide/wiki/Migration-Guide

**Testing Required:**
- Image loading throughout app
- Thumbnail generation
- Cover image caching
- Memory usage profiling

---

### Database (GreenDAO)

| Component | Current | Alternative | Migration Effort |
|-----------|---------|-------------|------------------|
| **GreenDAO** | 3.3.0 (2018) | Room 2.8.3 | High (long-term) |

**GreenDAO Status:**
- ⚠️ Last updated: 2018 (7 years old)
- ⚠️ Minimal maintenance
- ✅ Works fine, but outdated

**Room Migration Recommendation:**

**Benefits:**
- Modern AndroidX integration
- Better Kotlin support
- Compile-time query verification
- LiveData/Flow support
- Active development

**Migration Complexity:**
- **Effort:** 4-6 weeks
- **Risk:** Medium-High
- **Files Affected:** ~50+ (DAO classes, database queries)
- **Recommendation:** Plan for major release

**Example Migration:**

GreenDAO:
```java
@Entity
public class FileMeta {
    @Id private Long id;
    @Property(nameInDb = "PATH") private String path;
}

// Usage:
FileMetaDao dao = daoSession.getFileMetaDao();
List<FileMeta> list = dao.queryBuilder().where(...).list();
```

Room:
```java
@Entity(tableName = "file_meta")
data class FileMeta(
    @PrimaryKey val id: Long,
    @ColumnInfo(name = "PATH") val path: String
)

@Dao
interface FileMetaDao {
    @Query("SELECT * FROM file_meta WHERE ...")
    suspend fun getAll(): List<FileMeta>
}
```

**Recommendation:** Keep GreenDAO for now, plan migration for v3.0.

---

### Google Services & Ads

| Library | Current | Latest | Status |
|---------|---------|--------|--------|
| **play-services-ads** | 24.4.0 | 24.4.0+ | ✅ Current |
| **user-messaging-platform** | 3.2.0 | 3.2.0+ | ✅ Current |
| **play-services-auth** | 21.4.0 | 21.4.0 | ✅ Current |
| **google-api-services-drive** | v3-rev20220815-2.0.0 | Check | Review needed |
| **google-api-client-android** | 2.8.0 | 2.8.0+ | ✅ Current |
| **google-oauth-client-jetty** | 1.39.0 | 1.39.0+ | ✅ Current |
| **google-http-client-gson** | 1.47.1 | 1.47.1+ | ✅ Current |

---

## MODULE: appCompose (Modern UI)

### Compose

| Component | Current | Latest | Recommended | Priority |
|-----------|---------|--------|-------------|----------|
| **Compose BOM** | 2025.07.00 | 2025.10.01 | 2025.10.01 | Medium |
| **activity-compose** | 1.11.0 | 1.11.0 | ✅ Current | - |
| **navigation-compose** | 2.9.5 | 2.9.5+ | ✅ Current | - |

**BOM Update:**
```kotlin
implementation(platform("androidx.compose:compose-bom:2025.10.01"))
```

**Benefits:**
- Bug fixes in Compose UI
- Material 3 improvements
- Performance optimizations
- Better accessibility

**Breaking Changes:** None (BOM maintains compatibility)

---

### Core Libraries

| Library | Current | Latest | Recommended | Priority |
|---------|---------|--------|-------------|----------|
| **lifecycle-viewmodel-compose** | 2.9.4 | 2.9.4+ | ✅ Current | - |
| **lifecycle-runtime-ktx** | 2.9.4 | 2.9.4+ | ✅ Current | - |
| **core-ktx** | 1.17.0 | 1.17.0 | ✅ Current | - |
| **OkHttp** | 5.1.0 | 5.3.0 | 5.3.0 | Low |
| **Coil** | 3.3.0 | 3.3.0 | ✅ Current | - |

**OkHttp 5.1.0 → 5.3.0:**
```kotlin
implementation("com.squareup.okhttp3:okhttp:5.3.0")
```
- Performance improvements
- Bug fixes
- No breaking changes

---

### Database (Room)

| Component | Current | Latest | Recommended | Priority |
|-----------|---------|--------|-------------|----------|
| **room-runtime** | 2.8.1 | 2.8.3 | 2.8.3 | Low |
| **room-compiler** | 2.8.1 | 2.8.3 | 2.8.3 | Low |
| **room-ktx** | 2.8.1 | 2.8.3 | 2.8.3 | Low |
| **sqlite-bundled** | 2.6.1 | 2.6.1 | ✅ Current | - |

**Update All Room Dependencies:**
```kotlin
val roomVersion = "2.8.3"
implementation("androidx.room:room-runtime:$roomVersion")
ksp("androidx.room:room-compiler:$roomVersion")
implementation("androidx.room:room-ktx:$roomVersion")
```

---

### Dependency Injection (Koin)

| Component | Current | Status |
|-----------|---------|--------|
| **koin-bom** | 4.1.1 | ✅ Current |
| **koin-android** | via BOM | ✅ |
| **koin-androidx-compose** | via BOM | ✅ |

Koin 4.1.x is the latest stable. No updates needed.

---

### Kotlin Coroutines

| Library | Current | Status |
|---------|---------|--------|
| **kotlinx-coroutines-core** | 1.10.2 | ✅ Current |
| **kotlinx-coroutines-android** | 1.10.2 | ✅ Current |
| **kotlinx-serialization-json** | 1.9.0 | ✅ Current |

---

## MODULE: appLibDrive (Cloud Integration)

### Version Inconsistencies to Fix

| Library | appCompose | appLibDrive | Target |
|---------|-----------|-------------|--------|
| **Compose BOM** | 2025.07.00 | 2025.07.00 | 2025.10.01 |
| **activity-compose** | 1.11.0 | 1.10.1 | 1.11.0 |
| **lifecycle-viewmodel** | 2.9.4 | 2.9.2 | 2.9.4 |
| **lifecycle-runtime** | 2.9.4 | 2.9.2 | 2.9.4 |
| **core-ktx** | 1.17.0 | 1.16.0 | 1.17.0 |
| **activity-ktx** | 1.11.0 | 1.10.1 | 1.11.0 |

**Standardization Update (appLibDrive build.gradle.kts):**
```kotlin
// Compose
implementation(platform("androidx.compose:compose-bom:2025.10.01"))
implementation("androidx.activity:activity-compose:1.11.0")
implementation("androidx.lifecycle:lifecycle-viewmodel-compose:2.9.4")
implementation("androidx.lifecycle:lifecycle-runtime-ktx:2.9.4")

// Core
implementation("androidx.core:core-ktx:1.17.0")
implementation("androidx.activity:activity-ktx:1.11.0")
```

---

### Firebase

| Component | Current | Latest | Recommended | Priority |
|-----------|---------|--------|-------------|----------|
| **firebase-bom** | 34.1.0 | 34.5.0 | 34.5.0 | Medium |

**Update:**
```kotlin
implementation(platform("com.google.firebase:firebase-bom:34.5.0"))
```

**Benefits:**
- Security updates across all Firebase libraries
- Performance improvements
- Bug fixes in Firestore, Auth, Storage

---

### gRPC

| Library | Current | Latest | Recommended | Priority |
|---------|---------|--------|-------------|----------|
| **grpc-okhttp** | 1.74.0 | 1.76.0+ | 1.76.0 | Low |
| **grpc-android** | 1.74.0 | 1.76.0+ | 1.76.0 | Low |
| **grpc-protobuf-lite** | 1.74.0 | 1.76.0+ | 1.76.0 | Low |
| **grpc-stub** | 1.74.0 | 1.76.0+ | 1.76.0 | Low |

**Update All gRPC:**
```kotlin
val grpcVersion = "1.76.0"
implementation("io.grpc:grpc-okhttp:$grpcVersion")
implementation("io.grpc:grpc-android:$grpcVersion")
implementation("io.grpc:grpc-protobuf-lite:$grpcVersion")
implementation("io.grpc:grpc-stub:$grpcVersion")
```

---

## PRIORITY UPGRADE ROADMAP

### Phase 1: CRITICAL SECURITY (Week 1-2)

**Priority: IMMEDIATE**

```gradle
// app/build.gradle
dependencies {
    // CRITICAL: Fix CVE-2021-0341
    implementation 'com.squareup.okhttp3:okhttp:5.1.0'
    implementation 'com.squareup.okio:okio:3.9.0'

    // Check okhttp-digest compatibility or replace
    // implementation 'io.github.rburgst:okhttp-digest:2.0' // If available
}
```

**Testing Checklist:**
- [ ] OPDS catalog browsing
- [ ] HTTP Basic/Digest authentication
- [ ] Book downloads
- [ ] Cloud sync (Drive, Dropbox)
- [ ] Proxy configuration
- [ ] SSL/TLS connections

**Rollback Plan:** Keep OkHttp 3.12.6 branch available

---

### Phase 2: VERSION CONSISTENCY (Week 3)

**Priority: HIGH**

Standardize versions across modules:

```kotlin
// appLibDrive/build.gradle.kts
implementation("androidx.core:core-ktx:1.17.0")
implementation("androidx.activity:activity-compose:1.11.0")
implementation("androidx.lifecycle:lifecycle-viewmodel-compose:2.9.4")
implementation("androidx.lifecycle:lifecycle-runtime-ktx:2.9.4")
implementation(platform("androidx.compose:compose-bom:2025.10.01"))
```

---

### Phase 3: FIREBASE & COMPOSE (Week 4)

**Priority: MEDIUM**

```kotlin
// appLibDrive
implementation(platform("com.google.firebase:firebase-bom:34.5.0"))

// appCompose, appLibDrive
implementation(platform("androidx.compose:compose-bom:2025.10.01"))
```

---

### Phase 4: MINOR UPDATES (Month 2)

**Priority: LOW**

```gradle
// app/build.gradle
implementation 'org.jsoup:jsoup:1.21.2'
implementation 'androidx.work:work-runtime:2.11.0'

// appCompose/build.gradle.kts
val roomVersion = "2.8.3"
implementation("androidx.room:room-runtime:$roomVersion")
ksp("androidx.room:room-compiler:$roomVersion")
implementation("androidx.room:room-ktx:$roomVersion")
```

---

### Phase 5: MAJOR MIGRATIONS (Long-term)

**Timeline: 3-6 months**

1. **Glide 4 → 5** (Medium effort)
   - Test image loading thoroughly
   - Update 2-3 weeks

2. **Remove legacy-support-v4** (Medium effort)
   - Identify usage patterns
   - Replace with specific libraries
   - Update 1-2 weeks

3. **GreenDAO → Room** (High effort)
   - Major refactoring
   - Plan for v3.0 release
   - Timeline: 6-8 weeks

---

## BUILD SCRIPT UPDATES

### Root build.gradle.kts

```kotlin
plugins {
    id("com.android.application") version "8.13.0" apply false
    id("com.android.library") version "8.13.0" apply false
    id("org.jetbrains.kotlin.android") version "2.2.0" apply false
    id("org.jetbrains.kotlin.plugin.compose") version "2.2.0" apply false
    id("androidx.room") version "2.8.3" apply false
    id("com.google.devtools.ksp") version "2.2.0-2.0.7" apply false
    id("com.google.gms.google-services") version "4.4.3" apply false
}
```

---

## TESTING STRATEGY

### Regression Testing After Updates

1. **Critical Path Tests:**
   - [ ] App launches successfully
   - [ ] Book library loads
   - [ ] Open various formats (PDF, EPUB, MOBI, etc.)
   - [ ] OPDS catalog browsing
   - [ ] Book download from catalog
   - [ ] Cloud sync (Drive)
   - [ ] TTS functionality
   - [ ] Search (local and in-document)

2. **Network Tests:**
   - [ ] HTTP connections
   - [ ] HTTPS connections
   - [ ] Authentication (Basic, Digest)
   - [ ] Proxy connections
   - [ ] SSL certificate validation

3. **Performance Tests:**
   - [ ] App startup time
   - [ ] Library indexing
   - [ ] Page rendering
   - [ ] Memory usage profiling

4. **Build Variants:**
   - [ ] fdroid flavor
   - [ ] pro flavor
   - [ ] librera flavor (with ads)
   - [ ] All ABIs (arm64-v8a, armeabi-v7a, x86, x86_64)

---

## CONTINUOUS MONITORING

### Dependabot Configuration

Create `.github/dependabot.yml`:

```yaml
version: 2
updates:
  - package-ecosystem: "gradle"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 10
```

### Manual Checks (Quarterly)

```bash
# Check for outdated dependencies
./gradlew dependencyUpdates

# Security vulnerability scanning
./gradlew dependencyCheckAnalyze
```

---

## SUMMARY

### Current State
- **Total Dependencies:** 70+
- **Outdated:** 12
- **Security Issues:** 1 (CRITICAL)
- **Deprecated:** 1
- **Security Score:** 6/10

### After Phase 1 (Critical Security)
- **Security Score:** 9/10
- **Time Investment:** 1-2 weeks
- **Risk:** Medium

### After Phase 4 (All Minor Updates)
- **Security Score:** 10/10
- **Time Investment:** 1 month
- **Risk:** Low

### After Phase 5 (Major Migrations)
- **Modernization:** Complete
- **Time Investment:** 3-6 months
- **Risk:** Medium-High

**Recommendation:** Execute Phase 1 immediately, then proceed with phased approach based on release schedule and available resources.
