# Solution 3: Driver Code Fix (If You Can Modify Driver)

**Root Cause Fix for Hybrid App Element Command Routing Issue**

---

## Problem Summary

When using `automationName: FlutterIntegration` in a hybrid app (native Android/iOS elements + Flutter elements), `ExpectedConditions` like `visibilityOf()` and `elementToBeClickable()` fail on native elements because:

1. **Element finding works correctly** — `findElement` with native strategy (`xpath`, `id`, `-android uiautomator`) routes to platform driver (UIA2/XCUITest) and caches element with platform proxy
2. **Subsequent commands route incorrectly** — `getElementRect`, `getAttribute`, `elementDisplayed` are in `isFlutterDriverCommand()` list, so `executeCommand()` routes them to Flutter server instead of checking element cache
3. **Flutter server doesn't recognize native element IDs** — Returns `null` or wrong type, causing `ExpectedConditions` to fail

---

## Solution Overview

**Two-part fix:**

1. **`src/driver.ts`** — Modify `executeCommand()` to check `ELEMENT_CACHE` for proxy ownership before context-based routing
2. **`src/utils.ts`** — Narrow `isFlutterDriverCommand()` to only session management commands

---

## Detailed Implementation

### Part 1: Modify `executeCommand()` in `src/driver.ts`

#### Add Import

```typescript
// src/driver.ts - Add at top with other imports (around line 15)
import { ELEMENT_CACHE } from './commands/element.js';
```

#### Replace `executeCommand()` Method

**Current implementation (lines ~230-250):**
```typescript
async executeCommand(command: any, ...args: any) {
   if (
      this.currentContext === this.NATIVE_CONTEXT_NAME &&
      isFlutterDriverCommand(command)
   ) {
      this.log.debug(
         `executeCommand: command ${command} is flutter command using flutter driver`,
      );
      return await super.executeCommand(command, ...args);
   } else {
      this.log.info(
         `Executing the command: ${command} with args: ${args} and flutterCommand ${isFlutterDriverCommand(command)}`,
      );
   }

   this.handleContextSwitch(command, args);
   logger.default.info(
      `Executing the proxy command: ${command} with args: ${args}`,
   );
   return await this.proxydriver.executeCommand(command as string, ...args);
}
```

**Fixed implementation:**
```typescript
async executeCommand(command: any, ...args: any) {
   const elementId = args[0];  // First argument is typically element ID for element commands
   
   // NEW: Check if element belongs to platform driver cache
   if (elementId && typeof elementId === 'string' && ELEMENT_CACHE.has(elementId)) {
      const cachedProxy = ELEMENT_CACHE.get(elementId);
      
      // If element was found via platform driver (not Flutter proxy), route there
      if (cachedProxy !== this.proxy) {
         this.log.debug(
            `Routing ${command} to platform driver (element ${elementId} owned by platform proxy)`
         );
         return await this.proxydriver.executeCommand(command as string, ...args);
      }
   }
   
   // Existing logic: Flutter commands in NATIVE_APP context → Flutter server
   if (this.currentContext === this.NATIVE_CONTEXT_NAME && 
       isFlutterDriverCommand(command)) {
      this.log.debug(
         `executeCommand: command ${command} is flutter command using flutter driver`,
      );
      return await super.executeCommand(command, ...args);
   } 
   
   // All other commands → platform driver
   this.log.debug(`Executing command ${command} via platform driver`);
   this.handleContextSwitch(command, args);
   return await this.proxydriver.executeCommand(command as string, ...args);
}
```

---

### Part 2: Simplify `isFlutterDriverCommand()` in `src/utils.ts`

#### Current Implementation (lines ~75-95)

```typescript
export function isFlutterDriverCommand(command: string) {
   return (
      [
         'createSession',
         'deleteSession',
         'getSession',
         'getSessions',
         'findElement',
         'findElements',
         'findElementFromElement',
         'findElementsFromElement',
         'click',
         'getText',
         'setValue',
         'keys',
         'getName',
         'clear',
         'elementSelected',
         'elementEnabled',
         'getAttribute',
         'elementDisplayed',
         'execute',
         'getElementRect',
         'getSize',
      ].indexOf(command) >= 0
   );
}
```

#### Fixed Implementation

```typescript
export function isFlutterDriverCommand(command: string) {
   // Only session management commands that must go through Flutter driver
   // Element commands (click, getText, getAttribute, getElementRect, etc.)
   // now route based on element ownership via ELEMENT_CACHE
   return (
      [
         'createSession',
         'deleteSession',
         'getSession',
         'getSessions',
      ].indexOf(command) >= 0
   );
}
```

