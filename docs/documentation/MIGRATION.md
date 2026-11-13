# Migration Guide: v1.x → v2.x

Guide for upgrading from Appium Flutter Integration Driver v1.x to v2.x.

---

## Breaking Changes Summary

| Area | v1.x | v2.x |
|------|------|------|
| **Appium Version** | 2.x | **3.x (required)** |
| **Node.js** | 16+ | **18+** |
| **Driver Installation** | `appium driver install` | Same command |
| **Locator Prefixes** | `key`, `text`, etc. | **`-flutter key`, `-flutter text`, etc.** |
| **WDIO Service** | Separate package | **Included in repo** |
| **TypeScript** | Partial | **Full strict mode** |

---

## Pre-Migration Checklist

- [ ] Upgrade Appium to 3.x: `npm install -g appium@latest`
- [ ] Upgrade Node.js to 18+ LTS
- [ ] Backup current test suite
- [ ] Review [CHANGELOG](../CHANGELOG.md) for all changes
- [ ] Update Flutter app `integration_test` package

---

## Step-by-Step Migration

### 1. Upgrade Appium

```bash
# Uninstall old driver
appium driver uninstall flutter-integration

# Install Appium 3.x
npm install -g appium@latest

# Verify
appium --version  # Should show 3.x.x

# Install new driver
appium driver install flutter-integration
```

### 2. Update Capabilities

**v1.x (Old):**
```javascript
capabilities: [{
  platformName: 'Android',
  automationName: 'Flutter',  // Old name
  app: '/path/to/app.apk',
}]
```

**v2.x (New):**
```javascript
capabilities: [{
  platformName: 'Android',
  automationName: 'FlutterIntegration',  // Required: exact name
  app: '/path/to/app.apk',
  // New optional capabilities:
  flutterServerLaunchTimeout: 30000,
  flutterSystemPort: 10001,  // Fixed port (optional)
}]
```

### 3. Update Locator Strategies

**v1.x (Old - no prefix):**
```javascript
// These still work for backward compatibility but deprecated
await browser.flutterByValueKey$('key');           // Uses 'key'
await browser.flutterByText$('text');              // Uses 'text'
await browser.flutterBySemanticsLabel$('label');   // Uses 'semantics label'
```

**v2.x (New - prefixed):**
```javascript
// Recommended: Use prefixed strategies
await browser.flutterByValueKey$('key');           // Uses '-flutter key'
await browser.flutterByText$('text');              // Uses '-flutter text'
await browser.flutterBySemanticsLabel$('label');   // Uses '-flutter semantics label'

// Raw executeScript (if not using WDIO service)
await driver.executeScript('flutter: findElement', [{
  using: '-flutter key',      // Required prefix
  value: 'my_key'
}]);
```

**Locator Mapping:**

| v1.x Strategy | v2.x Strategy | WDIO Command |
|---------------|---------------|--------------|
| `key` | `-flutter key` | `flutterByValueKey$` |
| `semantics label` | `-flutter semantics label` | `flutterBySemanticsLabel$` |
| `text` | `-flutter text` | `flutterByText$` |
| `type` | `-flutter type` | `flutterByType$` |
| `text containing` | `-flutter text containing` | `flutterByTextContaining$` |
| `descendant` | `-flutter descendant` | `flutterByDescendant$` |
| `ancestor` | `-flutter ancestor` | `flutterByAncestor$` |

---

### 4. Update WDIO Configuration

**v1.x (Old):**
```javascript
// wdio.conf.js
services: [
  'appium',
  // flutter-by service was separate
],
```

**v2.x (New):**
```javascript
// wdio.conf.js
const path = require('path');

services: [
  ['flutter-by', {}],  // Included in driver repo
  [
    'appium',
    {
      args: {
        basePath: '/wd/hub',
        port: 4723,
        log: path.join(process.cwd(), 'appium-logs', 'logs.txt'),
        allowInsecure: '*:chromedriver_autodownload,*:adb_shell',
      },
    },
  ],
],
```

**Note:** The `flutter-by` service is now part of the driver repository at `flutter-finder/wdio-flutter-by-service/`.

---

### 5. Update Test Code

#### Double Click / Long Press

**v1.x:**
```javascript
// Old: offset as separate parameter
await browser.executeScript('flutter: doubleClick', [element, { x: 10, y: 0 }]);
```

**v2.x:**
```javascript
// New: unified options object
await browser.flutterDoubleClick({
  element: element,
  offset: { x: 10, y: 0 }
});

// Or with locator
await browser.executeScript('flutter: doubleClick', [{
  locator: { using: '-flutter key', value: 'button' },
  offset: { x: 10, y: 0 }
}]);
```

#### Scroll Till Visible

**v1.x:**
```javascript
// Old: different parameter names
await browser.executeScript('flutter: scrollTillVisible', [{
  finder: element,
  scrollView: scrollElement,
  dx: 100,  // Old: dx/dy
  dy: 0,
  maxSwipes: 50,  // Old: maxSwipes
]);
```

**v2.x:**
```javascript
// New: unified parameters
await browser.flutterScrollTillVisible({
  finder: element,
  scrollView: scrollElement,
  scrollDirection: 'down',  // Explicit direction
  delta: 100,               // Scroll distance
  maxScrolls: 50,           // Renamed from maxSwipes
  settleBetweenScrollsTimeout: 200,
  dragDuration: 150,
});
```

#### Launch App

**v1.x:**
```javascript
// Old: mobile: launchApp
await browser.execute('mobile: launchApp', { 
  appId: 'com.example.app' 
});
```

