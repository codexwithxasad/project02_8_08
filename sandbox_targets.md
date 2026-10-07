# Approved Test Sandboxes (automation QA targets)

Purpose-built test environments. No real accounts, no ToS constraints.
Selected 2026-10-06 for browser-harness and verification-pipeline testing.

| # | Target | URL | Focus |
|---|--------|-----|-------|
| 1 | SauceDemo | https://www.saucedemo.com/ | E2E e-commerce: login, browse, cart, checkout |
| 2 | The Internet | https://the-internet.herokuapp.com/ | Auth forms, dynamic controls, file upload, JS UI |
| 3 | Expand Testing | https://practice.expandtesting.com/ | Input validation, alerts, forms |

## Test plan per target
1. Land on page, wait_for_load, page_info + screenshot
2. Map interactive elements (AX tree first, DOM fallback)
3. Execute the core flow end to end
4. Verify result via a second independent source
5. Record selectors, coordinates, failure modes into skill
