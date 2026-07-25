# E2E Testing with Playwright

End-to-end testing guidelines for WooCommerce plugins using Playwright.

**IMPORTANT:** Always ask the user before running E2E tests. These tests interact with a real WordPress/WooCommerce environment and may modify data.

## References

- [WooCommerce E2E Testing Docs](https://developer.woocommerce.com/docs/contribution/testing/)
- [WooCommerce E2E Tests Repository](https://github.com/woocommerce/woocommerce/tree/trunk/plugins/woocommerce/tests/e2e-pw)
- [Playwright Best Practices](https://playwright.dev/docs/best-practices)

## Setup

### Installation

```bash
npm init playwright@latest
npm install @wordpress/env --save-dev
```

### wp-env Configuration

```json
// .wp-env.json
{
  "core": null,
  "phpVersion": "8.0",
  "plugins": [
    "https://downloads.wordpress.org/plugin/woocommerce.latest-stable.zip",
    "."
  ],
  "mappings": {
    "wp-content/plugins/my-plugin": "."
  },
  "config": {
    "WP_DEBUG": true,
    "SCRIPT_DEBUG": true
  },
  "port": 8889
}
```

### Playwright Configuration

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './tests/e2e',
  timeout: 30000,
  retries: process.env.CI ? 2 : 0,
  outputDir: './test-results',
  use: {
    baseURL: process.env.BASE_URL || 'http://localhost:8889',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
    trace: 'on-first-retry',
  },
  projects: [
    {
      name: 'setup',
      testMatch: /.*\.setup\.ts/,
    },
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] },
      dependencies: ['setup'],
    },
  ],
  reporter: [
    ['html', { outputFolder: 'test-results/playwright-report' }],
    ['json', { outputFile: 'test-results/results.json' }],
  ],
});
```

### Environment Variables

```bash
# .env (do not commit)
BASE_URL=http://localhost:8889
ADMIN_USER=admin
ADMIN_PASSWORD=password
CUSTOMER_USER=customer@example.com
CUSTOMER_PASSWORD=customer123
```

**Warning:** Some test scripts may overwrite custom `.env` files. Use `npx playwright` commands directly to preserve custom configs.

## Directory Structure

Following WooCommerce's official structure:

```
tests/
└── e2e/
    ├── bin/                      # Utility scripts
    ├── fixtures/
    │   ├── auth.fixture.ts       # Login fixtures
    │   └── wc.fixture.ts         # WooCommerce helpers
    ├── pages/
    │   ├── checkout.page.ts      # Page object models
    │   ├── cart.page.ts
    │   └── my-account.page.ts
    ├── test-data/
    │   └── data.ts               # Test data defaults
    ├── utils/
    │   └── helpers.ts            # Common utilities
    ├── tests/
    │   ├── merchant/             # Admin/merchant tests
    │   ├── shopper/              # Customer tests
    │   └── api/                  # API tests
    └── global.setup.ts           # Auth state setup
```

## Authentication Setup

### Global Auth Setup

```typescript
// tests/e2e/global.setup.ts
import { chromium, FullConfig } from '@playwright/test';

async function globalSetup(config: FullConfig) {
  const browser = await chromium.launch();
  const adminPage = await browser.newPage();

  // Admin login
  await adminPage.goto('/wp-admin/');
  await adminPage.fill('#user_login', process.env.ADMIN_USER!);
  await adminPage.fill('#user_pass', process.env.ADMIN_PASSWORD!);
  await adminPage.click('#wp-submit');
  await adminPage.waitForURL(/wp-admin/);
  await adminPage.context().storageState({ path: '.auth/admin.json' });

  // Customer login
  const customerPage = await browser.newPage();
  await customerPage.goto('/my-account/');
  await customerPage.fill('#username', process.env.CUSTOMER_USER!);
  await customerPage.fill('#password', process.env.CUSTOMER_PASSWORD!);
  await customerPage.click('button[name="login"]');
  await customerPage.waitForURL(/my-account/);
  await customerPage.context().storageState({ path: '.auth/customer.json' });

  await browser.close();
}

