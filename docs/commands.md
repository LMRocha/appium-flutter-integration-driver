# Commands Reference

Complete reference for all Flutter Integration Driver commands with examples.

---

## Table of Contents

- [Locator Commands](#locator-commands)
- [Gesture Commands](#gesture-commands)
- [Wait Commands](#wait-commands)
- [App Management Commands](#app-management-commands)
- [Image Mocking Commands](#image-mocking-commands)
- [Debug Commands](#debug-commands)
- [WebDriver Standard Commands](#webdriver-standard-commands)
- [Usage Patterns](#usage-patterns)

---

## Locator Commands

All locator commands are available via the **WDIO Service** (`wdio-flutter-by-service`).

### Basic Locators

| Command | Returns | Description |
|---------|---------|-------------|
| `flutterByValueKey(value)` | `Locator` | Get locator for ValueKey |
| `flutterByValueKey$(value)` | `Element` | Find single element by ValueKey |
| `flutterByValueKey$$(value)` | `Element[]` | Find multiple elements by ValueKey |
| `flutterBySemanticsLabel(label)` | `Locator` | Get locator for Semantics Label |
| `flutterBySemanticsLabel$(label)` | `Element` | Find by Semantics Label |
| `flutterBySemanticsLabel$$(label)` | `Element[]` | Find multiple by Semantics Label |
| `flutterByText(text)` | `Locator` | Get locator for exact Text |
| `flutterByText$(text)` | `Element` | Find by exact Text |
| `flutterByText$$(text)` | `Element[]` | Find multiple by Text |
| `flutterByType(type)` | `Locator` | Get locator for Widget Type |
| `flutterByType$(type)` | `Element` | Find by Widget Type |
| `flutterByType$$(type)` | `Element[]` | Find multiple by Widget Type |
| `flutterByTextContaining(text)` | `Locator` | Get locator for partial Text |
| `flutterByTextContaining$(text)` | `Element` | Find by partial Text |
| `flutterByTextContaining$$(text)` | `Element[]` | Find multiple by partial Text |
| `flutterByToolTip(tooltip)` | `Locator` | Get locator for Tooltip |
| `flutterByToolTip$(tooltip)` | `Element` | Find by Tooltip |
| `flutterByToolTip$$(tooltip)` | `Element[]` | Find multiple by Tooltip |

### Advanced Locators

| Command | Parameters | Returns |
|---------|------------|---------|
| `flutterByDescendant({of, matching})` | `of: Element, matching: Element` | `Locator` |
| `flutterByDescendant$({of, matching})` | `of: Element, matching: Element` | `Element` |
| `flutterByDescendant$$({of, matching})` | `of: Element, matching: Element` | `Element[]` |
| `flutterByAncestor({of, matching})` | `of: Element, matching: Element` | `Locator` |
| `flutterByAncestor$({of, matching})` | `of: Element, matching: Element` | `Element` |
| `flutterByAncestor$$({of, matching})` | `of: Element, matching: Element` | `Element[]` |

---

### Examples: Basic Locators

```javascript
// Find by ValueKey (RECOMMENDED - fastest, most stable)
const loginBtn = await browser.flutterByValueKey$('login_button');
await loginBtn.click();

// Find by Semantics Label (accessibility)
const usernameField = await browser.flutterBySemanticsLabel$('Username');
await usernameField.addValue('testuser');

// Find by exact Text
const welcomeText = await browser.flutterByText$('Welcome Back!');
await expect(welcomeText).toBeDisplayed();

// Find by Widget Type
const textFields = await browser.flutterByType$$('TextField');
console.log(`Found ${textFields.length} text fields`);

// Find by partial Text (contains)
const errorMsg = await browser.flutterByTextContaining$('error');
await expect(errorMsg).toBeDisplayed();

// Find by Tooltip
const helpBtn = await browser.flutterByToolTip$('Help');
await helpBtn.click();
```

---

### Examples: Advanced Locators

```javascript
// DESCENDANT: Find child within a parent
// Use case: List items, nested widgets, cards with multiple children

const parentCard = await browser.flutterByValueKey('product_card_123');
const priceText = await browser.flutterByDescendant$({
  of: parentCard,
  matching: await browser.flutterByText('$'),
});
await expect(priceText).toBeDisplayed();

// Multiple descendants
const allPrices = await browser.flutterByDescendant$$({
  of: await browser.flutterByValueKey('product_list'),
  matching: await browser.flutterByType('Text'),
});


// ANCESTOR: Find parent containing a child
// Use case: Tap card when you only know the button inside

const buyButton = await browser.flutterByValueKey$('buy_button');
const productCard = await browser.flutterByAncestor$({
  of: buyButton,
  matching: await browser.flutterByValueKey('product_card'),
});
await productCard.click(); // Tap the whole card


// CHAINED: Complex queries
const nestedItem = await browser.flutterByDescendant$({
  of: await browser.flutterByAncestor$({
    of: await browser.flutterByText('Target'),
    matching: await browser.flutterByValueKey('section'),
  }),
  matching: await browser.flutterByType('Icon'),
});
```

---

### Raw executeScript Usage (Without WDIO Service)

```javascript
// Direct driver command (returns Locator object)
const locator = await browser.executeScript('flutter: findElement', [{
  using: '-flutter key',
  value: 'my_button'
}]);

// Find element using locator
const element = await browser.executeScript('flutter: findElement', [{
  using: '-flutter descendant',
  selector: JSON.stringify({
    of: { using: '-flutter key', value: 'parent' },
    matching: { using: '-flutter text', value: 'Child' }
  })
}]);
```

---

## Gesture Commands

### `flutter: doubleClick` / `flutterDoubleClick`

**Double-tap an element.**

```javascript
// WDIO Service (recommended)
await browser.flutterDoubleClick({
  element: await browser.flutterByValueKey$('image'),
  offset: { x: 10, y: -10 }  // Optional: offset from center
});

// Raw executeScript
await browser.executeScript('flutter: doubleClick', [{
  origin: element,           // WebElement
  offset: { x: 0, y: 0 },    // Optional
  locator: {                 // Alternative to origin
    using: '-flutter key',
    value: 'image'
  }
}]);
```

**Parameters:**
| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `origin` | WebElement | No* | Element to double-click |
| `offset` | `{x,y}` | No | Offset from element center |
| `locator` | Locator | No* | Alternative to origin |

*Either `origin` or `locator` required.

---

### `flutter: longPress` / `flutterLongPress`

**Long press an element.**

```javascript
// WDIO Service
await browser.flutterLongPress({
  element: await browser.flutterByValueKey$('delete_button'),
  offset: { x: 5, y: 5 }
});

// Raw
await browser.executeScript('flutter: longPress', [{
  origin: element,
  offset: { x: 0, y: 0 },
  locator: { using: '-flutter key', value: 'delete_button' }
}]);
```

---

### `flutter: dragAndDrop` / `flutterDragAndDrop`

**Drag from source to target.**

```javascript
// WDIO Service
const source = await browser.flutterByValueKey$('draggable_item');
const target = await browser.flutterByValueKey$('drop_zone');
await browser.flutterDragAndDrop({ source, target });

// Raw
await browser.executeScript('flutter: dragAndDrop', [{
  source: sourceElement,
  target: targetElement
}]);
```

**Parameters:**
| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `source` | WebElement | Yes | Element to drag |
| `target` | WebElement | Yes | Drop target element |

---

### `flutter: scrollTillVisible` / `flutterScrollTillVisible`

**Scroll until element becomes visible.**

```javascript
// WDIO Service - Basic
const item = await browser.flutterScrollTillVisible({
  finder: await browser.flutterByText('Item 50'),
  scrollDirection: 'down',
});

// WDIO Service - Advanced
const item = await browser.flutterScrollTillVisible({
  finder: await browser.flutterByDescendant({
    of: await browser.flutterByValueKey('list_view'),
    matching: await browser.flutterByText('Item 100'),
  }),
  scrollView: await browser.flutterByValueKey('scroll_view'),  // Specific scroller
  scrollDirection: 'down',
  delta: 200,                    // Scroll distance per gesture
  maxScrolls: 100,               // Max attempts
  settleBetweenScrollsTimeout: 300,  // Wait between scrolls (ms)
  dragDuration: 200,             // Drag animation duration (ms)
});

// Raw
await browser.executeScript('flutter: scrollTillVisible', [{
  finder: { using: '-flutter text', value: 'Item 50' },
  scrollView: scrollElement,
  delta: 100,
  maxScrolls: 50,
  settleBetweenScrollsTimeout: 200,
  dragDuration: 150,
  scrollDirection: 'down'
}]);
```

**Parameters:**
| Param | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `finder` | Element\|Locator | Yes | - | Element to find |
| `scrollView` | WebElement | No | Auto | Scrollable container |
| `scrollDirection` | string | No | `'down'` | `'up'`, `'down'`, `'left'`, `'right'` |
| `delta` | number | No | `100` | Scroll distance |
| `maxScrolls` | number | No | `50` | Max scroll attempts |
| `settleBetweenScrollsTimeout` | number | No | `200` | Wait between scrolls (ms) |
| `dragDuration` | number | No | `150` | Drag animation (ms) |

**Returns:** Found element or `null`

---

## Wait Commands

### `flutter: waitForVisible` / `flutterWaitForVisible`

**Wait for element to appear.**

```javascript
// WDIO Service
await browser.flutterWaitForVisible({
  element: await browser.flutterByValueKey$('loading_spinner'),
  timeout: 10000  // ms
});

// Or with locator (re-finds each check)
await browser.flutterWaitForVisible({
  locator: await browser.flutterByValueKey('success_message'),
  timeout: 15000
});

// Raw
await browser.executeScript('flutter: waitForVisible', [{
  element: element,
  timeout: 10000
}]);
```

**Parameters:**
| Param | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `element` | WebElement | No* | - | Element to wait for |
| `locator` | Locator | No* | - | Locator to re-find |
| `timeout` | number | No | `5000` | Max wait (ms) |

*Either `element` or `locator` required.

---

### `flutter: waitForAbsent` / `flutterWaitForAbsent`

**Wait for element to disappear.**

```javascript
// WDIO Service
await browser.flutterWaitForAbsent({
  element: await browser.flutterByValueKey$('loading_spinner'),
  timeout: 10000
});

// With locator
await browser.flutterWaitForAbsent({
  locator: await browser.flutterByText('Loading...'),
  timeout: 15000
});

// Raw
await browser.executeScript('flutter: waitForAbsent', [{
  locator: { using: '-flutter text', value: 'Loading...' },
  timeout: 10000
}]);
```

---

## App Management Commands

### `flutter: launchApp` / `flutterLaunchApp`

**Launch or relaunch the application.**

```javascript
// WDIO Service (via executeScript)
await browser.executeScript('flutter: launchApp', [{
  appId: 'com.example.myapp',           // Required: package name (Android) or bundle ID (iOS)
  arguments: ['--disable-animations'],  // Optional: launch args
  environment: { 'TEST_MODE': 'true' }  // Optional: env vars
}]);

// iOS: Automatically adds --flutter-server-port argument
// Android: Uses mobile: activateApp
```

**Parameters:**
| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `appId` | string | Yes | Package name or bundle ID |
| `arguments` | string[] | No | Launch arguments |
| `environment` | Object | No | Environment variables |

**Use Cases:**
```javascript
// Restart app between tests
await browser.executeScript('flutter: launchApp', [{
  appId: 'com.example.app',
  arguments: ['--clear-cache', '--test-mode']
}]);

// Launch with specific config
await browser.executeScript('flutter: launchApp', [{
  appId: 'com.example.app',
  environment: { 'API_URL': 'https://staging.api.example.com' }
}]);
```

---

## Image Mocking Commands

### `flutter: injectImage` / `flutterInjectImage`

**Inject base64 image for camera/image picker mocking.**

```javascript
// WDIO Service (handles file → base64)
const imageId = await browser.flutterInjectImage('./test/qr_code.png');

// Raw (must provide base64)
const fs = require('fs');
const base64 = fs.readFileSync('./test/qr_code.png', 'base64');
const imageId = await browser.executeScript('flutter: injectImage', [{
  base64Image: base64
}]);
```

**Returns:** `imageId` string for later activation.

**Android Permissions:** Auto-requested based on API level:
- API < 33: `WRITE_EXTERNAL_STORAGE`, `READ_EXTERNAL_STORAGE`
- API ≥ 33: `MANAGE_EXTERNAL_STORAGE`

---

### `flutter: activateInjectedImage` / `flutterActivateInjectedImage`

**Activate previously injected image.**

```javascript
// WDIO Service
await browser.flutterActivateInjectedImage({ imageId: firstImageId });

// Raw
await browser.executeScript('flutter: activateInjectedImage', [{
  imageId: 'injected_image_123'
}]);
```

**Complete Image Mocking Flow:**

```javascript
it('should test image picker with multiple images', async () => {
  // Inject first image
  const firstImageId = await browser.flutterInjectImage('./test/photo1.jpg');
  
  // Trigger picker in app
  await browser.flutterByValueKey$('pick_image').click();
  await browser.flutterByText$('SELECT FROM GALLERY').click();
  
  // Verify first image
  await expect(browser.flutterByText$('Photo 1 selected')).toBeDisplayed();
  
  // Inject second image
  await browser.flutterInjectImage('./test/photo2.jpg');
  
  // Trigger picker again
  await browser.flutterByValueKey$('pick_image').click();
  await browser.flutterByText$('SELECT FROM GALLERY').click();
  
  // Verify second image
  await expect(browser.flutterByText$('Photo 2 selected')).toBeDisplayed();
  
  // Reactivate first image
  await browser.flutterActivateInjectedImage({ imageId: firstImageId });
  await browser.flutterByValueKey$('pick_image').click();
  await browser.flutterByText$('SELECT FROM GALLERY').click();
  
  // Verify first image again
  await expect(browser.flutterByText$('Photo 1 selected')).toBeDisplayed();
});
```

---

## Debug Commands

### `flutter: renderTree` / `flutterRenderTree`

**Get Flutter render tree for debugging.**

```javascript
// Get full tree
const tree = await browser.executeScript('flutter: renderTree', [{}]);
console.log(JSON.stringify(tree, null, 2));

// Filter by widget type
const buttons = await browser.executeScript('flutter: renderTree', [{
  widgetType: 'ElevatedButton'
}]);

// Filter by text content
const errorTexts = await browser.executeScript('flutter: renderTree', [{
  text: 'error'
}]);

// Filter by key
const specificWidget = await browser.executeScript('flutter: renderTree', [{
  key: 'submit_button'
}]);
```

**Parameters:**
| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `widgetType` | string | No | Filter by widget class name |
| `text` | string | No | Filter by text content |
| `key` | string | No | Filter by widget Key |

**Response Format:**
```json
[
  {
    "type": "ElevatedButton",
    "key": "submit_button",
    "text": "Submit",
    "rect": { "x": 100, "y": 200, "width": 120, "height": 48 },
    "children": []
  }
]
```

---

## WebDriver Standard Commands

These standard WebDriver commands work with Flutter elements when using Flutter locators:

| Command | Flutter Support | Notes |
|---------|----------------|-------|
| `click()` | ✅ Full | Routes to Flutter server |
| `getText()` | ✅ Full | Gets element text |
| `getAttribute(name)` | ✅ Full | `'displayed'`, `'enabled'`, `'flags'`, `'all'`, etc. |
| `getElementRect()` | ✅ Full | Position & size |
| `isDisplayed()` | ✅ Full | Via `displayed` attribute |
| `isEnabled()` | ✅ Full | Via `enabled` attribute |
| `addValue(text)` | ✅ Full | Text input |
| `clearValue()` | ✅ Full | Clear text field |
| `setValue(text)` | ✅ Full | Set text value |
| `getSize()` | ✅ Full | Width/height |
| `getLocation()` | ✅ Full | X/Y position |

### Special Attributes (Flutter-Specific)

```javascript
const element = await browser.flutterByValueKey$('button');

// Get all Flutter attributes
const allAttrs = await element.getAttribute('all');
// Returns: owner, rect, tags, actions, flags, label, value, 
//          hint, tooltip, textDirection, sortKey, elevation, etc.

// Get semantic flags
const flags = await element.getAttribute('flags');
// Returns: '[isTextField, hasEnabledState, isReadOnly]'

// Get actions
const actions = await element.getAttribute('actions');
// Returns: '[tap, longPress, scrollLeft, scrollRight]'

// Get rect
const rect = await element.getElementRect();
// Returns: { x, y, width, height }
```

---

## Usage Patterns

### Pattern 1: Page Object Model

```javascript
// pageobjects/LoginPage.js
class LoginPage {
  get usernameField() { return browser.flutterByValueKey$('username_field'); }
  get passwordField() { return browser.flutterByValueKey$('password_field'); }
  get loginButton() { return browser.flutterByValueKey$('login_button'); }
  get errorMessage() { return browser.flutterByTextContaining$('error'); }

  async login(username, password) {
    await this.usernameField.addValue(username);
    await this.passwordField.addValue(password);
    await this.loginButton.click();
  }

  async waitForError(timeout = 5000) {
    await browser.flutterWaitForVisible({
      locator: await this.errorMessage,
      timeout
    });
  }
}

module.exports = new LoginPage();
```

```javascript
// test/login.test.js
const LoginPage = require('../pageobjects/LoginPage');

describe('Login', () => {
  it('should show error for invalid credentials', async () => {
    await LoginPage.login('wrong', 'credentials');
    await LoginPage.waitForError();
    await expect(LoginPage.errorMessage).toBeDisplayed();
  });
});
```

---

### Pattern 2: Reusable Helpers

```javascript
// helpers/flutter.js
async function scrollToText(text, options = {}) {
  return browser.flutterScrollTillVisible({
    finder: await browser.flutterByText(text),
    scrollDirection: 'down',
    delta: 200,
    maxScrolls: 50,
    ...options
  });
}

async function waitAndClick(locator, timeout = 5000) {
  await browser.flutterWaitForVisible({ locator, timeout });
  const el = await locator();
  await el.click();
  return el;
}

async function fillForm(fields) {
  for (const [key, value] of Object.entries(fields)) {
    const field = await browser.flutterByValueKey$(key);
    await field.clearValue();
    await field.addValue(value);
  }
}

module.exports = { scrollToText, waitAndClick, fillForm };
```

```javascript
// test/form.test.js
const { scrollToText, fillForm } = require('../helpers/flutter');

it('should complete long form', async () => {
  await fillForm({
    first_name: 'John',
    last_name: 'Doe',
    email: 'john@example.com',
    phone: '555-1234',
  });
  
  await scrollToText('Submit');
  await browser.flutterByValueKey$('submit').click();
});
```

---

### Pattern 3: Cross-Platform Handling

```javascript
// helpers/platform.js
const isIOS = () => browser.capabilities.platformName?.toLowerCase() === 'ios';
const isAndroid = () => browser.capabilities.platformName?.toLowerCase() === 'android';

async function launchApp(appId, options = {}) {
  if (isIOS()) {
    // iOS needs bundle ID and port argument
    await browser.executeScript('flutter: launchApp', [{
      appId,
      arguments: options.arguments,
      environment: options.environment
    }]);
  } else {
    // Android uses activateApp
    await browser.executeScript('flutter: launchApp', [{
      appId,
      arguments: options.arguments
    }]);
  }
}

async function installApp(appPath) {
  if (isAndroid()) {
    await browser.installApp(appPath);
  } else {
    // iOS: app installed via capabilities
    console.log('iOS app pre-installed via capabilities');
  }
}
```

---

### Pattern 4: Robust Element Interaction

```javascript
// helpers/robust.js
async function robustClick(locatorFn, options = {}) {
  const { 
    timeout = 10000, 
    retries = 3, 
    waitAfterClick = 500 
  } = options;

  for (let attempt = 1; attempt <= retries; attempt++) {
    try {
      await browser.flutterWaitForVisible({ 
        locator: await locatorFn(), 
        timeout 
      });
      const el = await locatorFn();
      await el.click();
      await browser.pause(waitAfterClick);
      return el;
    } catch (err) {
      if (attempt === retries) throw err;
      await browser.pause(1000 * attempt); // Exponential backoff
    }
  }
}

async function robustInput(locatorFn, text, options = {}) {
  const { clearFirst = true, timeout = 5000 } = options;
  
  await browser.flutterWaitForVisible({ 
    locator: await locatorFn(), 
    timeout 
  });
  
  const el = await locatorFn();
  if (clearFirst) await el.clearValue();
  await el.addValue(text);
  
  // Verify input
  const value = await el.getAttribute('value');
  if (value !== text) {
    throw new Error(`Input verification failed: expected "${text}", got "${value}"`);
  }
  return el;
}
```

---

### Pattern 5: Test Data Management

```javascript
// fixtures/testData.js
const testUsers = {
  valid: { username: 'testuser', password: 'Pass123!' },
  invalid: { username: 'wrong', password: 'wrong' },
  locked: { username: 'lockeduser', password: 'Pass123!' },
};

const testProducts = [
  { id: 'prod_1', name: 'Widget A', price: '$19.99' },
  { id: 'prod_2', name: 'Gadget B', price: '$29.99' },
];

module.exports = { testUsers, testProducts };
```

```javascript
// test/purchase.test.js
const { testUsers, testProducts } = require('../fixtures/testData');

describe('Purchase Flow', () => {
  beforeEach(async () => {
    await LoginPage.login(testUsers.valid.username, testUsers.valid.password);
  });

  it('should purchase product', async () => {
    const product = testProducts[0];
    await scrollToText(product.name);
    await robustClick(() => browser.flutterByValueKey$(product.id));
    await robustClick(() => browser.flutterByValueKey$('add_to_cart'));
    await robustClick(() => browser.flutterByValueKey$('checkout'));
    await robustClick(() => browser.flutterByValueKey$('confirm_purchase'));
    await expect(browser.flutterByText$('Order Confirmed')).toBeDisplayed();
  });
});
```

---

## Command Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        QUICK REFERENCE CARD                              │
├─────────────────────────────────────────────────────────────────────────┤
│ LOCATORS                                                                 │
│   $key = flutterByValueKey$(key)        // Best: stable, fast          │
│   $label = flutterBySemanticsLabel$(lbl) // Good: accessibility        │
│   $text = flutterByText$(text)           // Good: visible text         │
│   $type = flutterByType$(type)           // Generic: widget class      │
│   $desc = flutterByDescendant$({of, matching}) // Nested               │
│   $anc = flutterByAncestor$({of, matching})    // Parent               │
├─────────────────────────────────────────────────────────────────────────┤
│ GESTURES                                                                 │
│   await flutterDoubleClick({element, offset})                          │
│   await flutterLongPress({element, offset})                            │
│   await flutterDragAndDrop({source, target})                           │
│   await flutterScrollTillVisible({finder, scrollDirection: 'down'})    │
├─────────────────────────────────────────────────────────────────────────┤
│ WAITS                                                                    │
│   await flutterWaitForVisible({element|locator, timeout})              │
│   await flutterWaitForAbsent({element|locator, timeout})               │
├─────────────────────────────────────────────────────────────────────────┤
│ APP                                                                      │
│   await executeScript('flutter: launchApp', [{appId, args, env}])      │
├─────────────────────────────────────────────────────────────────────────┤
│ IMAGES                                                                   │
│   const id = await flutterInjectImage('./file.png')                    │
│   await flutterActivateInjectedImage({imageId: id})                    │
├─────────────────────────────────────────────────────────────────────────┤
│ DEBUG                                                                    │
│   const tree = await executeScript('flutter: renderTree', [{}])        │
└─────────────────────────────────────────────────────────────────────────┘
```

---

*Commands Reference v2.0 | Last updated: 2025*