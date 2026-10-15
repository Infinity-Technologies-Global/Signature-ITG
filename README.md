# Signature-ITG — Standard Android authentication for all API Services

The standard guide for integrating authentication into an Android app that calls ITG API Services.

The library provides the value of the **`authorization`** header. Your app only passes a `Context` and
the parameters to a function, then puts the returned value into the request header.

- Two authentication versions are supported: **V1** and **V2**.
- Each API domain states which version it uses (V1 or V2) in its own API documentation.
- `minSdk 24`.

---

## 1. Installation (JitPack)

**Step 1.** Add the JitPack repository:

```groovy
// settings.gradle
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
        maven { url 'https://jitpack.io' }
    }
}
```

If your project uses the legacy setup:

```groovy
// build.gradle (root)
allprojects {
    repositories {
        google()
        mavenCentral()
        maven { url 'https://jitpack.io' }
    }
}
```

**Step 2.** Add the dependency to your app module:

```groovy
// app/build.gradle
dependencies {
    implementation 'com.github.Infinity-Technologies-Global:Signature-ITG:1.0.1'
}
```

**Step 3.** Sync Gradle. No extra ProGuard/R8 configuration is needed.

---

## 2. Functions

Class: `com.example.signaturelib.SignatureValidator` (static functions, callable from Kotlin and Java).

| Function | Used for | Parameters | Returns |
|---|---|---|---|
| `getAuthorizationV1(context, timestamp)` | **V1** authentication | `context: Context`, `timestamp: Long` | `String` — value of the `authorization` header |
| `getAuthorizationV2(context, payload)` | **V2** authentication | `context: Context`, `payload: Map<String, Any?>` | `String` — value of the `authorization` header |

Parameters:

| Parameter | Description |
|---|---|
| `context` | The application `Context` (prefer `applicationContext`) |
| `timestamp` | Current time as a Unix timestamp in **milliseconds**. Use the same value as the request's `x-timestamp` header |
| `payload` | The fields required by the **API domain documentation**. Field names and value types must match the documentation. Must not be empty |

Return value: both functions always return a `String` and never return `null`.
An **empty string** means the value could not be created. In that case do not send the request.

---

## 3. Request headers

Which headers are required and what their values are is defined by the **documentation of each API domain**.
The usual set of headers:

| Header | Value comes from |
|---|---|
| `x-package-name` | Your app (`context.packageName`) |
| `x-version-name` | Your app (version name) |
| `x-timestamp` | Your app (Unix timestamp in milliseconds) |
| `x-timezone` | Your app (depends on the domain, e.g. `Asia/Ho_Chi_Minh`) |
| `x-secret-key` | Your app (only some domains require it, value per the domain documentation) |
| **`authorization`** | **The value returned by `getAuthorizationV1` or `getAuthorizationV2`** |

Notes:

- Compute `timestamp` right before sending the request; do not reuse an old value.
- Use one single `timestamp` value for both the `x-timestamp` header and the function parameter.

---

## 4. Usage

### 4.1 V1

```kotlin
import com.example.signaturelib.SignatureValidator

val timestamp = System.currentTimeMillis()
val authorization = SignatureValidator.getAuthorizationV1(context, timestamp)

if (authorization.isEmpty()) {
    // Could not create the value -> do not send the request
}

val request = Request.Builder()
    .url(url)
    .header("x-package-name", context.packageName)
    .header("x-version-name", versionName)
    .header("x-timestamp", timestamp.toString())
    .header("authorization", authorization)
    // .header("x-secret-key", "...")   // if the domain documentation requires it
    .build()
```

Java:

```java
long timestamp = System.currentTimeMillis();
String authorization = SignatureValidator.getAuthorizationV1(context, timestamp);
```

### 4.2 V2

```kotlin
import com.example.signaturelib.SignatureValidator

val timestamp = System.currentTimeMillis()

// Fields as defined by the API domain documentation
val payload = mapOf<String, Any?>(
    "ts" to timestamp
    // "field" to value
)

val authorization = SignatureValidator.getAuthorizationV2(context, payload)

if (authorization.isEmpty()) {
    // Could not create the value -> do not send the request
}

val request = Request.Builder()
    .url(url)
    .header("x-package-name", context.packageName)
    .header("x-version-name", versionName)
    .header("x-timestamp", timestamp.toString())
    .header("authorization", authorization)
    .build()
```

Java:

```java
Map<String, Object> payload = new HashMap<>();
payload.put("ts", System.currentTimeMillis());
String authorization = SignatureValidator.getAuthorizationV2(context, payload);
```

### 4.3 Example: adding headers with an OkHttp Interceptor

```kotlin
class AuthInterceptor(
    private val context: Context,
    private val versionName: String,
    private val useV2: Boolean            // per the domain documentation
) : Interceptor {

    override fun intercept(chain: Interceptor.Chain): Response {
        val timestamp = System.currentTimeMillis()

        val authorization = if (useV2) {
            SignatureValidator.getAuthorizationV2(context, mapOf("ts" to timestamp))
        } else {
            SignatureValidator.getAuthorizationV1(context, timestamp)
        }
        if (authorization.isEmpty()) {
            throw IOException("Cannot create authorization")
        }

        val request = chain.request().newBuilder()
            .header("x-package-name", context.packageName)
            .header("x-version-name", versionName)
            .header("x-timestamp", timestamp.toString())
            .header("authorization", authorization)
            .build()
        return chain.proceed(request)
    }
}
```

---

## 5. Support

If the server returns an authentication error (for example `401`) or a function returns an empty string:

1. Check your parameters against section 2 and your headers against section 3.
2. Compare with the API documentation of the domain.
3. If the problem persists, contact the ITG team in charge of the domain.

---

## 6. Versions

| Version | Notes |
|---|---|
| `1.0.1` | `getAuthorizationV1` / `getAuthorizationV2` always return a `String` (empty on failure) |
| `1.0.0` | First release: V1 and V2 authentication |