---

## Commands Removed from `isFlutterDriverCommand()`

| Command | Reason for Removal |
|---------|-------------------|
| `findElement`, `findElements`, `findElementFromElement`, `findElementsFromElement` | Handled by `findElOrEls()` which uses `getProxyDriver()` based on **locator strategy** |
| `click`, `getText`, `setValue`, `clear` | Handled by element.ts methods (`click()`, `getText()`, `setValue()`, `clear()`) which use `ELEMENT_CACHE.get(elementId)` |
| `getAttribute`, `elementDisplayed`, `elementEnabled` | Handled by element.ts methods which use `ELEMENT_CACHE.get(elementId)` |
| `getElementRect`, `getSize` | Handled by element.ts methods which use `ELEMENT_CACHE.get(elementId)` |
| `execute` | Generic script execution, not Flutter-specific |
| `keys`, `getName`, `elementSelected` | Legacy/rarely used commands |

---

## How Element Routing Works After Fix

### Element Finding (Already Correct)

```typescript
// src/commands/element.ts - findElOrEls()
export async function findElOrEls(
   this: AppiumFlutterDriver,
   strategy: string,
   selector: string,
   mult: boolean,
   context: string,
): Promise<any> {
   const driver = await getProxyDriver.bind(this)(strategy);  // Routes by STRATEGY
   // ...
   ELEMENT_CACHE.set(element.ELEMENT || element[W3C_ELEMENT_KEY], driver);  // Caches PROXY
   return element;
}
```

- Flutter strategy (`-flutter key`, `key`, `semantics label`, etc.) → Returns `this.proxy` (Flutter server)
- Native strategy (`xpath`, `id`, `-android uiautomator`, etc.) → Returns platform driver's JWproxy (UIA2/XCUITest/Mac2)
- Element cached with **the proxy that found it**

### Subsequent Element Commands (Fixed)

```typescript
// src/commands/element.ts - Example: getElementRect()
export async function getElementRect(
   this: AppiumFlutterDriver,
   elementId: string,
) {
   const driver = ELEMENT_CACHE.get(elementId);  // Gets CORRECT proxy from cache
   return await driver.command(`/element/${elementId}/rect`, 'GET', {});
}
```

All element command implementations in `element.ts` already use `ELEMENT_CACHE.get(elementId)` — they just need `executeCommand()` to not intercept them first.

---

## Complete Flow After Fix

```
Test: ExpectedConditions.and(
         visibilityOf(nativeElement),      // getElementRect, getAttribute('displayed')
         elementToBeClickable(nativeElement) // getAttribute('enabled'), getElementRect
      )

┌─────────────────────────────────────────────────────────────────────────────┐
│ STEP 1: findElement("xpath", "//button[@id='access_account']")             │
│   getProxyDriver("xpath") → returns UIA2 jwproxy ✓                         │
│   ELEMENT_CACHE.set(elementId, UIA2_jwproxy) ✓                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ STEP 2: getElementRect(elementId)                                          │
│   executeCommand("getElementRect", elementId)                              │
│   ELEMENT_CACHE.has(elementId) = TRUE                                      │
│   cachedProxy = UIA2_jwproxy ≠ this.proxy (Flutter proxy)                  │
│   ROUTES TO: proxydriver.executeCommand() → UIA2 ✓                         │
│   Returns valid rect {x, y, width, height} ✓                               │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ STEP 3: getAttribute("enabled", elementId)                                 │
│   executeCommand("getAttribute", "enabled", elementId)                     │
│   ELEMENT_CACHE.has(elementId) = TRUE                                      │
│   cachedProxy = UIA2_jwproxy ≠ this.proxy                                  │
│   ROUTES TO: UIA2 ✓                                                        │
│   Returns boolean true/false ✓                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ STEP 4: ExpectedConditions evaluates correctly ✓                            │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Edge Cases Handled

| Scenario | Behavior |
|----------|----------|
| Element not in cache (first command on element) | Falls through to context-based routing |
| Flutter element (`-flutter key`) | Cached with `this.proxy` → routes to Flutter server ✓ |
| Native element (`xpath`, `id`) | Cached with platform proxy → routes to platform driver ✓ |
| WebView context (`WEBVIEW_*`) | `proxyWebViewActive = true` → `proxyActive()` logic unchanged |
| `execute` script command | Not in `isFlutterDriverCommand` list → routes to platform driver |
| Mac2Driver non-blocking click | Handled in `element.ts:click()`, unaffected |

---

## Files Modified Summary

| File | Change | Approximate Lines |
|------|--------|-------------------|
| `src/driver.ts` | Add `ELEMENT_CACHE` import + modify `executeCommand()` | ~15 (import), ~230-250 |
| `src/utils.ts` | Simplify `isFlutterDriverCommand()` | ~75-95 |

---

## Testing the Fix

### 1. Build Driver

```bash
cd /path/to/appium-flutter-integration-driver
npm run build
```

### 2. Run Unit Tests

```bash
npm run test:unit
```

### 3. Integration Test (Hybrid App Scenario)

Create test file: `test/specs/hybrid-app.e2e.js`

```javascript
describe('Hybrid app: Native → Flutter transition', () => {
   it('should handle ExpectedConditions on native elements', async () => {
      // 1. Find native element (e.g., "Access Account" button in native header)
      const accessBtn = await browser.findElement('id', 'access_account_button');
      
      // 2. Wait for visibility (uses getElementRect, getAttribute internally)
      await browser.waitUntil(
        async () => {
          const displayed = await accessBtn.isDisplayed();
          const enabled = await accessBtn.isEnabled();
          return displayed && enabled;
        },
        { timeout: 10000, timeoutMsg: 'Native button not clickable' }
      );
      
      // 3. Click native element
      await accessBtn.click();
      
      // 4. Now interact with Flutter elements
      const flutterBtn = await browser.flutterByValueKey$('account_button');
      await flutterBtn.click();
      
      // 5. Verify Flutter interaction worked
      await expect(browser.flutterByText$('Account Details')).toBeDisplayed();
   });
});
```

Run:
```bash
npm run wdio-android
# or
npm run wdio-ios
```

### 4. Unit Test for Command Routing

```typescript
// test/unit/command-routing.test.ts
import { AppiumFlutterDriver } from '../../src/driver.js';
import { ELEMENT_CACHE } from '../../src/commands/element.js';

