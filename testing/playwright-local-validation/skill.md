# Playwright Local Validation

Run end-to-end tests locally against your development stack to validate features before pushing to CI.

---

## Quick Start

```bash
# Run all E2E tests
npx playwright test

# Run specific test file
npx playwright test e2e/contact.spec.ts

# Run tests matching a pattern
npx playwright test --grep "should create"

# Run with visible browser (headed mode)
npx playwright test --headed

# Run single worker for debugging
npx playwright test --workers=1

# Debug mode (opens inspector)
npx playwright test --debug
```

---

## Before Running Tests

Ensure the local stack is running:

```bash
docker-compose up -d
docker-compose logs -f backend    # Wait for "Uvicorn running on http://0.0.0.0:8000"
docker-compose logs -f frontend   # Wait for "start worker process"
```

Health check:
```bash
curl -s http://localhost/health && echo "✓ Frontend"
curl -s http://localhost:8000/health && echo "✓ Backend"
```

---

## Test Structure

Tests live in `e2e/` directory organized by feature:

| File | Purpose | Notes |
|------|---------|-------|
| `contact.spec.ts` | Contact form submission | Happy path, validation, API 500 mock |
| `cv-download.spec.ts` | CV download flow | Popup handling, locale selection |
| `uigen-crud-authenticated.spec.ts` | UIGen CRUD operations | Auth flow, protected endpoints |
| `uigen-crud.spec.ts` | UIGen discovery tests | Button locators, form inspection |

---

## Common Patterns

### Wait for Element
```typescript
await page.locator("button").first().waitFor({ timeout: 5000 });
```

### Screenshot
```typescript
await page.screenshot({ path: "debug-step1.png" });
```

### Fill Form & Submit
```typescript
const input = page.locator("input[name='email']");
await input.fill("test@example.com");

const submitBtn = page.locator("button[type='submit']");
await submitBtn.click();
```

### Check API Response
```typescript
const response = await page.request.post("http://localhost:8000/api/contact", {
  data: { name: "Test", email: "test@example.com", message: "Hello" }
});
expect(response.status()).toBe(200);
```

### Mock API Response
```typescript
await page.route("**/api/cv/**", route => {
  route.abort("failed");
});
```

### Capture Console Logs
```typescript
const logs = [];
page.on("console", msg => logs.push(msg.text()));

// ... test code ...

console.log("Captured logs:", logs);
```

### Handle Popups
```typescript
const [popup] = await Promise.all([
  page.waitForEvent("popup"),
  page.locator("a[target='_blank']").click()
]);
await popup.waitForLoadState();
```

---

## Debugging Failed Tests

### View test report
```bash
npx playwright show-report
```

### Run in debug mode
```bash
npx playwright test e2e/contact.spec.ts --debug
```

### Check screenshots
Failed tests create screenshots in `test-results/` directory:
```bash
ls -lah test-results/*.png
open test-results/contact-step-1.png
```

### Increase timeouts for slow CI/containers
```typescript
test.setTimeout(30000);  // 30 second timeout for this test
```

---

## CI Integration

Tests run automatically on:
- PR creation → `test-frontend.yml` (manual dispatch)
- After dev deploy → runs inside `deploy-pr-preview.yml`
- Prod smoke tests → `smoke-test-prod.yml` (3x daily on weekdays)

Local test results won't affect CI until you push.

---

## Tips

**Isolate tests**: Use `test.only()` to run one test:
```typescript
test.only("should do something", async ({ page }) => {
  // ...
});
```

**Skip tests**: Use `test.skip()`:
```typescript
test.skip("not ready yet", async ({ page }) => {
  // ...
});
```

**Conditional waits**: Don't assume elements are immediately ready:
```typescript
const isVisible = await page.locator("button").isVisible({ timeout: 5000 }).catch(() => false);
if (isVisible) { /* ... */ }
```

**Check page content when lost**:
```typescript
const text = await page.content();
console.log(text.substring(0, 500));
```

**List all buttons to find targets**:
```typescript
const buttons = await page.locator("button").all();
for (const btn of buttons) {
  const text = await btn.textContent();
  console.log(`Button: "${text}"`);
}
```

---

## Cleanup

Remove test artifacts:
```bash
rm -f *.png test-results/**/*.png
```

Stop containers when done:
```bash
docker-compose down
```