**v2.x:**
```javascript
// New: flutter: launchApp (handles both platforms)
await browser.executeScript('flutter: launchApp', [{
  appId: 'com.example.app',
  arguments: ['--test-mode'],
  environment: { 'ENV': 'test' }
}]);
```

---

### 6. Update Flutter App

Ensure your Flutter app uses compatible `integration_test` version:

```yaml
# pubspec.yaml
dev_dependencies:
  integration_test:
    sdk: flutter
  flutter_test:
    sdk: flutter
```

Run:
```bash
flutter pub upgrade integration_test
flutter build apk --target=integration_test_driver.dart
```

**Minimum Flutter Server Version:** `0.0.18` (enforced by driver)

---

## Common Migration Issues

### Issue: "Locator not found" after migration

**Cause:** Using old strategy names without prefix.

**Fix:** Update all locators to use prefixed strategies or WDIO service commands.

```javascript
// WRONG (v1 style)
await browser.executeScript('flutter: findElement', [{
  using: 'key',  // Missing prefix
  value: 'button'
}]);

// CORRECT (v2 style)
await browser.executeScript('flutter: findElement', [{
  using: '-flutter key',  // With prefix
  value: 'button'
}]);

// BEST (WDIO service)
await browser.flutterByValueKey$('button');
```

---

### Issue: "Session creation failed" on iOS

**Cause:** Missing `flutterSystemPort` capability for iOS simulator parallel execution.

**Fix:** Add capability or ensure unique ports:

```javascript
capabilities: [{
  platformName: 'iOS',
  automationName: 'FlutterIntegration',
  udid: process.env.UDID,  // Required for real device
  flutterSystemPort: 9001,  // Fixed port for simulator
}]
```

---

### Issue: WebView context not working

**Cause:** v2.x handles WebView proxying differently.

**Fix:** Ensure `allowInsecure` includes `adb_shell` for Android:

```javascript
services: [
  [
    'appium',
    {
      args: {
        allowInsecure: '*:chromedriver_autodownload,*:adb_shell',
      },
    },
  ],
],
```

---

### Image Mocking Not Working

**Cause:** Android permission handling changed.

**Fix:** Ensure `flutterEnableMockCamera` capability:

```javascript
capabilities: [{
  // ... other caps
  'appium:flutterEnableMockCamera': true,
}]
```

---

## Automated Migration Script

Create a script to help migrate test files:

```javascript
// migrate-tests.js
const fs = require('fs');
const path = require('path');

const replacements = [
  // Locator strategies in executeScript
  [/using:\s*['"]key['"]/g, 'using: \'-flutter key\''],
  [/using:\s*['"]text['"]/g, 'using: \'-flutter text\''],
  [/using:\s*['"]semantics label['"]/g, 'using: \'-flutter semantics label\''],
  [/using:\s*['"]type['"]/g, 'using: \'-flutter type\''],
  [/using:\s*['"]text containing['"]/g, 'using: \'-flutter text containing\''],
  [/using:\s*['"]descendant['"]/g, 'using: \'-flutter descendant\''],
  [/using:\s*['"]ancestor['"]/g, 'using: \'-flutter ancestor\''],
  
  // Double click / long press
  [/flutter: doubleClick',\s*\[(\w+),\s*(\{.*?\})\]/g, 
   'flutter: doubleClick\', [{ origin: $1, offset: $2 }]'],
  [/flutter: longPress',\s*\[(\w+),\s*(\{.*?\})\]/g,
   'flutter: longPress\', [{ origin: $1, offset: $2 }]'],
  
  // Scroll parameters
  [/dx:/g, 'delta:'],
  [/dy:/g, ''], // Removed, use scrollDirection
  [/maxSwipes:/g, 'maxScrolls:'],
];

function migrateFile(filePath) {
  let content = fs.readFileSync(filePath, 'utf8');
  let changed = false;
  
  for (const [regex, replacement] of replacements) {
    const newContent = content.replace(regex, replacement);
    if (newContent !== content) {
      changed = true;
      content = newContent;
    }
  }
  
  if (changed) {
    fs.writeFileSync(filePath, content);
    console.log(`Migrated: ${filePath}`);
  }
}

function migrateDir(dir) {
  const files = fs.readdirSync(dir);
  for (const file of files) {
    const fullPath = path.join(dir, file);
    const stat = fs.statSync(fullPath);
    if (stat.isDirectory()) {
      migrateDir(fullPath);
    } else if (file.endsWith('.js') || file.endsWith('.ts')) {
      migrateFile(fullPath);
    }
  }
}

// Run
migrateDir('./test');
console.log('Migration complete!');
```

Run with:
```bash
node migrate-tests.js
```

---

## Verification Checklist

After migration, verify:

- [ ] All tests pass on Android
- [ ] All tests pass on iOS (simulator & device)
- [ ] All tests pass on macOS (if applicable)
- [ ] WebView context switching works
- [ ] Image mocking works
- [ ] Custom commands work
- [ ] CI/CD pipeline passes
- [ ] Performance is same or better

---

## Rollback Plan

If issues block migration:

```bash
# 1. Reinstall v1.x driver
appium driver uninstall flutter-integration
appium driver install flutter-integration@1.5.1

# 2. Use Appium 2.x
npm install -g appium@2

# 3. Revert capability changes
# automationName: 'Flutter' (not 'FlutterIntegration')

# 4. Keep v1.x locator strategies
```

---

## Support

- **Issues:** [GitHub Issues](https://github.com/AppiumTestDistribution/appium-flutter-integration-driver/issues)
- **Discussions:** [Appium Discuss](https://discuss.appium.io/)
- **CHANGELOG:** [Full history](../CHANGELOG.md)

---

*Migration Guide v2.0 | For v1.x → v2.x upgrades*