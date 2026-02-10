# JCEF Browser Fixture Troubleshooting Guide

## Problem Description

When running UI tests with IntelliJ Remote Robot 0.11.23, JCEF webview tests fail with:

```
com.intellij.remoterobot.fixtures.JCefBrowserFixture$JCEFScriptExecutionError: Failed to execute script:
__cefBrowser was not initialized
__Query was not initialized
```

**Key Observations:**
- Tests pass on macOS locally with version 0.11.22
- Tests fail on Linux (Jenkins CI) with both 0.11.22 and 0.11.23
- Error occurs during JCEF browser initialization phase

## Root Causes

### 1. Missing Critical System Property
The `ide.browser.jcef.jsQueryPoolSize` system property is **required** but was missing from the build configuration. This property reserves callback slots for JCEF JavaScript queries.

### 2. JCEF Initialization Race Condition
On Linux systems (especially headless CI environments), JCEF takes longer to initialize. The original code had no retry logic, causing it to fail if JCEF wasn't ready immediately.

### 3. Linux-Specific JCEF Behavior
JCEF on Linux requires proper display configuration (Xvfb) and different rendering settings compared to macOS.

## Fixes Applied

### Fix 1: Added Required JCEF System Property ✅

**File:** `ui-test-example/build.gradle`

Added the critical system property:
```groovy
"-Dide.browser.jcef.jsQueryPoolSize=10000",
```

This reserves 10,000 callback slots for JCEF JavaScript queries. Each `JCefBrowserFixture` instance consumes one slot.

### Fix 2: Retry Logic for Initialization ✅

**File:** `remote-fixtures/src/main/kotlin/com/intellij/remoterobot/fixtures/JCefBrowserFixture.kt`

Enhanced `initializeBrowser()` with:
- **10 retry attempts** with 500ms delay between attempts
- **Better error messages** that indicate specific failure reasons
- **Warning logs** for each retry attempt to aid debugging

This handles timing issues where JCEF initialization is delayed on Linux systems.

### Fix 3: Linux-Specific JCEF Configuration ✅

**File:** `ui-test-example/build.gradle`

Added platform-specific properties:
```groovy
"-Dide.browser.jcef.headless.enabled=false",
"-Djcef.trace.cefbrowser=true",

// Linux-only:
"-Dide.browser.jcef.gpu.disable=false",
"-Djcef.browser.offScreenRendering=false",
```

## Additional Recommendations for Jenkins/CI

### 1. Xvfb Configuration
Ensure your Jenkins job has proper Xvfb setup:

```bash
export DISPLAY=:99.0
Xvfb -ac :99 -screen 0 1920x1080x24 &
sleep 10
```

### 2. Add Explicit Wait Before Tests
In your test setup, add a wait to ensure JCEF is fully loaded:

```kotlin
@BeforeEach
fun waitForJcefReady(remoteRobot: RemoteRobot) {
    // Give JCEF extra time to initialize on Linux
    if (remoteRobot.isLinux()) {
        Thread.sleep(2000)
    }
}
```

### 3. Increase Timeout Values
If you still see intermittent failures, consider increasing the initialization timeout in `JCefBrowserFixture.kt`:

```kotlin
private const val EXECUTE_JS_TIMEOUT_MS = 5000  // Increased from 3000
```

### 4. Check Jenkins System Requirements
Ensure your Jenkins agents have:
- **Java 17** (as specified in build.gradle)
- **libgbm1** and other JCEF dependencies: `sudo apt-get install libgbm1 libasound2`
- **Sufficient memory** (at least 4GB for IntelliJ + JCEF)

### 5. Enable JCEF Debug Logging
For detailed JCEF debugging, add to your build.gradle:

```groovy
"-Djcef.debug=true",
"-Dide.browser.jcef.log.level=TRACE",
```

## Debugging Steps

If issues persist after applying fixes:

### 1. Check JCEF Availability
Add this diagnostic code to your test:

```kotlin
fun checkJcefStatus(remoteRobot: RemoteRobot) {
    val jcefAvailable = remoteRobot.callJs("""
        try {
            const jbCefApp = com.intellij.ui.jcef.JBCefApp.getInstance()
            return jbCefApp.isSupported() + ""
        } catch (e) {
            return "Error: " + e.message
        }
    """)
    println("JCEF Available: $jcefAvailable")
}
```

### 2. Verify Component Hierarchy
Check that the webview component has the required parent:

```kotlin
val hasJbCefBrowser = remoteRobot.callJs("""
    let currentComponent = component;
    while (currentComponent !== null) {
        const jbCefBrowser = currentComponent.getClientProperty("JBCefBrowser.instance");
        if (jbCefBrowser !== null) {
            return "Found at: " + currentComponent.getClass().getName();
        }
        currentComponent = currentComponent.getParent();
    }
    return "Not found";
""")
```

### 3. Check Query Pool Status
Verify the query pool isn't exhausted:

```kotlin
val poolSize = System.getProperty("ide.browser.jcef.jsQueryPoolSize")
println("Configured JCEF query pool size: $poolSize")
```

## Version-Specific Notes

### Remote Robot 0.11.23
- Contains improvements to JCEF handling
- The fixes in this guide are compatible with this version

### Remote Robot 0.11.22
- Also works with these fixes
- May have slightly different behavior on edge cases

## Platform Differences

| Platform | JCEF Component | Initialization Time | Special Requirements |
|----------|----------------|---------------------|---------------------|
| **macOS** | JBCefOsrComponent | Fast (~100ms) | None |
| **Linux** | Canvas/JBCef | Slow (~1-3s) | Xvfb, GPU config |
| **Windows** | Canvas/JBCef | Medium (~500ms) | None |

## Success Criteria

After applying these fixes, you should see:
1. ✅ Tests pass consistently on Linux CI
2. ✅ No `__cefBrowser was not initialized` errors
3. ✅ Initialization completes within 5 seconds (with retries)
4. ✅ Warning logs showing successful retry attempts (if needed)

## Further Reading

- [IntelliJ JCEF Documentation](https://plugins.jetbrains.com/docs/intellij/jcef.html)
- [Remote Robot GitHub Issues](https://github.com/JetBrains/intellij-ui-test-robot/issues)
- [JCEF System Properties](https://github.com/JetBrains/intellij-community/blob/master/platform/platform-impl/src/com/intellij/ui/jcef/JBCefApp.java)

## Contact

If issues persist after trying all solutions, please file an issue with:
- Full error stack trace
- IDE logs from `build/idea-sandbox/system/log/`
- Platform details (OS, Java version, JCEF version)
- Output of diagnostic scripts above
