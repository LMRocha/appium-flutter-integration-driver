# Troubleshooting Guide

Common issues and solutions for the Appium Flutter Integration Driver.

---

## Table of Contents

- [Quick Diagnosis](#quick-diagnosis)
- [Session Creation Issues](#session-creation-issues)
- [Element Location Issues](#element-location-issues)
- [Gesture & Interaction Issues](#gesture--interaction-issues)
- [Port Forwarding Issues](#port-forwarding-issues)
- [WebView / Context Issues](#webview--context-issues)
- [Image Mocking Issues](#image-mocking-issues)
- [iOS-Specific Issues](#ios-specific-issues)
- [Android-Specific Issues](#android-specific-issues)
- [macOS-Specific Issues](#macos-specific-issues)
- [Performance Issues](#performance-issues)
- [CI/CD Issues](#cicd-issues)
- [Error Code Reference](#error-code-reference)
- [Debugging Techniques](#debugging-techniques)

---

## Quick Diagnosis

### Start Here

1. **Check Appium logs** - Run with `--log-level debug`
2. **Verify Flutter server** - Is `integrationDriver()` in your app?
3. **Check capabilities** - Platform name, automation name, app path
4. **Review error message** - Match to sections below

### Health Check Commands

```bash
# Verify Appium installation
appium --version  # Should be 3.x

# Verify driver installation
appium driver list | grep flutter-integration

# Check connected devices
adb devices          # Android
xcrun xctrace list devices  # iOS

# Test port availability
lsof -i :4723        # Appium port
lsof -i :9000-9020   # Flutter server ports (device)
lsof -i :10000-11000 # System ports (host)
```

---

## Session Creation Issues

### "Flutter server is not started"

**Error:**
```
Error: Flutter server is not started. Please make sure the application 
under test is configured properly.
```

**Causes & Solutions:**

| Cause | Solution |
|-------|----------|
| App not built with `integration_test` | Add `integration_test` dependency and call `integrationDriver()` |
| Wrong app installed | Verify `app` capability points to correct .apk/.ipa |
| Flutter server crashed | Check device logs for Flutter errors |
| Port forwarding failed | See [Port Forwarding Issues](#port-forwarding-issues) |
| App not launched | Increase `flutterServerLaunchTimeout` (default 5000ms) |

**Debug Steps:**
```bash
# Android: Check if server port is listening on device
adb shell netstat -tlnp | grep :9000

# iOS Simulator: Check localhost ports
lsof -i :9000

# Check Appium logs for server discovery attempts
# Look for: "Checking if flutter server is running on port XXXX"
```

**Fix: Add integration test driver to Flutter app**
```dart
// integration_test_driver.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:integration_test/integration_test_driver.dart';

void main() {
  integrationDriver(); // Starts Flutter test server
}
```

Build with:
```bash
flutter build apk --target=integration_test_driver.dart
```

---

### "The server version X does not satisfy the driver version requirement"

**Error:**
```
Error: The server version 0.0.17 does not satisfy the driver 
version requirement '>=0.0.18 <1.0.0'.
```

**Solution:**
Update Flutter `integration_test` package in your app:
```yaml
# pubspec.yaml
dev_dependencies:
  integration_test:
    sdk: flutter
  flutter_test:
    sdk: flutter
```
Run `flutter pub upgrade` and rebuild.

**Version Matrix:**
| Driver Version | Required Flutter Server |
|----------------|------------------------|
| 2.x | >=0.0.18 <1.0.0 |
| 1.x | >=0.0.18 <1.0.0 |

---

### "Unsupported platformName"

**Error:**
```
Error: Unsupported platformName: Windows. Only the following 
platforms are supported: iOS, Android, Mac
```

**Solution:**
Fix `platformName` capability - must be exactly `iOS`, `Android`, or `Mac` (case-insensitive).

---

### "Session creation failed: App not installed"

**Error:**
```
Error: The application at '/path/to/app.apk' does not exist 
or is not accessible
```

**Solutions:**
- Verify `app` capability path is correct and accessible
- For CI: Use absolute paths or `${{ github.workspace }}`
- For real devices: Ensure app is signed for that device

---

## Element Location Issues

### "Element not found" / "NoSuchElement"

**Common Causes:**

| Cause | Solution |
|-------|----------|
| Wrong locator strategy | Use `flutterByValueKey$` for Key widgets |
| Element not rendered yet | Use `flutterWaitForVisible` before interacting |
| Wrong context | Ensure `NATIVE_APP` context for Flutter elements |
| Dynamic content | Use `flutterByTextContaining` for partial matches |
| Element in scroll view | Use `flutterScrollTillVisible` |

**Debug: List all available strategies**
```javascript
// In test
console.log('Strategies:', browser.locatorStrategies);
// Or check driver.locatorStrategies
```

**Best Practice: Use ValueKeys**
```dart
// Flutter app - Add Keys to important widgets
TextField(key: Key('username_field'), ...)
ElevatedButton(key: Key('login_button'), ...)
```

```javascript
// Test - Find by Key (fastest, most reliable)
await browser.flutterByValueKey$('username_field').addValue('user');
await browser.flutterByValueKey$('login_button').click();
```

---

### "Stale element reference"

**Error:**
```
Error: stale element reference: element is not attached to the page
```

**Causes:**
- App navigated to new screen
- Flutter rebuild (StatefulWidget setState)
- Element removed from tree

**Solutions:**
```javascript
// Re-find element after navigation/state change
await browser.flutterWaitForVisible({ 
  locator: await browser.flutterByValueKey('button') 
});
const freshElement = await browser.flutterByValueKey$('button');
await freshElement.click();
```

---

### "Invalid selector" for descendant/ancestor

**Error:**
```
Error: invalid argument: invalid locator
```

**Cause:** Selector must be JSON string for complex locators.

**Correct Usage:**
```javascript
// Descendant - find child within parent
const child = await browser.flutterByDescendant$({
  of: await browser.flutterByValueKey('parent_card'),
  matching: await browser.flutterByText('Child Title'),
});

// The WDIO service handles JSON conversion automatically
// But raw executeScript needs JSON string:
await browser.executeScript('flutter: scrollTillVisible', [{
  finder: JSON.stringify({
    using: '-flutter descendant',
    selector: {
      of: { using: '-flutter key', value: 'parent_card' },
      matching: { using: '-flutter text', value: 'Child Title' }
    }
  })
}]);
```

---

### Element found but click doesn't work

**Symptoms:** No error, but UI doesn't respond.

**Causes & Solutions:**

| Cause | Solution |
|-------|----------|
| Element not clickable (disabled) | Check `await element.isEnabled()` |
| Element covered by overlay | Use `flutterWaitForVisible` with `displayed` check |
| Mac2Driver blocking | Known issue - driver uses non-blocking click |
| Wrong coordinates | Use `getElementRect` to verify position |

**Debug:**
```javascript
const rect = await element.getElementRect();
console.log('Element rect:', rect); // {x, y, width, height}
const displayed = await element.isDisplayed();
const enabled = await element.isEnabled();
```

---

## Gesture & Interaction Issues

### Double Click / Long Press Not Working

**Symptoms:** Command executes but no visual effect.

**Solutions:**

1. **Verify element is in Flutter context:**
```javascript
await browser.switchContext('NATIVE_APP');
```

2. **Use correct parameters:**
```javascript
// Double click with offset
await browser.flutterDoubleClick({
  element: await browser.flutterByValueKey$('button'),
  offset: { x: 10, y: 0 }  // Optional offset from center
});

// Long press
await browser.flutterLongPress({
  element: await browser.flutterByValueKey$('button'),
  offset: { x: 5, y: 5 }
});
```

3. **Check element is actually a Flutter element** (not native)

---

### Scroll Till Visible Not Finding Element

**Common Issues:**

| Issue | Solution |
|-------|----------|
| Wrong scroll direction | Try `'down'`, `'up'`, `'left'`, `'right'` |
| Scroll view not specified | Add `scrollView` parameter for nested scrollers |
| Element not in scroll path | Verify element is actually in the scrollable area |
| Delta too small/large | Adjust `delta` (default 100) |
| Max scrolls too low | Increase `maxScrolls` (default 50) |

**Debug:**
```javascript
const element = await browser.flutterScrollTillVisible({
  finder: await browser.flutterByText('Item 100'),
  scrollDirection: 'down',
  delta: 200,           // Scroll more per gesture
  maxScrolls: 100,      // Allow more attempts
  settleBetweenScrollsTimeout: 500,  // Wait longer between scrolls
});
```

---

### Drag and Drop Fails

**Requirements:**
- Both source and target must be Flutter elements
- Elements must be visible and interactable

```javascript
const source = await browser.flutterByValueKey$('draggable_item');
const target = await browser.flutterByValueKey$('drop_zone');

// Verify both exist
await expect(source).toBeDisplayed();
await expect(target).toBeDisplayed();

await browser.flutterDragAndDrop({ source, target });
```

---

## Port Forwarding Issues

### "Port #XXXX is occupied by another process"

**Error:**
```
Error: The port #10001 is occupied by another process. 
You can either quit that process or select another free port.
```

**Solutions:**

1. **Kill existing Appium processes:**
```bash
pkill -f appium
pkill -f iproxy
```

2. **Use fixed port:**
```javascript
capabilities: {
  'appium:flutterSystemPort': 10001  // Fixed system port
}
```

3. **Check what's using the port:**
```bash
lsof -i :10001
# Kill the process
kill -9 <PID>
```

---

### "ADB port forward failed" (Android)

**Error:**
```
Error: adb forward tcp:10001 tcp:9001 failed: 
device offline / no devices/emulators found
```

**Solutions:**
```bash
# Verify device connected
adb devices

# Restart ADB
adb kill-server && adb start-server

# Check emulator running
adb -e shell getprop sys.boot_completed

# Reconnect device
adb disconnect && adb connect <device-ip>:5555
```

---

### "iproxy connection failed" (iOS Real Device)

**Error:**
```
Error: Failed to start iproxy for device UDID on port XXXX
```

**Solutions:**
```bash
# Verify device trusted & connected
idevice_id -l

# Check libimobiledevice
brew install libimobiledevice  # macOS
# or
apt-get install libimobiledevice-utils  # Linux

# Test iproxy manually
iproxy 10001 9001 <UDID>

# Restart usbmuxd (macOS)
sudo launchctl stop com.apple.usbmuxd
sudo launchctl start com.apple.usbmuxd
```

**Common Causes:**
- Device not trusted (tap "Trust" on device)
- Screen locked (unlock device)
- USB cable/port issues
- Multiple devices - specify UDID in capabilities

---

### iOS Simulator: "Flutter server not found on any port"

**Cause:** Simulator shares host network, but server not running on expected ports.

**Solutions:**
1. **Verify app built with integration test SDK**
2. **Check simulator logs:**
```bash
xcrun simctl spawn booted log stream --predicate 'process == "Runner"'
```
3. **Use fixed port capability:**
```javascript
capabilities: {
  'appium:flutterSystemPort': 9001  // Use device port directly
}
```

---

## WebView / Context Issues

### "No WEBVIEW context found"

**Symptoms:** `getContexts()` returns only `NATIVE_APP`.

**Causes & Solutions:**

| Cause | Solution |
|-------|----------|
| WebView not loaded yet | Wait for web content to load |
| Android: Chromedriver version mismatch | `appium driver install chromedriver` |
| iOS: WebView not debuggable | Enable "Web Inspector" in Settings > Safari > Advanced |
| WebView in separate process | Some WebViews not inspectable |

**Debug:**
```javascript
// Poll for WebView context
const webviewContext = await browser.waitUntil(
  async () => {
    const contexts = await browser.getContexts();
    return contexts.find(ctx => ctx.includes('WEBVIEW'));
  },
  { timeout: 20000, timeoutMsg: 'WEBVIEW not found' }
);
await browser.switchContext(webviewContext);
```

---

### "Command not supported in WebView context"

**Error:** Native commands fail while in WebView context.

**Solution:** The driver automatically routes certain commands natively even in WebView:
```javascript
// These work in WebView context (no-proxy list):
await browser.getContexts();
await browser.getContext();
await browser.switchContext('NATIVE_APP');
await browser.getWindowHandle();
await browser.getPageSource();
await browser.getTitle();
await browser.url('https://example.com'); // Navigation
```

**For other commands, switch back to NATIVE_APP:**
```javascript
await browser.switchContext('NATIVE_APP');
await browser.flutterByText$('Flutter Button').click();
```

---

### Context Switch Hangs / Times Out

**Symptoms:** `switchContext` never returns.

**Solutions:**
1. **Increase timeout:** `newCommandTimeout: 240`
2. **Check Chromedriver/WDA logs**
3. **Restart Appium server between sessions**

---

## Image Mocking Issues

### "Permission denied" on Image Injection (Android)

**Error:**
```
Error: Permission denied writing image to device storage
```

**Solution:** Driver auto-requests permissions, but may need manual grant:
```bash
# Android 13+ (API 33+)
adb shell appops set <package> MANAGE_EXTERNAL_STORAGE allow

# Android 12- (API < 33)
adb shell pm grant <package> android.permission.WRITE_EXTERNAL_STORAGE
adb shell pm grant <package> android.permission.READ_EXTERNAL_STORAGE
```

**Capability to enable:**
```javascript
'appium:flutterEnableMockCamera': true
```

---

### "Image injection failed" / "Invalid base64"

**Causes:**
- File too large (>10MB)
- Invalid image format
- Base64 encoding error

**Solutions:**
```javascript
// Use WDIO service wrapper (handles encoding)
const imageId = await browser.flutterInjectImage('./test/photo.png');

// Manual: verify file exists and is valid
const fs = require('fs');
const base64 = fs.readFileSync('./test/photo.png', 'base64');
console.log('Base64 length:', base64.length);
```

**Supported formats:** PNG, JPEG, WebP (Flutter supported formats)

---

### "Activate injected image failed"

**Error:**
```
Error: Image with ID 'xxx' not found
```

**Cause:** Image ID expired or never injected.

**Solution:**
```javascript
// Inject returns imageId - use immediately
const imageId = await browser.flutterInjectImage('./test/photo.png');
// Use same session - don't restart driver
await browser.flutterActivateInjectedImage({ imageId });
```

---

## iOS-Specific Issues

### "XCUITest driver session creation failed"

**Common Causes:**

| Cause | Solution |
|-------|----------|
| WDA not installed | `appium driver install xcuitest` |
| Bundle ID mismatch | Verify `appium:bundleId` matches app |
| Device not trusted | Trust developer cert on device |
| Xcode version mismatch | Update Xcode + `xcode-select -s` |
| Simulator not booted | `xcrun simctl boot <UDID>` |

**Debug WDA:**
```bash
# Check WDA port
curl http://localhost:8100/status

# Manual WDA start
xcodebuild -project WebDriverAgent.xcodeproj \
  -scheme WebDriverAgentRunner \
  -destination 'id=<UDID>' test
```

---

### "WebDriverAgentRunner not installed"

**Solution:**
```bash
# Install via Appium
appium driver install xcuitest

# Or manual
cd /usr/local/lib/node_modules/appium/node_modules/appium-xcuitest-driver
./Scripts/bootstrap.sh -d
```

---

### iOS Real Device: "Could not proxy command to WebDriverAgent"

**Solutions:**
1. **Check iproxy tunnel:**
```bash
iproxy 8100 8100 <UDID>
# Test
curl http://localhost:8100/status
```

2. **Verify device logs:**
```bash
idevicesyslog | grep -i webdriver
```

3. **Restart device trust:**
- Settings > General > VPN & Device Management > Trust Developer App

---

## Android-Specific Issues

### "UiAutomator2 driver session creation failed"

**Common Causes:**

| Cause | Solution |
|-------|----------|
| UIAutomator2 not installed | `appium driver install uiautomator2` |
| App package/activity wrong | Verify `appPackage` / `appActivity` |
| ADB version mismatch | Update Android SDK Platform Tools |
| Emulator not ready | Wait for `sys.boot_completed=1` |

**Debug:**
```bash
# Check UIAutomator2 server
adb shell pm list packages -3 | grep uiautomator

# Manual install
adb install -r appium-uiautomator2-server.apk
adb install -r appium-uiautomator2-server-debug.apk
```

---

### "INSTRUMENTATION_FAILED" / "Permission denial"

**Solution:** Grant permissions:
```bash
adb shell pm grant <package> android.permission.ACCESS_FINE_LOCATION
adb shell pm grant <package> android.permission.READ_EXTERNAL_STORAGE
# etc.

# For test runner permissions
adb shell appops set <package> RUN_IN_BACKGROUND allow
```

---

### Chromedriver Version Mismatch (WebView)

**Error:**
```
Error: Chrome version 120.0.6099.129 not supported. 
Supported versions: 119.0.6045.105
```

**Solution:**
```bash
# Auto-download matching chromedriver
appium driver install chromedriver

# Or set specific version
appium driver install --source=chromedriver@120.0.6099.129
```

**Capability to allow auto-download:**
```javascript
'appium:allowInsecure': 'chromedriver_autodownload'
```

---

## macOS-Specific Issues

### "Mac2 driver session creation failed"

**Requirements:**
- macOS 13+ (Ventura)
- Xcode 15+
- App built for macOS (not iOS)

**Debug:**
```bash
# Check Mac2 driver
appium driver install mac2

# Verify app architecture
file /path/to/App.app/Contents/MacOS/App
# Should be: Mach-O 64-bit executable arm64 (or x86_64)
```

---

## Performance Issues

### Slow Test Execution

**Symptoms:** Tests take much longer than expected.

**Diagnosis:**
```javascript
// Add timing to test
const start = Date.now();
await browser.flutterByValueKey$('button').click();
console.log(`Click took ${Date.now() - start}ms`);
```

**Optimizations:**

| Issue | Solution |
|-------|----------|
| Port scanning on each session | Use `flutterSystemPort` fixed port |
| Repeated element finds | Cache elements, reuse locators |
| Large `pause()` calls | Replace with `flutterWaitForVisible` |
| Slow scroll | Increase `delta`, reduce `settleBetweenScrollsTimeout` |
| Network latency (remote device) | Run Appium on same network as device |

---

### Memory Leaks / Crashes

**Symptoms:** Appium process grows, eventually crashes.

**Solutions:**
1. **Restart Appium between test suites**
2. **Limit parallel sessions** - `maxInstances: 1`
3. **Update Node.js** - Memory fixes in newer versions
4. **Check for element cache buildup** - `deleteSession` clears cache

---

## CI/CD Issues

### "Tests pass locally but fail in CI"

**Common Causes:**

| Cause | Solution |
|-------|----------|
| Emulator not ready | Add health check before tests |
| Different Node/Appium versions | Pin versions in CI config |
| Missing dependencies | Install `libimobiledevice`, ADB in CI |
| Resource limits | Increase CI runner RAM/CPU |
| Timing differences | Increase timeouts in CI |

**Android Emulator in CI:**
```yaml
# GitHub Actions
- uses: reactivecircus/android-emulator-runner@v2
  with:
    api-level: 34
    force-avd-creation: true
    emulator-options: -no-window -gpu swiftshader_indirect -no-audio -no-boot-anim
    script: echo "Emulator ready"
```

**iOS Simulator in CI:**
```yaml
# Requires macOS runner
- name: Boot Simulator
  run: |
    xcrun simctl boot "iPhone 15"
    xcrun simctl list devices
```

---

### "Appium server not ready"

**Solution:** Wait for Appium before tests:
```bash
# Wait script
for i in {1..30}; do
  if curl -s http://localhost:4723/status > /dev/null; then
    echo "Appium ready"
    break
  fi
  sleep 2
done
```

---

## Error Code Reference

### Driver Errors

| Code | HTTP Status | Message | Resolution |
|------|-------------|---------|------------|
| `E_FLUTTER_SERVER_NOT_FOUND` | 500 | Flutter server not started | Verify app has integration test SDK |
| `E_VERSION_MISMATCH` | 500 | Server version incompatible | Update Flutter integration_test package |
| `E_PACKAGE_MISMATCH` | 500 | Wrong app connected | Verify bundleId/packageName |
| `E_PORT_FORWARD_FAILED` | 500 | Port forwarding failed | Check ADB/iproxy, kill conflicts |
| `E_SESSION_CREATE_FAILED` | 500 | Session creation failed | Check capabilities, app path |
| `E_NO_SUCH_ELEMENT` | 404 | Element not found | Verify locator, wait for element |
| `E_STALE_ELEMENT` | 400 | Element reference stale | Re-find element |
| `E_INVALID_SELECTOR` | 400 | Invalid locator syntax | Check JSON format for complex locators |
| `E_UNKNOWN_COMMAND` | 404 | Command not recognized | Check command spelling, driver version |

### Platform Driver Errors

| Source | Common Errors |
|--------|---------------|
| UIAutomator2 | `INSTRUMENTATION_FAILED`, `UiAutomator2 not installed` |
| XCUITest | `WDA not running`, `Device not trusted`, `Bundle ID mismatch` |
| Mac2 | `App not found`, `Architecture mismatch` |
| Chromedriver | `Version mismatch`, `Session not created` |

---

## Debugging Techniques

### 1. Enable Debug Logging

```bash
# Appium server
appium server --log-level debug --log-timestamp

# WDIO
logLevel: 'debug'  # in wdio.conf.js
```

### 2. Use Appium Inspector

```bash
# Start Appium with Inspector
appium server --allow-insecure=adb_shell
# Open http://localhost:4723/inspector
```

### 3. Capture Screenshots on Failure

```javascript
// wdio.conf.js
afterTest: async function(test, context, { error }) {
  if (error) {
    await browser.takeScreenshot();
  }
}
```

### 4. Inspect Flutter Render Tree

```javascript
// Get full render tree for debugging
const tree = await browser.executeScript('flutter: renderTree', [{}]);
console.log(JSON.stringify(tree, null, 2));

// Filter by widget type
const textFields = await browser.executeScript('flutter: renderTree', [{
  widgetType: 'TextField'
}]);
```

### 5. Check Flutter Server Status Directly

```bash
# Android (via ADB forward)
adb forward tcp:9001 tcp:9001
curl http://localhost:9001/status

# iOS Simulator
curl http://localhost:9001/status

# Expected response:
{
  "appInfo": { "packageName": "com.example.app" },
  "serverVersion": "0.0.18"
}
```

### 6. Monitor Device Logs

```bash
# Android
adb logcat -s FlutterIntegrationDriver

# iOS Simulator
xcrun simctl spawn booted log stream --predicate 'process == "Runner"'

# iOS Device
idevicesyslog | grep -i flutter
```

### 7. Network Traffic Capture

```bash
# Capture Appium ↔ Flutter server traffic
# Android: traffic goes through ADB forward
# Use mitmproxy or Wireshark on host port (10000-11000)
```

### 8. Test Isolation

```javascript
// Run single test with maximum debugging
npx wdio run wdio.conf.js --spec ./test/specs/test.e2e.js --mochaOpts.grep "login"
```

---

## Getting Help

### Before Reporting Issues

1. **Check existing issues:** [GitHub Issues](https://github.com/AppiumTestDistribution/appium-flutter-integration-driver/issues)
2. **Include in report:**
   - Driver version: `appium driver list | grep flutter`
   - Appium version: `appium --version`
   - Platform: Android/iOS/macOS, version
   - Device: Emulator/Simulator/Real device
   - Full error stack trace
   - Minimal reproduction case
   - Appium logs (debug level)
   - Flutter app `pubspec.yaml` (integration_test version)

### Community Resources

- **GitHub Issues:** Bug reports, feature requests
- **Appium Discuss:** [discuss.appium.io](https://discuss.appium.io/)
- **Discord/Slack:** Appium community channels

---

## Maintenance Checklist

### Regular Maintenance

- [ ] Update Appium & drivers monthly
- [ ] Update Flutter SDK & integration_test package
- [ ] Review and update CI/CD configurations
- [ ] Clean up old emulator/simulator images
- [ ] Rotate device certificates (iOS)

### Before Major Releases

- [ ] Test on all supported platforms
- [ ] Run full E2E test suite
- [ ] Verify WebView functionality
- [ ] Test image mocking
- [ ] Check port forwarding reliability
- [ ] Update documentation

---

*Troubleshooting Guide v2.0 | Last updated: 2025*