export default globalSetup;
```

### Auth Fixtures

```typescript
// tests/e2e/fixtures/auth.fixture.ts
import { test as base } from '@playwright/test';

export const test = base.extend({
  adminPage: async ({ browser }, use) => {
    const context = await browser.newContext({
      storageState: '.auth/admin.json',
    });
    const page = await context.newPage();
    await use(page);
    await context.close();
  },

  customerPage: async ({ browser }, use) => {
    const context = await browser.newContext({
      storageState: '.auth/customer.json',
    });
    const page = await context.newPage();
    await use(page);
    await context.close();
  },
});

export { expect } from '@playwright/test';
```

## Page Object Models

### Checkout Page (Block Checkout)

```typescript
// tests/e2e/pages/checkout.page.ts
import { Page, Locator, expect } from '@playwright/test';

export class CheckoutPage {
  readonly page: Page;
  readonly emailInput: Locator;
  readonly firstNameInput: Locator;
  readonly lastNameInput: Locator;
  readonly addressInput: Locator;
  readonly cityInput: Locator;
  readonly postcodeInput: Locator;
  readonly phoneInput: Locator;
  readonly placeOrderButton: Locator;
  readonly orderReceivedHeading: Locator;

  constructor(page: Page) {
    this.page = page;
    // Block checkout selectors
    this.emailInput = page.locator('#email');
    this.firstNameInput = page.locator('#billing-first_name');
    this.lastNameInput = page.locator('#billing-last_name');
    this.addressInput = page.locator('#billing-address_1');
    this.cityInput = page.locator('#billing-city');
    this.postcodeInput = page.locator('#billing-postcode');
    this.phoneInput = page.locator('#billing-phone');
    this.placeOrderButton = page.locator('button:has-text("Place Order")');
    this.orderReceivedHeading = page.locator('h1:has-text("Order received")');
  }

  async goto() {
    await this.page.goto('/checkout/');
  }

  async fillBillingDetails(details: {
    email: string;
    firstName: string;
    lastName: string;
    address: string;
    city: string;
    postcode: string;
    phone: string;
  }) {
    await this.emailInput.fill(details.email);
    await this.firstNameInput.fill(details.firstName);
    await this.lastNameInput.fill(details.lastName);
    await this.addressInput.fill(details.address);
    await this.cityInput.fill(details.city);
    await this.postcodeInput.fill(details.postcode);
    await this.phoneInput.fill(details.phone);
  }

  async placeOrder() {
    await this.placeOrderButton.click();
    await expect(this.orderReceivedHeading).toBeVisible({ timeout: 15000 });
  }
}
```

### Classic Checkout Page

```typescript
// tests/e2e/pages/checkout-classic.page.ts
import { Page, Locator, expect } from '@playwright/test';

export class CheckoutClassicPage {
  readonly page: Page;

  constructor(page: Page) {
    this.page = page;
  }

  // Classic checkout selectors (different from block checkout)
  get emailInput() { return this.page.locator('#billing_email'); }
  get firstNameInput() { return this.page.locator('#billing_first_name'); }
  get lastNameInput() { return this.page.locator('#billing_last_name'); }
  get addressInput() { return this.page.locator('#billing_address_1'); }
  get cityInput() { return this.page.locator('#billing_city'); }
  get postcodeInput() { return this.page.locator('#billing_postcode'); }
  get phoneInput() { return this.page.locator('#billing_phone'); }
  get countrySelect() { return this.page.locator('#billing_country'); }
  get placeOrderButton() { return this.page.locator('#place_order'); }

  async goto() {
    await this.page.goto('/checkout/');
  }

  async fillBillingDetails(details: {
    email: string;
    firstName: string;
    lastName: string;
    address: string;
    city: string;
    postcode: string;
    phone: string;
    country?: string;
  }) {
    await this.firstNameInput.fill(details.firstName);
    await this.lastNameInput.fill(details.lastName);

    if (details.country) {
      await this.countrySelect.selectOption(details.country);
    }

    await this.addressInput.fill(details.address);
    await this.cityInput.fill(details.city);
    await this.postcodeInput.fill(details.postcode);
    await this.phoneInput.fill(details.phone);
    await this.emailInput.fill(details.email);
  }

