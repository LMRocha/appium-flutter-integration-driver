# FLUTTER Context Implementation Analysis

**Adding a `FLUTTER` context to enable explicit switching between platform-native (UIAutomator2/XCUITest/Mac2) and Flutter test server routing within a single session.**

---

## Current Architecture

The driver has **two routing modes** controlled by `currentContext`:

| Context | `currentContext` | `proxyWebViewActive` | Routing Behavior |
|---------|------------------|---------------------|------------------|
| `NATIVE_APP` (default) | `"NATIVE_APP"` | `false` | Flutter commands (`isFlutterDriverCommand()`) → Flutter server; Others → Platform driver |
| `WEBVIEW_*` | `"WEBVIEW_..."` | `true` | Most commands → Platform driver (with proxy logic for Chromedriver/WDA) |

**Problem:** No context exists for "platform-native only" when using `automationName: FlutterIntegration`. Native element commands like `getElementRect`, `getAttribute`, `click` are intercepted by `isFlutterDriverCommand()` and routed to Flutter server.

---

## Required Source Code Changes

### 1. Add Context Constants & State (`src/driver.ts`)

```typescript
// Around lines 86-88
private proxyWebViewActive: boolean = false;
public readonly NATIVE_CONTEXT_NAME: string = `NATIVE_APP`;
public readonly FLUTTER_CONTEXT_NAME: string = `FLUTTER`;  // NEW
public currentContext: string = this.NATIVE_CONTEXT_NAME;
private flutterContextActive: boolean = false;  // NEW
```

### 2. Modify `handleContextSwitch()` (`src/driver.ts` ~line 324)

```typescript
private handleContextSwitch(command: string, args: any[]): void {
   if (command === 'setContext') {
      const contextName = args[0];
      const isWebviewContext = typeof contextName === 'string' && contextName.includes('WEBVIEW');
      const isFlutterContext = contextName === this.FLUTTER_CONTEXT_NAME;  // NEW
      const isNativeAppContext = contextName === this.NATIVE_CONTEXT_NAME;

      if (typeof contextName === 'string' && contextName.length > 0) {
         this.currentContext = contextName;
      } else {
         logger.default.warn(
            `Attempted to set context to invalid value: ${contextName}. Keeping current context: ${this.currentContext}`,
         );
      }

      this.proxyWebViewActive = isWebviewContext;
      this.flutterContextActive = isFlutterContext;  // NEW
      
      // Reset flags for NATIVE_APP
      if (isNativeAppContext) {
         this.proxyWebViewActive = false;
         this.flutterContextActive = false;
      }
   }
}
```

### 3. Modify `executeCommand()` Routing Logic (`src/driver.ts` ~line 297)

```typescript
async executeCommand(command: any, ...args: any) {
   // NEW: FLUTTER context → always Flutter server for Flutter commands
   if (this.flutterContextActive) {
      if (isFlutterDriverCommand(command)) {
         this.log.debug(`Executing ${command} in FLUTTER context via Flutter server`);
         return await super.executeCommand(command, ...args);
      } else {
         // Non-Flutter commands in FLUTTER context: error or fallback
         this.log.warn(`Command ${command} not supported in FLUTTER context. Use NATIVE_APP context for platform commands.`);
         throw new Error(`Command ${command} not supported in FLUTTER context`);
      }
   }

   // EXISTING: NATIVE_APP context → Flutter commands to Flutter server
   if (
      this.currentContext === this.NATIVE_CONTEXT_NAME &&
      isFlutterDriverCommand(command)
   ) {
      this.log.debug(
         `executeCommand: command ${command} is flutter command using flutter driver`,
      );
      return await super.executeCommand(command, ...args);
   }

   // EXISTING: Everything else → Platform driver
   this.handleContextSwitch(command, args);
   this.log.debug(`Executing command ${command} via platform driver`);
   return await this.proxydriver.executeCommand(command as string, ...args);
}
```

### 4. Modify `proxyActive()` (`src/driver.ts` ~line 625)

```typescript
public proxyActive(): boolean {
   // WebView proxying (Android Chromedriver via UIA2)
   if (this.proxyWebViewActive && !(this.proxydriver instanceof XCUITestDriver)) {
      return true;
   }
   // NEW: FLUTTER context doesn't use platform proxy
   if (this.flutterContextActive) {
      return false;
   }
   return false;
}
```

### 5. Implement `getContexts()` (`src/driver.ts` - NEW METHOD)

```typescript
/**
 * Get all available contexts.
 * Returns: NATIVE_APP, FLUTTER, and any WEBVIEW contexts from platform driver.
 */
async getContexts(): Promise<string[]> {
   const contexts: string[] = [this.NATIVE_CONTEXT_NAME, this.FLUTTER_CONTEXT_NAME];
   
   // Add WebView contexts from platform driver
   try {
      const platformContexts = await this.proxydriver.getContexts();
      if (Array.isArray(platformContexts)) {
         platformContexts.forEach(ctx => {
            if (ctx.includes('WEBVIEW') && !contexts.includes(ctx)) {
               contexts.push(ctx);
            }
         });
      }
   } catch (e) {
      this.log.debug('Could not get platform contexts:', e);
   }
   
   return contexts;
}
```

### 6. Implement `getContext()` (`src/driver.ts` - NEW METHOD)

```typescript
/**
 * Get current context.
 */
async getContext(): Promise<string> {
   return this.currentContext;
}
```

---

## Complete Routing Matrix After Changes

