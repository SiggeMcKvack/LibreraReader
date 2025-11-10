# OPDS v1.2 Implementation Review

**Audit Date:** 2025-11-10
**Auditor:** Claude Code
**Specification:** OPDS Catalog 1.2 (https://specs.opds.io/opds-1.2.html)

---

## Executive Summary

LibreraReader implements a **functional but non-compliant** OPDS 1.2 client. While it successfully handles common use cases and popular OPDS servers, it has several specification violations and missing features that could cause compatibility issues.

**Compliance Score:** 6/10

**Status:**
- ✅ **Implemented:** 18 features
- ⚠️ **Partial/Incomplete:** 6 features
- ❌ **Missing/Violated:** 12 features

**Critical Issues:**
- Missing required feed `<id>` element
- Missing feed-level `<author>` element
- Incomplete link relation support
- No namespace handling
- Limited OpenSearch integration

---

## CODE STRUCTURE

### Core OPDS Package

Located in `app/src/main/java/com/foobnix/opds/`:

| File | Lines | Purpose |
|------|-------|---------|
| **OPDS.java** | 403 | HTTP client, XML parsing, authentication |
| **Feed.java** | 49 | Feed model |
| **Entry.java** | 86 | Entry/publication model |
| **Link.java** | 148 | Link with MIME types and relations |
| **Hrefs.java** | 58 | URL normalization utilities |
| **OpdsItem.java** | 51 | Serializable catalog item |
| **SamlibOPDS.java** | 515 | Custom HTML→OPDS converter |
| **WebviewCookieHandler.java** | 64 | Cookie management |

### UI Components

- **OpdsFragment2.java** (893 lines): Main OPDS browser UI
- **EntryAdapter.java** (355 lines): RecyclerView adapter
- **AddCatalogDialog.java** (304 lines): Catalog management, authentication

---

## SPECIFICATION COMPLIANCE

### ✅ IMPLEMENTED FEATURES

#### Feed-Level Elements

**`<title>`** ✅
- **Location:** OPDS.java:303
- **Implementation:** Correctly parsed
```java
if (tagName.equalsIgnoreCase("title")) {
    feed.title = xpp.nextText();
}
```

**`<subtitle>`** ✅
- **Location:** OPDS.java:306
- **Status:** Correctly parsed

**`<updated>`** ✅
- **Location:** OPDS.java:308-309
- **Implementation:** Parsed and converted to date
```java
if (tagName.equalsIgnoreCase("updated")) {
    feed.updated = ISO8601.toDate(xpp.nextText());
}
```

**`<icon>`** ✅
- **Location:** OPDS.java:311-312
- **Status:** Correctly parsed

**`<link>` elements** ✅
- **Location:** OPDS.java:314-316
- **Status:** Parsed into Link objects

---

#### Entry-Level Elements

**`<title>`** ✅
- **Location:** OPDS.java:329-331, 376-378
- **Implementation:** Handles both tag content and text nodes
```java
if (tagName.equalsIgnoreCase("title")) {
    isTitle = true;
}
// ... later in TEXT event
if (isTitle) {
    entry.title += " " + xpp.getText();
}
```

**Note:** Has minor bug - concatenates with space, can create extra spaces

**`<id>`** ✅
- **Location:** OPDS.java:343-344
```java
if (tagName.equalsIgnoreCase("id")) {
    entry.id = xpp.nextText();
}
```

**`<updated>`** ✅
- **Location:** OPDS.java:340-341

**`<author><name>`** ✅
- **Location:** OPDS.java:346-348
```java
if (tagName.equalsIgnoreCase("name") && isAuthor) {
    entry.authorName += TxtUtils.COMMA + xpp.nextText();
}
```

**`<author><uri>`** ✅
- **Location:** OPDS.java:349-351

**`<summary>`** ✅
- **Location:** OPDS.java:334-336

**`<content>`** ✅ (with issues)
- **Location:** OPDS.java:326-328, 369-375
- **Implementation:** Concatenates text nodes with `<br/>`
```java
if (isContent) {
    final String text = xpp.getText();
    if (TxtUtils.isNotEmpty(text)) {
        entry.content = entry.content + text + "<br/>";
    }
}
```

**Issue:** This breaks XHTML content. Should handle `type="xhtml"` attribute properly.

**`<category>`** ✅
- **Location:** OPDS.java:352-360
- **Implementation:** Captures both `label` and `term` attributes
```java
if (tagName.equalsIgnoreCase("category")) {
    String label = xpp.getAttributeValue(null, "label");
    String term = xpp.getAttributeValue(null, "term");
    // ... concatenates with ", "
}
```

**Issue:** Fragile string concatenation logic

**`<dc:issued>` (Dublin Core year)** ✅
- **Location:** OPDS.java:337-339

**`<link>` elements** ✅
- **Location:** OPDS.java:362-364

---

#### Link Relations

**`rel="search"`** ✅
- **Location:** Link.java:83
```java
public boolean isSeacrh() {
    return "search".equalsIgnoreCase(rel);
}
```

**`rel="thumbnail"`** ✅
- **Location:** Link.java:78-79
- **Supports:** Both `http://opds-spec.org/image/thumbnail` and `http://opds-spec.org/thumbnail`

**`rel="next"`** ✅
- **Location:** OpdsFragment2.java:751-755
- **Implementation:** Handles pagination
```java
if (feed.links != null) {
    for (Link l : feed.links) {
        if (Link.NEXT_LINK.equals(l.rel)) {
            nextUrl = l.href;
        }
    }
}
```

**Navigation vs Acquisition Detection** ✅ (Partial)
- **Location:** Link.java:88
```java
public boolean isOpds() {
    return type != null && type.contains("opds");
}
```

Detects OPDS feeds by MIME type `application/atom+xml;profile=opds-catalog`

---

#### MIME Type Support

**Location:** Link.java:32-52

Comprehensive mapping including:
```java
ext.put("fb2", "application/fb2");
ext.put("epub", "application/epub+zip");
ext.put("mobi", "application/x-mobipocket-ebook");
ext.put("azw3", "application/x-mobi8-ebook");
ext.put("pdf", "application/pdf");
ext.put("djvu", "image/vnd.djvu");
ext.put("cbz", "application/x-cbz");
ext.put("cbr", "application/x-cbr");
// ... and many more
```

**Status:** Excellent coverage of ebook formats

---

#### Authentication

**HTTP Basic Auth** ✅
- **Location:** OPDS.java:123-124, 156-157
```java
okHttpClient = okHttpClient.newBuilder()
    .authenticator(new CachingAuthenticator(login, password))
    .build();
```

**HTTP Digest Auth** ✅
- **Location:** OPDS.java:125, 158
```java
.addInterceptor(new DigestAuthInterceptor(
    new Credentials(login, password)
))
```

**401 Handling** ✅
- **Location:** OPDS.java:117-119, 147-148, 176-179
- **Implementation:** Automatic retry with credentials

**Credential Storage** ✅
- **Location:** OPDS.java:250-267
- **Storage:** SharedPreferences by hostname
```java
String key = SP_OPDS_USER + host;
String value = login + TxtUtils.TTS_PAUSE + password;
sp.edit().putString(key, value).commit();
```

**Security Issue:** ⚠️ Credentials stored in plain text

---

#### HTTP Features

**OkHttp3 Client** ✅
- **Location:** OPDS.java:45-52
- **Features:** Connection pooling, cookie jar integration

**Proxy Support** ✅
- **Location:** OPDS.java:59-79
- **Types:** HTTP and SOCKS proxy

**Caching** ✅
- **Location:** OPDS.java:97-98
- **Configuration:** 10-minute cache with 5MB size

**Custom Headers** ✅
- User-Agent (line 57)
- Accept-Language (line 95)

---

### ⚠️ PARTIAL/INCOMPLETE FEATURES

#### 1. OpenSearch Integration (Partial)

**Implemented:**
- Search link detection ✅ (Link.java:83)
- Search UI rendering ✅ (EntryAdapter.java:162-208)
- Template parameter substitution ✅ (line 189)
```java
String searchUrl = link.href.replace("{searchTerms}", query);
```

**Missing:**
- ❌ OpenSearch Description Document parsing
- ❌ `totalResults`, `startIndex`, `itemsPerPage` handling
- ❌ Namespace `xmlns:opensearch` support

**Impact:** Only inline search links work, not separate OpenSearch descriptors

---

#### 2. Content Type Handling (Partial)

**Implemented:**
- Text content ✅
- HTML content ✅ (sort of)

**Issues:**
- ❌ No proper XHTML content handling
- ❌ No `type="xhtml"` attribute check
- ❌ Breaks structured content by concatenating with `<br/>`

---

#### 3. Link Relations (Incomplete)

**Implemented:**
- `search` ✅
- `thumbnail` ✅
- `next` ✅

**Missing:**
- ❌ `previous` (no pagination backwards)
- ❌ `first`, `last` (no jump to start/end)
- ❌ `self` (not used)
- ❌ `start`, `up` (no navigation hierarchy)
- ❌ `subsection` (not detected)

---

### ❌ MISSING/SPEC VIOLATIONS

#### 1. Missing Required `<id>` at Feed Level 🔴

**Severity:** CRITICAL SPEC VIOLATION

**Location:** Feed.java:8
```java
public String id;  // Declared but NEVER populated
```

**Specification Requirement:**
> Each Feed MUST have an id element that conveys a permanent, universally unique identifier for the Feed.

**Impact:**
- Cannot uniquely identify feeds
- Cache invalidation issues
- Violates Atom syndication format

**Fix Required:**
```java
// In OPDS.java parsing:
if (tagName.equalsIgnoreCase("id") && !isEntry) {
    feed.id = xpp.nextText();
}
```

---

#### 2. Missing Feed-Level `<author>` Element 🔴

**Severity:** HIGH - SPEC VIOLATION

**Status:** Not parsed at all

**Specification Requirement:**
> Each Feed MUST have at least one author element

**Current Implementation:** Only parses entry-level authors

**Fix Required:**
```java
boolean isFeedAuthor = false;

if (tagName.equalsIgnoreCase("author") && !isEntry) {
    isFeedAuthor = true;
}

if (tagName.equalsIgnoreCase("name") && isFeedAuthor) {
    feed.authorName = xpp.nextText();
}
```

---

#### 3. No Acquisition Link Type Differentiation ⚠️

**Severity:** MEDIUM

**Problem:** All non-OPDS links treated as downloads

**Location:** EntryAdapter.java:235-241
```java
if (!link.isOpds()) {
    // Assumes all are acquisition links
    // No distinction between buy/borrow/subscribe/sample
}
```

**Missing Relations:**
- `http://opds-spec.org/acquisition`
- `http://opds-spec.org/acquisition/buy`
- `http://opds-spec.org/acquisition/borrow`
- `http://opds-spec.org/acquisition/sample`
- `http://opds-spec.org/acquisition/subscribe`

**Impact:** User cannot distinguish between purchase, borrow, and free acquisitions

**Fix Required:**
```java
// In Link.java
public enum AcquisitionType {
    OPEN_ACCESS,
    BUY,
    BORROW,
    SAMPLE,
    SUBSCRIBE,
    GENERIC
}

public AcquisitionType getAcquisitionType() {
    if (rel == null) return null;
    if (rel.contains("open-access")) return AcquisitionType.OPEN_ACCESS;
    if (rel.contains("buy")) return AcquisitionType.BUY;
    if (rel.contains("borrow")) return AcquisitionType.BORROW;
    // ...
}
```

---

#### 4. No Namespace Handling ⚠️

**Severity:** MEDIUM

**Problem:** XmlPullParser not configured for namespaces

**Location:** OPDS.java:287-290
```java
XmlPullParserFactory factory = XmlPullParserFactory.newInstance();
factory.setFeature(Xml.FEATURE_RELAXED, true);
XmlPullParser xpp = factory.newPullParser();
// No namespace awareness!
```

**Impact:**
- May fail with prefixed elements (`opds:price`, `dc:issued`)
- Cannot properly handle `xmlns:opds`, `xmlns:dc`
- Fragile when servers use different namespace prefixes

**Fix Required:**
```java
XmlPullParserFactory factory = XmlPullParserFactory.newInstance();
factory.setNamespaceAware(true);  // ✅ Enable namespaces
factory.setFeature(Xml.FEATURE_RELAXED, true);
```

Then update parsing to use namespace URIs:
```java
String ns = xpp.getNamespace();
String localName = xpp.getName();

if ("http://purl.org/dc/terms/".equals(ns) && "issued".equals(localName)) {
    // Handle dc:issued
}
```

---

#### 5. Missing Faceted Browsing Support ❌

**Status:** Not implemented

**Missing Features:**
- `rel="http://opds-spec.org/facet"`
- `opds:facetGroup` attribute
- `opds:activeFacet` attribute

**Impact:** Cannot use advanced filtering/sorting on OPDS servers that support it

---

#### 6. Incomplete Pagination ⚠️

**Implemented:**
- `rel="next"` ✅

**Missing:**
- `rel="previous"` ❌ (no backward navigation)
- `rel="first"` ❌
- `rel="last"` ❌
- Page number display ❌

**Impact:** User can only go forward in paginated feeds, cannot jump to start/end

---

#### 7. Missing Entry Elements ⚠️

**`<published>`** ❌
- Not parsed (uses `dc:issued` instead)

**`<rights>`** ❌
- License/copyright information not captured

**`<contributor>`** ❌
- Additional contributors not parsed

**Multiple Authors** ⚠️ (Partial)
- Concatenates multiple `<author>` elements into single string
- No structured author list

---

#### 8. Missing `opds:price` Support ❌

**Status:** Not implemented

**Specification:**
```xml
<link rel="http://opds-spec.org/acquisition/buy" ...>
  <opds:price currencycode="USD">9.99</opds:price>
</link>
```

**Impact:** Cannot display book prices for purchase links

---

#### 9. Missing `opds:indirectAcquisition` Support ❌

**Status:** Not implemented

**Purpose:** Indicates format transformations (e.g., DRM, encryption)

**Example:**
```xml
<link rel="http://opds-spec.org/acquisition/buy" type="application/epub+zip">
  <opds:indirectAcquisition type="application/epub+zip">
    <opds:indirectAcquisition type="application/vnd.adobe.adept+xml"/>
  </opds:indirectAcquisition>
</link>
```

**Impact:** Cannot warn users about DRM or format conversions

---

#### 10. No Navigation vs Acquisition Feed Distinction ⚠️

**Problem:** All feeds treated identically

**Specification:** Navigation feeds should not show acquisition buttons

**Current Implementation:** May show "Download" buttons on navigation entries

---

## BUGS AND ISSUES

### Bug 1: Title Parsing Creates Extra Spaces

**Location:** OPDS.java:376-378
```java
if (isTitle) {
    entry.title += " " + xpp.getText();
}
```

**Problem:** If title has multiple text nodes, they're concatenated with spaces: `"Book   Title"`

**Fix:**
```java
if (isTitle) {
    String text = xpp.getText().trim();
    if (!text.isEmpty()) {
        if (!entry.title.isEmpty()) {
            entry.title += " ";
        }
        entry.title += text;
    }
}
```

Or use `xpp.nextText()` instead of manual accumulation.

---

### Bug 2: Category Concatenation Logic

**Location:** OPDS.java:356-359, 383
```java
// Adds ", " before each category
entry.categories += ", " + categoryLabel;

// Removes first ", " at end
if (entry.categories.startsWith(", ")) {
    entry.categories = entry.categories.substring(2);
}
```

**Problem:** Fragile - empty categories will break the logic

**Fix:**
```java
List<String> categories = new ArrayList<>();
// ... collect categories

entry.categories = String.join(", ", categories);
```

---

### Bug 3: Relative URL Handling

**Location:** Hrefs.java:15-16
```java
if (!href.contains("://")) {
    href = "http://" + href;  // Assumes HTTP
}
```

**Problem:** Should use base URL from feed, not assume HTTP

**Fix:**
```java
public static String fixHref(String href, String baseUrl) {
    if (!href.contains("://")) {
        try {
            URL base = new URL(baseUrl);
            URL absolute = new URL(base, href);
            return absolute.toString();
        } catch (MalformedURLException e) {
            return "http://" + href;  // Fallback
        }
    }
    return href;
}
```

---

### Bug 4: Proxy Credentials Logged

**Location:** OPDS.java:68-70
```java
System.out.println("proxy-user " + proxyUser);
System.out.println("proxy-password " + proxyPassword);
```

**Security Risk:** Credentials in logs

**Fix:** Remove or use proper logging with redaction

---

### Bug 5: No Error Reporting

**Location:** OPDS.java:277-280
```java
} catch (Exception e) {
    LOG.e(e);
    return null;  // Silent failure
}
```

**Problem:** User has no idea why catalog loading failed

**Fix:**
```java
public class OPDSResult {
    Feed feed;
    String error;
    int statusCode;
}

public OPDSResult fetchFeed(String url) {
    try {
        // ...
        return new OPDSResult(feed, null, 200);
    } catch (Exception e) {
        return new OPDSResult(null, e.getMessage(), -1);
    }
}
```

---

## COMPATIBILITY ISSUES

### Works With
- ✅ Project Gutenberg (simple feeds)
- ✅ Internet Archive (basic OPDS)
- ✅ Most simple OPDS 1.x servers

### May Have Issues With
- ⚠️ Servers requiring proper feed IDs
- ⚠️ Servers using extensive namespacing
- ⚠️ Servers with complex faceted browsing
- ⚠️ Servers with DRM/indirect acquisition
- ⚠️ Servers with purchase workflows
- ⚠️ Servers with external OpenSearch descriptors

---

## RECOMMENDATIONS

### Critical (Must Fix)

1. **Add Feed ID Parsing** (1 hour)
   - Parse `<id>` at feed level
   - Essential for spec compliance

2. **Add Feed Author Parsing** (1 hour)
   - Parse `<author>` at feed level
   - Required by spec

3. **Enable Namespace Handling** (2 hours)
   - Configure XmlPullParser for namespaces
   - Update all tag checks to use namespaces
   - Test with various OPDS servers

---

### High Priority (Should Fix)

4. **Implement Acquisition Link Types** (4 hours)
   - Detect buy/borrow/subscribe/open-access
   - Show appropriate UI (price, rental period, etc.)
   - Warn users about purchases

5. **Fix Content Handling** (2 hours)
   - Check `type` attribute
   - Handle XHTML content properly
   - Don't break structured content

6. **Add Backward Pagination** (2 hours)
   - Detect `rel="previous"`
   - Add "Previous" button in UI
   - Track navigation history

---

### Medium Priority (Nice to Have)

7. **Add Price Support** (3 hours)
   - Parse `opds:price` elements
   - Display in UI
   - Format by currency

8. **Implement Full OpenSearch** (4 hours)
   - Parse OpenSearch Description Documents
   - Support all query parameters
   - Show result counts

9. **Add Faceted Browsing** (6 hours)
   - Parse facet links
   - Show filter/sort options in UI
   - Indicate active facets

---

### Long-Term Improvements

10. **Better Error Handling** (2 weeks)
    - Specific error messages
    - Retry logic
    - Offline mode with cache

11. **Advanced Features** (1 month)
    - DRM detection and warnings
    - Subscription management
    - Reading lists/wish lists
    - Catalog recommendations

12. **OPDS 2.0 Support** (2-3 months)
    - JSON-based catalogs
    - Modern API
    - Better metadata

---

## TESTING RECOMMENDATIONS

### Test Catalogs

Test against these OPDS servers:

1. **Project Gutenberg**
   - URL: `https://m.gutenberg.org/ebooks.opds/`
   - Tests: Basic feed parsing, pagination

2. **Internet Archive**
   - URL: `https://bookserver.archive.org/catalog/`
   - Tests: Complex feeds, many entries

3. **Standard Ebooks**
   - URL: `https://standardebooks.org/opds/`
   - Tests: Modern OPDS features

4. **Feedbooks**
   - URL: `https://www.feedbooks.com/publicdomain/catalog.atom`
   - Tests: Categories, filtering

5. **OPDS Test Server**
   - Set up local test server with edge cases
   - Tests: All spec features, error conditions

---

### Test Cases

**Basic Functionality:**
- [ ] Parse simple feed
- [ ] Navigate to subfeed
- [ ] Download book from catalog
- [ ] Search in catalog
- [ ] Page through results

**Authentication:**
- [ ] HTTP Basic Auth
- [ ] HTTP Digest Auth
- [ ] Remember credentials
- [ ] Handle 401 errors

**Edge Cases:**
- [ ] Empty feeds
- [ ] Feeds with missing elements
- [ ] Malformed XML
- [ ] Network errors
- [ ] Slow connections
- [ ] Large feeds (1000+ entries)

**Compatibility:**
- [ ] Namespace prefixes (opds:, dc:, atom:)
- [ ] Different date formats
- [ ] Relative URLs
- [ ] Various MIME types
- [ ] International characters (UTF-8)

---

## SUMMARY

### Strengths
- ✅ Functional for common use cases
- ✅ Good authentication support
- ✅ Comprehensive MIME type handling
- ✅ Decent HTTP client configuration
- ✅ Works with popular OPDS servers

### Weaknesses
- ❌ Spec violations (missing required elements)
- ❌ No namespace handling
- ❌ Incomplete link relation support
- ❌ Limited OpenSearch integration
- ❌ No acquisition type differentiation
- ❌ Poor error handling

### Compliance Score Breakdown

| Category | Score | Weight | Weighted |
|----------|-------|--------|----------|
| **Required Elements** | 4/6 | 30% | 2.0/3.0 |
| **Optional Elements** | 8/10 | 20% | 1.6/2.0 |
| **Link Relations** | 3/8 | 15% | 0.6/1.5 |
| **Authentication** | 5/5 | 15% | 1.5/1.5 |
| **Search** | 2/4 | 10% | 0.5/1.0 |
| **Error Handling** | 1/5 | 10% | 0.2/1.0 |
| **Total** | **23/38** | **100%** | **6.4/10** |

### Overall Assessment

**Rating:** 6.4/10 - "Functional but Non-Compliant"

LibreraReader's OPDS implementation is good enough for everyday use with popular servers, but lacks the robustness and completeness required for full specification compliance. Priority should be given to fixing critical spec violations (feed ID, feed author) and implementing namespace support.

**Recommended Action Plan:**
1. Fix critical spec violations (Days 1-2)
2. Add namespace support (Day 3)
3. Implement acquisition types (Week 2)
4. Improve error handling (Week 3)
5. Plan OPDS 2.0 support (Month 2-3)