  async placeOrder() {
    await this.placeOrderButton.click();
    await expect(this.page.locator('.woocommerce-order-received')).toBeVisible({ timeout: 15000 });
  }
}
```

### Cart Page

```typescript
// tests/e2e/pages/cart.page.ts
import { Page, Locator, expect } from '@playwright/test';

export class CartPage {
  readonly page: Page;
  readonly cartTable: Locator;
  readonly emptyCartMessage: Locator;
  readonly checkoutButton: Locator;
  readonly updateCartButton: Locator;

  constructor(page: Page) {
    this.page = page;
    this.cartTable = page.locator('.woocommerce-cart-form');
    this.emptyCartMessage = page.locator('.cart-empty');
    this.checkoutButton = page.locator('a:has-text("Proceed to checkout")');
    this.updateCartButton = page.locator('button[name="update_cart"]');
  }

  async goto() {
    await this.page.goto('/cart/');
  }

  async getItemCount(): Promise<number> {
    const rows = this.page.locator('.woocommerce-cart-form__cart-item');
    return await rows.count();
  }

  async removeItem(productName: string) {
    const row = this.page.locator(`.woocommerce-cart-form__cart-item:has-text("${productName}")`);
    await row.locator('.remove').click();
  }

  async updateQuantity(productName: string, quantity: number) {
    const row = this.page.locator(`.woocommerce-cart-form__cart-item:has-text("${productName}")`);
    await row.locator('input.qty').fill(String(quantity));
    await this.updateCartButton.click();
  }

  async proceedToCheckout() {
    await this.checkoutButton.click();
  }
}
```

## WooCommerce API Helpers

```typescript
// tests/e2e/fixtures/wc.fixture.ts
import { test as base } from '@playwright/test';

interface WcApi {
  createProduct(data: ProductData): Promise<number>;
  deleteProduct(id: number): Promise<void>;
  createOrder(data: OrderData): Promise<number>;
  deleteOrder(id: number): Promise<void>;
}

interface ProductData {
  name: string;
  regular_price: string;
  type?: 'simple' | 'variable';
}

interface OrderData {
  status?: string;
  customer_id?: number;
  line_items: Array<{ product_id: number; quantity: number }>;
}

export const test = base.extend<{ wcApi: WcApi }>({
  wcApi: async ({ request }, use) => {
    const baseURL = process.env.BASE_URL || 'http://localhost:8889';
    const auth = Buffer.from(
      `${process.env.WC_API_KEY}:${process.env.WC_API_SECRET}`
    ).toString('base64');

    const api: WcApi = {
      async createProduct(data: ProductData) {
        const response = await request.post(`${baseURL}/wp-json/wc/v3/products`, {
          headers: { Authorization: `Basic ${auth}` },
          data,
        });
        const json = await response.json();
        return json.id;
      },

      async deleteProduct(id: number) {
        await request.delete(`${baseURL}/wp-json/wc/v3/products/${id}?force=true`, {
          headers: { Authorization: `Basic ${auth}` },
        });
      },

      async createOrder(data: OrderData) {
        const response = await request.post(`${baseURL}/wp-json/wc/v3/orders`, {
          headers: { Authorization: `Basic ${auth}` },
          data,
        });
        const json = await response.json();
        return json.id;
      },

      async deleteOrder(id: number) {
        await request.delete(`${baseURL}/wp-json/wc/v3/orders/${id}?force=true`, {
          headers: { Authorization: `Basic ${auth}` },
        });
      },
    };

    await use(api);
  },
});

export { expect } from '@playwright/test';
```

## Test Specs

### Shopper Tests

```typescript
// tests/e2e/tests/shopper/checkout.spec.ts
import { test, expect } from '../../fixtures/auth.fixture';
import { CheckoutPage } from '../../pages/checkout.page';