| Context | `getElementRect` | `click` | `findElement (-flutter key)` | `findElement (xpath)` | `mobile:*` | `flutter:*` |
|---------|------------------|---------|------------------------------|----------------------|------------|-------------|
| `NATIVE_APP` | Flutter server | Flutter server | Flutter server | **Platform driver** | Platform driver | Flutter server |
| `FLUTTER` (NEW) | Flutter server | Flutter server | Flutter server | **Error/Unsupported** | Error | Flutter server |
| `WEBVIEW_*` | Platform driver | Platform driver | Platform driver | Platform driver | Platform driver | Platform driver |

---

## Usage Example (Test Code)

```javascript
// Step 1: Native Android flow (UIAutomator2)
await driver.switchContext('NATIVE_APP');
const accessBtn = await driver.findElement('id', 'access_account_button');
await driver.waitUntil(
  async () => await accessBtn.isDisplayed() && await accessBtn.isEnabled(),
  { timeout: 10000 }
);
await accessBtn.click();

// Step 2: Flutter flow (Flutter test server)
await driver.switchContext('FLUTTER');
const flutterBtn = await driver.flutterByValueKey$('account_button');
await flutterBtn.click();

// Step 3: Back to native if needed
await driver.switchContext('NATIVE_APP');
```

---

## Impacts & Considerations

### ✅ Benefits

| Benefit | Description |
|---------|-------------|
| **Explicit control** | Test authors choose: `FLUTTER` for Flutter, `NATIVE_APP` for platform |
| **Solves hybrid issue** | `ExpectedConditions` on native elements work in `NATIVE_APP` context |
| **Clear mental model** | Context name matches automation engine |
| **Backward compatible** | Default `NATIVE_APP` behavior unchanged |

### ⚠️ Risks & Breaking Changes

| Impact | Severity | Mitigation |
|--------|----------|------------|
| **Native strategy in `FLUTTER` context** | High | Currently routes to platform driver; in `FLUTTER` would error. Must decide: error, fallback, or support. |
| **WebView context detection** | Medium | `getContexts()` must merge platform driver's WebView contexts |
| **Appium Inspector** | Low | Will show `FLUTTER` as available context |
| **WDIO service** | Medium | `flutter-by` service commands may need context awareness |
| **`execute()` / `executeScript()`** | Medium | `flutter:*` commands should work in both `NATIVE_APP` and `FLUTTER` |

### 🔧 Edge Cases to Handle

```typescript
// In executeCommand() - native strategy in FLUTTER context
async executeCommand(command: any, ...args: any) {
   if (this.flutterContextActive) {
      if (isFlutterDriverCommand(command)) {
         return await super.executeCommand(command, ...args);
      }
      
      // Option A: Strict - throw error (recommended)
      throw new Error(`Native strategy not supported in FLUTTER context. Use NATIVE_APP context.`);
      
      // Option B: Lenient - fallback to platform driver
      // return await this.proxydriver.executeCommand(command as string, ...args);
   }
   // ...
}
```

### 📱 Platform-Specific Impacts

| Platform | Impact |
|----------|--------|
| **Android (UIA2)** | `NATIVE_APP` = UIA2 for native elements; `FLUTTER` = Flutter server |
| **iOS (XCUITest)** | Same as Android; WebView handled by WDA natively |
| **macOS (Mac2)** | Same pattern |

---

## Implementation Priority

### Phase 1: Core Context (Minimal Viable)
1. Add `FLUTTER_CONTEXT_NAME` constant
2. Add `flutterContextActive` flag
3. Modify `handleContextSwitch()` 
4. Modify `executeCommand()` routing
5. Implement `getContexts()` and `getContext()`

### Phase 2: Polish
1. Update `proxyActive()` 
2. Add validation in `executeCommand()` for unsupported commands in `FLUTTER` context
3. Update `WEBVIEW_NO_PROXY` if needed
4. Handle `findElement`/`findElements` in `FLUTTER` context with native strategies

### Phase 3: Testing & Documentation
1. Unit tests for context switching
2. Integration tests: `NATIVE_APP` → native elements, `FLUTTER` → Flutter elements
3. Update `commands.md`, `API.md`, `ONBOARDING.md`

---

## Files Modified Summary

| File | Changes | Lines |
|------|---------|-------|
| `src/driver.ts` | Constants, state, `handleContextSwitch()`, `executeCommand()`, `proxyActive()`, `getContexts()`, `getContext()` | ~100+ |

---

## Comparison with Other Solutions

| Approach | Lines Changed | Test Changes | Breaking Changes | Complexity |
|----------|---------------|--------------|------------------|------------|
| **Solution 3: Element Cache Fix** | ~20 | None | None | Low |
| **FLUTTER Context (This)** | ~100+ | Optional (for explicit control) | Low (new context) | Medium |
| **Separate Sessions (Solution 1)** | 0 | Yes (two driver instances) | None | Low (architectural) |

---

## Recommendation

| Scenario | Recommended Approach |
|----------|---------------------|
| **Fix hybrid issue transparently** | Solution 3 (Element Cache Fix) |
| **Want explicit context control in tests** | FLUTTER Context (this approach) |
| **Maximum isolation, parallel pipelines** | Separate Sessions |

The **FLUTTER Context** approach is technically feasible (~100 lines) but the **Element Cache Fix (Solution 3)** achieves the same user-facing result with less code, zero breaking changes, and no test modifications required.

---

## Rollback Plan

```bash
# Revert changes
git checkout src/driver.ts

# Rebuild
npm run build
```

---

*Generated: 2025 | Based on driver v2.x source code analysis*