describe('Command routing with element cache', () => {
   let driver: AppiumFlutterDriver;
   const mockFlutterProxy = { command: vi.fn() };
   const mockPlatformProxy = { command: vi.fn() };
   
   beforeEach(() => {
      ELEMENT_CACHE.clear();
      driver = new AppiumFlutterDriver({}, false);
      driver.proxy = mockFlutterProxy;
      driver.proxydriver = { executeCommand: vi.fn() } as any;
      driver.currentContext = 'NATIVE_APP';
   });
   
   it('routes native element commands to platform driver', async () => {
      const nativeElementId = 'native-123';
      ELEMENT_CACHE.set(nativeElementId, mockPlatformProxy);
      
      await driver.executeCommand('getElementRect', nativeElementId);
      
      expect(driver.proxydriver.executeCommand).toHaveBeenCalledWith('getElementRect', nativeElementId);
      expect(mockFlutterProxy.command).not.toHaveBeenCalled();
   });
   
   it('routes Flutter element commands to Flutter server', async () => {
      const flutterElementId = 'flutter-456';
      ELEMENT_CACHE.set(flutterElementId, mockFlutterProxy);
      
      await driver.executeCommand('getElementRect', flutterElementId);
      
      expect(mockFlutterProxy.command).toHaveBeenCalled();
      expect(driver.proxydriver.executeCommand).not.toHaveBeenCalled();
   });
});
```

---

## Backward Compatibility

| Concern | Mitigation |
|---------|------------|
| Existing Flutter tests | Unchanged — Flutter elements cached with `this.proxy` |
| Custom commands (`flutter:*`) | Handled by `execute()` method, not `executeCommand()` |
| WebView context | `proxyActive()` and `getProxyAvoidList()` unchanged |
| iOS/macOS | Same `ELEMENT_CACHE` logic applies to XCUITest/Mac2 proxies |

---

## Related Files (for reference)

| File | Role |
|------|------|
| `src/driver.ts` | Main driver, `executeCommand()`, `execute()`, command routing |
| `src/utils.ts` | `isFlutterDriverCommand()`, `getProxyDriver()`, port forwarding |
| `src/commands/element.ts` | `ELEMENT_CACHE`, `findElOrEls()`, element command implementations |
| `src/session.ts` | Platform driver creation (`proxydriver`) |

---

## Rollback Plan

If issues arise:

```bash
# Revert changes
git checkout src/driver.ts src/utils.ts

# Rebuild
npm run build
```

---

## References

- [Hybrid Flutter Integration Issue Analysis](./hybrid_flutter_integration_issue.md) — Root cause analysis
- [Architecture Overview](./ARCHITECTURE.md) — Command routing, element cache, port forwarding
- [API Reference](./API.md) — Driver class, element commands, custom commands

---

*Generated: 2025 | Driver version: 2.x*