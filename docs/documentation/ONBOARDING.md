# Onboarding Guide

**Appium Flutter Integration Driver** - A guide for new team members, QA engineers, and stakeholders.

---

## What is This Project?

### In Simple Terms

This is a **test automation tool** that lets you write tests for Flutter mobile apps (Android, iOS, macOS) using the popular **Appium** framework.

Think of it as a **bridge** between:
- Your test scripts (written in JavaScript/TypeScript)
- The Flutter app running on a device/emulator
- The Appium automation server

### Why Does It Exist?

Flutter apps render their own UI (they don't use native Android/iOS UI components). Standard Appium drivers can't "see" inside Flutter apps. This driver solves that by talking to a **Flutter test server** embedded in your app.

---

## Key Concepts (Non-Technical)

| Term | Simple Explanation |
|------|-------------------|
| **Appium** | Industry-standard mobile automation framework |
| **Flutter** | Google's UI toolkit for building natively compiled apps |
| **Driver** | The "translator" between Appium and your specific app platform |
| **Locator** | How we find elements (buttons, text fields, etc.) in the app |
| **Session** | A single test run connection to a device |
| **Capabilities** | Settings that tell Appium what device/app to test |

---

## Who Uses This?

| Role | How They Use It |
|------|-----------------|
| **QA Engineers** | Write and run automated test scripts |
| **Developers** | Run tests locally during development |
| **CI/CD Pipelines** | Automated testing on every code change |
| **Release Managers** | Validate releases before deployment |

---

## Getting Started Checklist

### For Everyone

- [ ] Read this guide completely
- [ ] Understand the [Architecture Overview](ARCHITECTURE.md) (high-level)
- [ ] Know where to find [Troubleshooting](TROUBLESHOOTING.md)
- [ ] Join the team chat channel for this project

### For QA Engineers (Test Writers)

- [ ] Complete the [Quick Start Tutorial](#quick-start-tutorial)
- [ ] Learn [Locator Strategies](#finding-elements-in-flutter-apps)
- [ ] Practice with [Example Tests](../test/specs/test.e2e.js)
- [ ] Understand [WebView Testing](#testing-hybrid-apps-webview)

### For Developers (App Maintainers)

- [ ] Add [Flutter Test SDK](../README.md#flutter-app-setup) to your Flutter app
- [ ] Build testable app variants (debug/profile)
- [ ] Configure CI/CD with [example configs](../android.conf.ts)
- [ ] Set up [image mocking](../README.md#image-mocking) for camera features

### For DevOps/CI Engineers

- [ ] Set up Appium 3.x on build agents
- [ ] Configure device farms (emulators/simulators/real devices)
- [ ] Add driver installation to pipeline: `appium driver install flutter-integration`
- [ ] Review [System Requirements](ARCHITECTURE.md#system-requirements)

---

## Quick Start Tutorial

### Step 1: Prerequisites (5 minutes)

**You need:**
- Node.js 18+ installed
- Appium 3.x installed: `npm install -g appium`
- A Flutter app with integration test SDK (ask your Flutter developer)

### Step 2: Install the Driver (2 minutes)

```bash
# Option A: From npm (recommended)
appium driver install flutter-integration

# Option B: From source (for development)
git clone https://github.com/AppiumTestDistribution/appium-flutter-integration-driver.git
cd appium-flutter-integration-driver
npm install
npm run build
appium driver install --source=local .
```

### Step 3: Start Appium Server (1 minute)

```bash
appium server -ka 800 -pa /wd/hub --allow-insecure=adb_shell
```

You should see: `Appium REST http interface listener started on 127.0.0.1:4723`

### Step 4: Write Your First Test (10 minutes)

Create `test/login.test.js`:

```javascript
describe('Login Flow', () => {
  it('should login with valid credentials', async () => {
    // Find username field by its Flutter Key
    const usernameField = await browser.flutterByValueKey$('username_text_field');
    await usernameField.addValue('testuser');
    
    // Find password field
    const passwordField = await browser.flutterByValueKey$('password_text_field');
    await passwordField.addValue('password123');
    
    // Tap login button
    await browser.flutterByValueKey$('login_button').click();
    
    // Verify success - wait for welcome text
    await expect(browser.flutterByText$('Welcome, testuser!')).toBeDisplayed();
  });
});
```

### Step 5: Run the Test (2 minutes)

```bash
# Configure your app path
export APP_PATH=/path/to/your/app.apk

# Run with WebDriverIO
npx wdio run wdio.conf.js
```

---

## Finding Elements in Flutter Apps

### The Golden Rule: **Use ValueKeys**

Flutter developers should add `Key` widgets to important elements:

```dart
// Good - Easy to find in tests
TextField(key: Key('username_text_field'), ...)
ElevatedButton(key: Key('login_button'), ...)

// Harder to find - Avoid when possible
TextField(decoration: InputDecoration(labelText: 'Username'), ...)
```

### Locator Cheat Sheet

| What You're Looking For | Command | Example |
|------------------------|---------|---------|
| Button with Key `'submit'` | `flutterByValueKey$('submit')` | ✅ Best |
| Text "Welcome" | `flutterByText$('Welcome')` | ✅ Good |
| Input field labeled "Email" | `flutterBySemanticsLabel$('Email')` | ✅ Good |
| Any `TextField` widget | `flutterByType$('TextField')` | ⚠️ Generic |
| Item inside a list | `flutterByDescendant$({of: list, matching: item})` | 🔧 Advanced |
| Parent card of a button | `flutterByAncestor$({of: button, matching: card})` | 🔧 Advanced |

### Pro Tips

1. **Ask developers to add Keys** - It makes tests stable and fast
2. **Prefer `flutterByValueKey$`** - Most reliable, fastest
3. **Use semantics labels** - Also improves accessibility
4. **Avoid XPath/CSS** - They don't work well with Flutter's custom rendering

---

## Common Workflows

### Daily Development

```bash
# 1. Pull latest code
git pull

# 2. Install dependencies (if package.json changed)
npm install

# 3. Build driver (if source changed)
npm run build

# 4. Start Appium
appium server -ka 800 -pa /wd/hub --allow-insecure=adb_shell

# 5. Run tests
npm run wdio-android  # or wdio-ios
```

### Adding a New Test

1. **Identify the feature** - What user flow are you testing?
2. **Find the elements** - Use Appium Inspector or ask dev for Keys
3. **Write the test** - Follow existing patterns in `test/specs/`
4. **Run locally** - Verify it passes
5. **Commit** - Follow conventional commits: `test: add login flow test`

### Debugging a Failing Test

1. **Check Appium logs** - Look for error messages
2. **Add screenshots** - `await browser.takeScreenshot()`
3. **Use Appium Inspector** - Visual element inspection
4. **Check Flutter server** - Is the app built with integration test SDK?
5. **See [Troubleshooting](TROUBLESHOOTING.md)** - Common issues and fixes

---

## Testing Hybrid Apps (WebView)

Some Flutter apps embed web content (WebView). The driver handles this automatically:

```javascript
// 1. Switch to WebView context
const contexts = await browser.getContexts();
const webview = contexts.find(c => c.includes('WEBVIEW'));
await browser.switchContext(webview);

// 2. Use standard web commands
await browser.$('#web-button').click();
const title = await browser.getTitle();

// 3. Switch back to Flutter
await browser.switchContext('NATIVE_APP');
await browser.flutterByText$('Back to Flutter').click();
```

**No special setup needed** - The driver detects WebView contexts automatically.

---

## Image Mocking (Camera Testing)

Test camera/image picker features without a physical camera:

```javascript
it('should upload profile photo', async () => {
  // Inject a test image
  const imageId = await browser.flutterInjectImage('./test/photo.jpg');
  
  // Trigger image picker in app
  await browser.flutterByValueKey$('pick_photo_button').click();
  await browser.flutterByText$('SELECT FROM GALLERY').click();
  
  // Verify the mock image was "selected"
  await expect(browser.flutterByText$('Photo uploaded!')).toBeDisplayed();
  
  // Can switch between multiple injected images
  await browser.flutterInjectImage('./test/photo2.jpg');
  await browser.flutterActivateInjectedImage({ imageId: firstImageId });
});
```

**Use cases:** QR code scanning, profile photos, document uploads, barcode scanning

---

## CI/CD Integration

### GitHub Actions Example

```yaml
# .github/workflows/flutter-tests.yml
name: Flutter Integration Tests

on: [push, pull_request]

jobs:
  test-android:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20' }
      
      - name: Install Appium
        run: npm install -g appium
      
      - name: Install Driver
        run: appium driver install flutter-integration
      
      - name: Start Emulator
        uses: reactivecircus/android-emulator-runner@v2
        with:
          api-level: 34
          force-avd-creation: true
          emulator-options: -no-window -gpu swiftshader_indirect
      
      - name: Install App
        run: adb install app/build/outputs/apk/debug/app-debug.apk
      
      - name: Start Appium Server
        run: appium server -ka 800 -pa /wd/hub --allow-insecure=adb_shell &
      
      - name: Run Tests
        env:
          APP_PATH: /home/runner/work/.../app-debug.apk
        run: npx wdio run wdio.conf.js
```

### Key CI Considerations

| Concern | Solution |
|---------|----------|
| Emulator startup time | Use `reactivecircus/android-emulator-runner` |
| iOS testing | Requires macOS runners (GitHub macOS, MacStadium, etc.) |
| Parallel execution | Use `maxInstances: 1` per device, multiple devices |
| Flaky tests | Add retries, increase timeouts, use wait commands |
| Artifacts | Upload screenshots/videos on failure |

---

## Project Structure Overview

```
appium-flutter-integration-driver/
├── src/                          # Driver source code
│   ├── driver.ts                 # Main driver class
│   ├── session.ts                # Session management
│   ├── commands/element.ts       # Element interactions
│   ├── utils.ts                  # Core utilities
│   ├── android.ts / iOS.ts / macOS.ts  # Platform specifics
│   └── types.ts                  # TypeScript types
├── flutter-finder/
│   └── wdio-flutter-by-service/  # WDIO service (custom commands)
├── test/
│   ├── specs/                    # E2E test examples
│   └── unit/                     # Unit tests
├── docs/                         # Documentation
│   ├── API.md                    # Technical API reference
│   ├── ARCHITECTURE.md           # System architecture
│   ├── COMMANDS.md               # Command reference
│   ├── ONBOARDING.md             # This file
│   └── TROUBLESHOOTING.md        # Common issues
├── android.conf.ts / ios.conf.ts # WDIO configs
├── wdio.conf.ts                  # Base WDIO config
└── package.json                  # Project config
```

---

## Team Contacts

| Area | Contact | Channel |
|------|---------|---------|
| Driver Core | @driver-team | #flutter-driver-core |
| WDIO Service | @wdio-team | #flutter-wdio |
| CI/CD | @devops-team | #flutter-ci |
| Flutter App Integration | @flutter-team | #flutter-app |

---

## Learning Resources

### Essential Reading
- [README](../README.md) - Project overview and quick start
- [Commands Reference](COMMANDS.md) - All available commands
- [Troubleshooting](TROUBLESHOOTING.md) - Solutions to common problems

### External Resources
- [Appium Documentation](https://appium.io/docs/) - Core Appium concepts
- [WebDriverIO Guide](https://webdriver.io/docs/gettingstarted) - Test framework
- [Flutter Integration Testing](https://docs.flutter.dev/testing/integration-tests) - Flutter side setup
- [Appium Flutter Driver Issues](https://github.com/AppiumTestDistribution/appium-flutter-integration-driver/issues) - Known issues

### Video Tutorials
- [Appium 3.x with Flutter](https://youtube.com/...) - Coming soon
- [WDIO Service Deep Dive](https://youtube.com/...) - Coming soon

---

## Glossary

| Term | Definition |
|------|------------|
| **AUT** | Application Under Test |
| **E2E** | End-to-End (testing) |
| **WDIO** | WebDriverIO (test framework) |
| **UIA2** | UIAutomator2 (Android driver) |
| **WDA** | WebDriverAgent (iOS driver component) |
| **iproxy** | iOS port forwarding tool |
| **JWProxy** | JSON Wire Protocol Proxy |
| **Semver** | Semantic Versioning (MAJOR.MINOR.PATCH) |
| **Locator** | Strategy + value to find an element |
| **Session** | Active connection to a device for testing |

---

## FAQ

### Q: Do I need to modify my Flutter app?
**A:** Yes, you must add the `integration_test` package and call `integrationDriver()` in your test entry point. See [Flutter App Setup](../README.md#flutter-app-setup).

### Q: Can I test on real devices?
**A:** Yes! Android (USB/WiFi), iOS (requires Apple Developer account), and macOS all work on real devices.

### Q: Does this work with Flutter Web?
**A:** No, this is for mobile/desktop Flutter apps. For Flutter Web, use standard WebDriver/ChromeDriver.

### Q: How do I update the driver?
**A:** `appium driver uninstall flutter-integration && appium driver install flutter-integration`

### Q: My test is flaky. What should I do?
**A:** 
1. Use wait commands (`flutterWaitForVisible`)
2. Add explicit waits instead of `pause()`
3. Check for timing issues in CI
4. See [Troubleshooting](TROUBLESHOOTING.md#flaky-tests)

### Q: Can I run tests in parallel?
**A:** Yes, but each session needs its own device/emulator. Configure `maxInstances` in WDIO.

---

## Next Steps

1. **Complete the Quick Start Tutorial** above
2. **Read the [Commands Reference](COMMANDS.md)** to learn all available actions
3. **Explore [Example Tests](../test/specs/test.e2e.js)** for real-world patterns
4. **Bookmark [Troubleshooting](TROUBLESHOOTING.md)** for when things go wrong
5. **Ask questions** in the team channel - no question is too basic!

---

*Welcome to the team! 🎉 This guide is a living document - please suggest improvements.*