test.describe('Shopper Checkout Flow', () => {
  test('guest can complete checkout', async ({ page }) => {
    // Add product to cart
    await page.goto('/product/test-product/');
    await page.click('button:has-text("Add to cart")');
    await expect(page.locator('.added_to_cart')).toBeVisible();

    // Go to checkout
    const checkout = new CheckoutPage(page);
    await checkout.goto();

    // Fill details and place order
    await checkout.fillBillingDetails({
      email: 'test@example.com',
      firstName: 'John',
      lastName: 'Doe',
      address: '123 Main St',
      city: 'Athens',
      postcode: '12345',
      phone: '1234567890',
    });

    await checkout.placeOrder();
    await expect(page.locator('text=Thank you')).toBeVisible();
  });

  test('logged-in customer has pre-filled details', async ({ customerPage }) => {
    await customerPage.goto('/product/test-product/');
    await customerPage.click('button:has-text("Add to cart")');
    await customerPage.goto('/checkout/');

    const emailInput = customerPage.locator('#email');
    await expect(emailInput).toHaveValue(process.env.CUSTOMER_USER!);
  });
});
```

### Merchant Tests

```typescript
// tests/e2e/tests/merchant/settings.spec.ts
import { test, expect } from '../../fixtures/auth.fixture';

test.describe('Merchant Settings', () => {
  test('can access plugin settings', async ({ adminPage }) => {
    await adminPage.goto('/wp-admin/admin.php?page=my-plugin-settings');
    await expect(adminPage.locator('h1')).toContainText('My Plugin Settings');
  });

  test('can save settings', async ({ adminPage }) => {
    await adminPage.goto('/wp-admin/admin.php?page=my-plugin-settings');

    await adminPage.fill('#my_plugin_option', 'new value');
    await adminPage.click('input[type="submit"]');

    await expect(adminPage.locator('.notice-success')).toBeVisible();
    await expect(adminPage.locator('#my_plugin_option')).toHaveValue('new value');
  });
});
```

### API Tests

```typescript
// tests/e2e/tests/api/products.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Products API', () => {
  const baseURL = process.env.BASE_URL || 'http://localhost:8889';
  const auth = Buffer.from(
    `${process.env.WC_API_KEY}:${process.env.WC_API_SECRET}`
  ).toString('base64');

  test('can create a product', async ({ request }) => {
    const response = await request.post(`${baseURL}/wp-json/wc/v3/products`, {
      headers: { Authorization: `Basic ${auth}` },
      data: {
        name: 'API Test Product',
        regular_price: '29.99',
        type: 'simple',
      },
    });

    expect(response.ok()).toBeTruthy();
    const product = await response.json();
    expect(product.name).toBe('API Test Product');

    // Cleanup
    await request.delete(`${baseURL}/wp-json/wc/v3/products/${product.id}?force=true`, {
      headers: { Authorization: `Basic ${auth}` },
    });
  });
});
```

### Store API Tests (Block Checkout Path)

Block cart/checkout goes through `/wc/store/v1` — not `/wc/v3`. This is the API that exercises `woocommerce_store_api_*` hooks, ExtendSchema data, and Additional Checkout Fields (see `05-woocommerce.md`), so plugin checkout extensions must be tested here:

```typescript
// tests/e2e/tests/api/store-checkout.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Store API Checkout', () => {
  const baseURL = process.env.BASE_URL || 'http://localhost:8889';

  test('cart → checkout flow', async ({ request }) => {
    // The Store API is cookie/nonce based — the first cart response returns the nonce.
    const cartRes = await request.get(`${baseURL}/wp-json/wc/store/v1/cart`);
    const nonce = cartRes.headers()['nonce'];

    const addRes = await request.post(`${baseURL}/wp-json/wc/store/v1/cart/add-item`, {
      headers: { Nonce: nonce },
      data: { id: Number(process.env.TEST_PRODUCT_ID), quantity: 1 },
    });
    expect(addRes.ok()).toBeTruthy();

    const address = {
      first_name: 'John', last_name: 'Doe', address_1: '123 Main St',
      city: 'Athens', postcode: '11111', country: 'GR',
    };

    const checkoutRes = await request.post(`${baseURL}/wp-json/wc/store/v1/checkout`, {
      headers: { Nonce: nonce },
      data: {
        billing_address: { ...address, email: 'test@example.com', phone: '2101234567' },
        shipping_address: address,
        payment_method: 'cod',
        // Additional Checkout Fields are posted under their location key:
        additional_fields: { 'nvm-plugin/vat-number': 'EL123456789' },
      },
    });
    expect(checkoutRes.ok()).toBeTruthy();

    const order = await checkoutRes.json();
    expect(order.order_id).toBeGreaterThan(0);
    // Assert YOUR extension's effect on the order via /wc/v3 (admin auth) here.
  });

  test('checkout rejects invalid extension data', async ({ request }) => {
    const cartRes = await request.get(`${baseURL}/wp-json/wc/store/v1/cart`);
    const nonce = cartRes.headers()['nonce'];

    const res = await request.post(`${baseURL}/wp-json/wc/store/v1/checkout`, {
      headers: { Nonce: nonce },
      data: { additional_fields: { 'nvm-plugin/vat-number': 'not-a-vat' } },
    });
    expect(res.status()).toBe(400); // Custom validation must reject, not silently accept.
  });
});
```

Playwright's `request` fixture keeps cookies per test, so the cart session persists between the calls.

## Accessibility Testing

Scan every screen the plugin adds or modifies. Failing on serious+ violations keeps regressions out without blocking on pre-existing theme noise:

```bash
npm install @axe-core/playwright --save-dev
```

```typescript
// tests/e2e/tests/shopper/a11y.spec.ts
import { test, expect } from '../../fixtures/auth.fixture';
import AxeBuilder from '@axe-core/playwright';

