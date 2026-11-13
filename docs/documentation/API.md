# API Reference

Complete API documentation for the Appium Flutter Integration Driver.

## Table of Contents

- [Driver Class](#driver-class)
- [Session Management](#session-management)
- [Element Interaction](#element-interaction)
- [Flutter Custom Commands](#flutter-custom-commands)
- [Platform-Specific APIs](#platform-specific-apis)
- [Error Codes](#error-codes)
- [Type Definitions](#type-definitions)

---

## Driver Class

### `AppiumFlutterDriver`

Main driver class extending `@appium/base-driver`.

#### Constructor

```typescript
constructor(args: any, shouldValidateCaps: boolean)
```

**Parameters:**
- `args` - Appium driver arguments
- `shouldValidateCaps` - Whether to validate capabilities

#### Properties

| Property | Type | Description |
|----------|------|-------------|
| `proxydriver` | `XCUITestDriver \| AndroidUiautomator2Driver \| Mac2Driver` | Underlying platform driver |
| `flutterPort` | `number \| null` | Port where Flutter test server is running |
| `proxy` | `JWProxy \| undefined` | Proxy to Flutter test server |
| `currentContext` | `string` | Current context (`NATIVE_APP` or `WEBVIEW_*`) |
| `NATIVE_CONTEXT_NAME` | `string` | Constant `'NATIVE_APP'` |
| `locatorStrategies` | `string[]` | Supported locator strategies |

#### Locator Strategies

The driver supports these strategies (in order of precedence):

```typescript
[
  'xpath',
  'css selector',
  'id',
  'name',
  'class name',
  '-android uiautomator',
  'accessibility id',
  '-ios predicate string',
  '-ios class chain',
  ...FLUTTER_LOCATORS,           // Backward compatibility
  ...FLUTTER_LOCATORS.map(l => `-flutter ${l}`),  // Prefixed strategies
  '-flutter descendant',
  '-flutter ancestor',
]
```

Where `FLUTTER_LOCATORS = ['key', 'semantics label', 'text', 'type', 'text containing', 'descendant', 'ancestor']`

---

## Session Management

### `createSession`

Creates a new session with the Flutter driver.

```typescript
async createSession(...args): Promise<DefaultCreateSessionResult>
```

**Process:**
1. Validates capabilities
2. Creates platform-specific proxy driver (UIA2, XCUITest, or Mac2)
3. Attaches launch arguments for iOS simulator
4. Establishes port forwarding
5. Discovers Flutter test server port
6. Creates proxy connection to Flutter server
7. Validates server version compatibility

**Capabilities:**

| Capability | Type | Required | Description |
|------------|------|----------|-------------|
| `platformName` | string | Yes | `iOS`, `Android`, or `Mac` |
| `automationName` | string | Yes | Must be `FlutterIntegration` |
| `app` | string | Yes | Path to application |
| `udid` | string | iOS/Device | Device unique identifier |
| `flutterServerLaunchTimeout` | number | No | Server startup timeout (ms, default: 5000) |
| `flutterSystemPort` | number | No | Fixed system port for forwarding |
| `address` | string | No | Server bind address (default: 127.0.0.1) |
| `avd` | string | Android | AVD name to launch |

**Returns:** `[sessionId, capabilities]`

**Throws:**
- `Error` - If Flutter server not found or version incompatible
- `Error` - If platform not supported

---

### `deleteSession`

Cleans up session resources.

```typescript
async deleteSession(): Promise<void>
```

**Process:**
1. Removes Android port forwarding (if applicable)
2. Deletes proxy driver session
3. Calls parent deleteSession

---

## Element Interaction

The driver implements standard WebDriver element commands, routed to either the Flutter server or platform driver based on locator strategy.

### `findElOrEls`

```typescript
async findElOrEls(
  strategy: string,
  selector: string,
  mult: boolean,
  context: string
): Promise<any>
```

**Parameters:**
- `strategy` - Locator strategy (e.g., `-flutter key`, `xpath`)
- `selector` - Selector value
- `mult` - `true` for findElements, `false` for findElement
- `context` - Element ID for relative search, or empty for root

**Returns:** Element object(s) with `ELEMENT` or `element-6066-11e4-a52e-4f735466cecf` key

**Routing Logic:**
- Flutter strategies (`-flutter *`, `key`, `semantics label`, etc.) → Flutter server
- Native strategies → Platform driver (UIA2, XCUITest, Mac2)

---

### Standard Element Commands

| Command | Flutter Server Endpoint | Description |
|---------|------------------------|-------------|
| `click` | `POST /element/{id}/click` | Tap element |
| `getText` | `GET /element/{id}/text` | Get element text |
| `getAttribute` | `GET /element/{id}/attribute/{name}` | Get attribute |
| `getElementRect` | `GET /element/{id}/rect` | Get position/size |
| `elementDisplayed` | `GET /element/{id}/attribute/displayed` | Check visibility |
| `elementEnabled` | `GET /element/{id}/attribute/enabled` | Check enabled state |
| `setValue` | `POST /element/{id}/value` | Input text |
| `clear` | `POST /element/{id}/clear` | Clear text field |

**Special Handling:**
- Mac2Driver uses non-blocking click for Flutter elements
- Element cache maps element IDs to their proxy driver

---

## Flutter Custom Commands

All custom commands are executed via `executeScript('flutter:commandName', [params])`.

### Command Map

```typescript
static executeMethodMap = {
  'flutter: doubleClick': { ... },
  'flutter: waitForVisible': { ... },
  'flutter: waitForAbsent': { ... },
  'flutter: scrollTillVisible': { ... },
  'flutter: longPress': { ... },
  'flutter: dragAndDrop': { ... },
  'flutter: launchApp': { ... },
  'flutter: injectImage': { ... },
  'flutter: activateInjectedImage': { ... },
  'flutter: renderTree': { ... },
}
```

---

### Gesture Commands

#### `flutter: doubleClick`

Double-tap an element.

```typescript
await driver.executeScript('flutter: doubleClick', [{
  origin?: WebElement,      // Element to double-click
  offset?: { x: number; y: number },  // Offset from element center
  locator?: Locator,        // Alternative: locate by strategy
}]);
```

**Endpoint:** `POST /session/{sessionId}/appium/gestures/double_click`

**Example:**
```javascript
// By element
await browser.executeScript('flutter: doubleClick', [{
  origin: await browser.flutterByValueKey$('button'),
}]);

// With offset
await browser.executeScript('flutter: doubleClick', [{
  origin: element,
  offset: { x: 10, y: 0 },
}]);
```

---

#### `flutter: longPress`

Long press an element.

```typescript
await driver.executeScript('flutter: longPress', [{
  origin?: WebElement,
  offset?: { x: number; y: number },
  locator?: Locator,
}]);
```

**Endpoint:** `POST /session/{sessionId}/appium/gestures/long_press`

---

#### `flutter: dragAndDrop`

Drag from source to target.

```typescript
await driver.executeScript('flutter: dragAndDrop', [{
  source: WebElement,    // Required: Source element
  target: WebElement,    // Required: Target element
}]);
```

**Endpoint:** `POST /session/{sessionId}/appium/gestures/drag_drop`

---

#### `flutter: scrollTillVisible`

Scroll until element becomes visible.

```typescript
await driver.executeScript('flutter: scrollTillVisible', [{
  finder: Locator | WebElement,     // Required: Element to find
  scrollView?: WebElement,          // Scrollable container
  delta?: number,                   // Scroll amount (default: 100)
  maxScrolls?: number,              // Max scroll attempts (default: 50)
  settleBetweenScrollsTimeout?: number,  // Wait between scrolls (ms)
  dragDuration?: number,            // Drag animation duration (ms)
  scrollDirection?: 'up' | 'down' | 'left' | 'right',
}]);
```

**Endpoint:** `POST /session/{sessionId}/appium/gestures/scroll_till_visible`

**Returns:** Found element object

---

### Wait Commands

#### `flutter: waitForVisible`

Wait for element to become visible.

```typescript
await driver.executeScript('flutter: waitForVisible', [{
  element?: WebElement,
  locator?: Locator,
  timeout?: number,  // Milliseconds
}]);
```

**Endpoint:** `POST /session/{sessionId}/element/wait/visible`

---

#### `flutter: waitForAbsent`

Wait for element to disappear.

```typescript
await driver.executeScript('flutter: waitForAbsent', [{
  element?: WebElement,
  locator?: Locator,
  timeout?: number,
}]);
```

**Endpoint:** `POST /session/{sessionId}/element/wait/absent`

---

### App Management

#### `flutter: launchApp`

Launch or relaunch the application.

```typescript
await driver.executeScript('flutter: launchApp', [{
  appId: string,           // Required: Package name (Android) or bundle ID (iOS)
  arguments?: string[],    // Launch arguments
  environment?: Record<string, string>,  // Environment variables
}]);
```

**Endpoint:** Platform-specific (mobile: launchApp / mobile: activateApp)

**Note:** On iOS, automatically attaches `--flutter-server-port` argument.

---

#### `flutter: renderTree`

Get Flutter render tree for debugging.

```typescript
await driver.executeScript('flutter: renderTree', [{
  widgetType?: string,  // Filter by widget type
  text?: string,        // Filter by text content
  key?: string,         // Filter by key
}]);
```

**Endpoint:** `POST /session/{sessionId}/element/render_tree`

---

### Image Mocking

#### `flutter: injectImage`

Inject a base64-encoded image for camera/image picker mocking.

```typescript
await driver.executeScript('flutter: injectImage', [{
  base64Image: string,  // Required: Base64 encoded image
}]);
```

**Endpoint:** `POST /session/{sessionId}/inject_image`

**Returns:** `imageId` string

**Android Permissions:** Automatically requests storage permissions based on API level.

---

#### `flutter: activateInjectedImage`

Activate a previously injected image.

```typescript
await driver.executeScript('flutter: activateInjectedImage', [{
  imageId: string,  // Required: ID from injectImage
}]);
```

**Endpoint:** `POST /session/{sessionId}/activate_inject_image`

---

## Platform-Specific APIs

### Android

#### `startAndroidSession`

```typescript
async startAndroidSession(...args): Promise<AndroidUiautomator2Driver>
```

Creates UIAutomator2 driver session.

#### `androidPortForward`

```typescript
async androidPortForward(adb: ADB, systemPort: number, devicePort: number)
```

Forwards `systemPort` → `devicePort` via ADB.

#### `androidRemovePortForward`

```typescript
async androidRemovePortForward(adb: ADB, systemPort: number)
```

Removes port forwarding.

---

### iOS

#### `startIOSSession`

```typescript
async startIOSSession(...args): Promise<XCUITestDriver>
```

Creates XCUITest driver session.

#### `iosPortForward`

```typescript
async iosPortForward(udid: string, systemPort: number, devicePort: number)
```

Uses `iproxy` via `DEVICE_CONNECTIONS_FACTORY`.

#### `iosRemovePortForward`

```typescript
iosRemovePortForward(udid: string, systemPort: number)
```

Releases iproxy connection.

---

### macOS

#### `startMacOsSession`

```typescript
async startMacOsSession(...args): Promise<Mac2Driver>
```

Creates Mac2 driver session.

#### `macOsPortForward` / `macOsRemovePortForward`

Same as iOS, uses `iproxy`.

---

## Error Codes

### Driver Errors

| Code | Message | Cause | Resolution |
|------|---------|-------|------------|
| `E_FLUTTER_SERVER_NOT_FOUND` | Flutter server is not started | Server not running in app | Ensure `integrationDriver()` called in app |
| `E_VERSION_MISMATCH` | Server version does not satisfy requirement | Driver/server version mismatch | Update Flutter test SDK or driver |
| `E_PACKAGE_MISMATCH` | Unexpected package name | Wrong app connected | Verify app package/bundle ID |
| `E_PORT_FORWARD_FAILED` | Port forwarding failed | Port in use / ADB/iproxy issue | Check port availability, restart ADB |
| `E_SESSION_CREATE_FAILED` | Session creation failed | Capabilities invalid / App not installed | Verify capabilities and app path |

### Server Validation Errors

The driver validates server response at `/status`:

```typescript
interface ServerStatus {
  appInfo: {
    packageName: string;
  };
  serverVersion: string;  // Must satisfy '>=0.0.18 <1.0.0'
}
```

**Validation Checks:**
1. Response is valid object
2. Contains `appInfo.packageName` matching capability
3. Contains `serverVersion` satisfying semver range
4. Package name matches expected app

---

## Type Definitions

### Core Types

```typescript
// From @appium/types
interface DefaultCreateSessionResult<T> {
  sessionId: string;
  capabilities: T;
}

// Driver capabilities constraints
interface FlutterDriverConstraints {
  avd: { isString: true };
  automationName: { isString: true; presence: true };
  platformName: { 
    inclusionCaseInsensitive: ['iOS', 'Android', 'Mac'];
    isString: true; 
    presence: true;
  };
  udid: { isString: true };
  launchTimeout: { isNumber: true };
  flutterServerLaunchTimeout: { isNumber: true };
  flutterSystemPort: { isNumber: true };
  address: { isString: true };
}
```

### Port Callbacks

```typescript
type PortForwardCallback = (
  udid: string,
  systemPort: number,
  devicePort: number
) => Promise<void>;

type PortReleaseCallback = (
  udid: string,
  systemPort: number
) => Promise<void>;
```

### Flutter Locator

```typescript
interface FlutterLocator {
  using: string;      // Strategy (e.g., '-flutter key')
  value?: string;     // Selector value
  selector?: any;     // For complex locators (descendant/ancestor)
  context?: string;   // Parent element ID
}
```

### Gesture Parameters

```typescript
interface DoubleClickParams {
  origin?: WebElement;
  offset?: { x: number; y: number };
  locator?: FlutterLocator;
}

interface LongPressParams extends DoubleClickParams {}

interface DragDropParams {
  source: WebElement;
  target: WebElement;
}

interface ScrollTillVisibleParams {
  finder: WebElement | FlutterLocator;
  scrollView?: WebElement;
  delta?: number;
  maxScrolls?: number;
  settleBetweenScrollsTimeout?: number;
  dragDuration?: number;
  scrollDirection?: 'up' | 'down' | 'left' | 'right';
}

interface WaitParams {
  element?: WebElement;
  locator?: FlutterLocator;
  timeout?: number;
}

interface LaunchAppParams {
  appId: string;
  arguments?: string[];
  environment?: Record<string, string>;
}

interface RenderTreeParams {
  widgetType?: string;
  text?: string;
  key?: string;
}

interface InjectImageParams {
  base64Image: string;
}

interface ActivateInjectedImageParams {
  imageId: string;
}
```

---

## WDIO Service API

The `wdio-flutter-by-service` provides browser/element commands:

### Locator Commands (Browser & Element)

| Command | Returns | Description |
|---------|---------|-------------|
| `flutterByValueKey(value)` | `Locator` | Get locator by ValueKey |
| `flutterByValueKey$(value)` | `Element` | Find single element by ValueKey |
| `flutterByValueKey$$(value)` | `Element[]` | Find multiple by ValueKey |
| `flutterBySemanticsLabel(label)` | `Locator` | Get locator by semantics label |
| `flutterBySemanticsLabel$(label)` | `Element` | Find by semantics label |
| `flutterBySemanticsLabel$$(label)` | `Element[]` | Find multiple by semantics label |
| `flutterByText(text)` | `Locator` | Get locator by exact text |
| `flutterByText$(text)` | `Element` | Find by exact text |
| `flutterByText$$(text)` | `Element[]` | Find multiple by text |
| `flutterByType(type)` | `Locator` | Get locator by widget type |
| `flutterByType$(type)` | `Element` | Find by widget type |
| `flutterByType$$(type)` | `Element[]` | Find multiple by type |
| `flutterByTextContaining(text)` | `Locator` | Get locator by partial text |
| `flutterByTextContaining$(text)` | `Element` | Find by partial text |
| `flutterByTextContaining$$(text)` | `Element[]` | Find multiple by partial text |

### Advanced Locators

| Command | Parameters | Returns |
|---------|------------|---------|
| `flutterByDescendant({of, matching})` | `of: Element, matching: Element` | `Locator` |
| `flutterByDescendant$({of, matching})` | `of: Element, matching: Element` | `Element` |
| `flutterByDescendant$$({of, matching})` | `of: Element, matching: Element` | `Element[]` |
| `flutterByAncestor({of, matching})` | `of: Element, matching: Element` | `Locator` |
| `flutterByAncestor$({of, matching})` | `of: Element, matching: Element` | `Element` |
| `flutterByAncestor$$({of, matching})` | `of: Element, matching: Element` | `Element[]` |

### Action Commands (Browser)

| Command | Parameters | Returns |
|---------|------------|---------|
| `flutterWaitForVisible({element, locator, timeout})` | `WaitParams` | `void` |
| `flutterWaitForAbsent({element, locator, timeout})` | `WaitParams` | `void` |
| `flutterDoubleClick({element, offset})` | `DoubleClickParams` | `void` |
| `flutterLongPress({element, offset})` | `LongPressParams` | `void` |
| `flutterScrollTillVisible(options)` | `ScrollTillVisibleParams` | `Element \| null` |
| `flutterDragAndDrop({source, target})` | `DragDropParams` | `void` |
| `flutterInjectImage(filePath)` | `string` | `Promise<string>` (imageId) |
| `flutterActivateInjectedImage({imageId})` | `ActivateInjectedImageParams` | `Promise<string>` |

---

## Maintenance Notes

- **Version Compatibility:** Driver version 2.x requires Appium 3.x and Flutter server `>=0.0.18 <1.0.0`
- **Port Ranges:** Device ports 9000-9020, System ports 10000-11000
- **Server Validation:** Always validates `/status` endpoint on connection
- **Element Cache:** Uses `Map` keyed by element ID → proxy driver

---

*Last updated: 2025 | Driver version: 2.0.3*