const scan = (page: import('@playwright/test').Page) =>
  new AxeBuilder({ page }).withTags(['wcag2a', 'wcag2aa']).analyze();

test('checkout has no serious a11y violations', async ({ page }) => {
  await page.goto('/checkout/');
  const results = await scan(page);
  const serious = results.violations.filter(
    (v) => v.impact === 'serious' || v.impact === 'critical'
  );
  expect(serious, JSON.stringify(serious, null, 2)).toEqual([]);
});

test('plugin settings page has no serious a11y violations', async ({ adminPage }) => {
  await adminPage.goto('/wp-admin/admin.php?page=my-plugin-settings');
  const results = await scan(adminPage);
  const serious = results.violations.filter(
    (v) => v.impact === 'serious' || v.impact === 'critical'
  );
  expect(serious, JSON.stringify(serious, null, 2)).toEqual([]);
});
```

To scan only the plugin's own UI inside a noisy page, scope with `.include('#nvm-plugin-root')`.

## Visual Regression (Optional)

For UI-heavy screens (settings pages, checkout customizations):

```typescript
test('settings page renders correctly', async ({ adminPage }) => {
  await adminPage.goto('/wp-admin/admin.php?page=my-plugin-settings');
  await expect(adminPage).toHaveScreenshot('settings-page.png', {
    fullPage: true,
    maxDiffPixelRatio: 0.02,
    mask: [adminPage.locator('#wpadminbar'), adminPage.locator('#footer-upgrade')],
  });
});
```

- Commit the `*-snapshots/` directory; update intentionally with `npx playwright test --update-snapshots`.
- Snapshots are OS/browser-specific — generate and compare them **in CI only** (or in the same container), or the diff is rendering noise, not regressions.
- Mask dynamic regions (admin bar, dates, order numbers).

## Seeding Test Data with WP-CLI

Faster than REST for bulk fixtures and store configuration — use the API fixture for per-test data, WP-CLI for environment state:

```bash
# Products.
npx wp-env run cli wp wc product create \
  --name="E2E Test Product" --type=simple --regular_price=19.99 --user=admin

# Store configuration.
npx wp-env run cli wp option update woocommerce_enable_guest_checkout yes
npx wp-env run cli wp wc payment_gateway update cod --enabled=true --user=admin

# Inspect state while debugging.
npx wp-env run cli wp wc shop_order list --user=admin --format=table
```

Put idempotent seeding in a `tests/e2e/bin/seed.sh` and run it after `wp-env start` — tests must not depend on manually-created data.

## Running Tests

### NPM Scripts

```json
{
  "scripts": {
    "env:start": "wp-env start",
    "env:stop": "wp-env stop",
    "env:restart": "wp-env destroy && wp-env start",
    "test:e2e": "playwright test",
    "test:e2e:ui": "playwright test --ui",
    "test:e2e:debug": "playwright test --debug",
    "test:e2e:headed": "playwright test --headed",
    "test:api": "playwright test tests/api"
  }
}
```

### Commands

```bash
# Start environment
npm run env:start

# Run all tests (headless)
npm run test:e2e

# Run with browser visible
npm run test:e2e:headed

# Interactive UI mode
npm run test:e2e:ui

# Debug mode with inspector
npm run test:e2e:debug

# Run specific test file
npx playwright test checkout.spec.ts

# Run specific folder
npx playwright test tests/merchant

# Reset environment to fresh state
npm run env:restart
```

## Test Reports

### Playwright HTML Report

```bash
npx playwright show-report test-results/playwright-report
```

### Allure Report (optional)

```bash
npm install allure-playwright allure-commandline --save-dev
```

```typescript
// playwright.config.ts
reporter: [
  ['allure-playwright'],
  ['html'],
],
```

```bash
# Generate report
npx allure generate --clean test-results/allure-results -o test-results/allure-report

# Open report
npx allure open test-results/allure-report
```

## CI/CD Integration

```yaml
# .github/workflows/e2e.yml
name: E2E Tests

on:
  pull_request:
    branches: [main]

jobs:
  e2e:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install dependencies
        run: npm ci

      - name: Install Playwright
        run: npx playwright install --with-deps

      - name: Start WordPress
        run: npm run env:start

      - name: Run E2E tests
        run: npm run test:e2e

      - uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: playwright-report
          path: test-results/playwright-report/
```

## Best Practices

### 1. Create Isolated Tests

Each test should be independent and not rely on state from other tests.

### 2. Use Fixtures for Common Setup

```typescript
test.describe('Order Management', () => {
  let productId: number;

  test.beforeAll(async ({ wcApi }) => {
    productId = await wcApi.createProduct({
      name: 'E2E Test Product',
      regular_price: '19.99',
    });
  });

  test.afterAll(async ({ wcApi }) => {
    await wcApi.deleteProduct(productId);
  });

  test('can purchase product', async ({ page }) => {
    await page.goto(`/?p=${productId}`);
    // ... test logic
  });
});
```

### 3. Use Web-First Assertions

```typescript
// Good - waits automatically
await expect(page.locator('.success')).toBeVisible();

// Avoid - manual waits
await page.waitForTimeout(1000);
```

### 4. Prioritize User-Facing Attributes

```typescript
// Good - uses visible text
await page.click('button:has-text("Add to cart")');

// Avoid - relies on implementation details
await page.click('.single_add_to_cart_button');
```

### 5. Detect Checkout Type

```typescript
async function getCheckoutType(page: Page): Promise<'block' | 'classic'> {
  await page.goto('/checkout/');
  const isBlock = await page.locator('.wc-block-checkout').count() > 0;
  return isBlock ? 'block' : 'classic';
}
```

### 6. Wait for AJAX

```typescript
async function waitForWcAjax(page: Page) {
  await page.waitForFunction(() => {
    return !document.body.classList.contains('processing');
  });
}
```

## Debugging

```bash
# Run with browser visible
npx playwright test --headed

# Run with Playwright Inspector
npx playwright test --debug

# Run with UI mode
npx playwright test --ui

# View trace
npx playwright show-trace test-results/trace.zip
```

## Checklist Before Running E2E Tests

- [ ] User confirmed they want to run tests (tests may modify data)
- [ ] WordPress/WooCommerce environment is running (`npm run env:start`)
- [ ] Test products exist or will be created via API
- [ ] Environment variables are configured (`.env`)
- [ ] Playwright browsers are installed (`npx playwright install`)
