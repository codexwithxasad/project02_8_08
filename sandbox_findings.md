# Sandbox Test Findings — 2026-10-06

Testing the Hermes browser harness against approved test sandboxes.
Purpose: validate form-fill, verification, and interaction patterns for
the automation pipeline.

## SauceDemo (https://www.saucedemo.com/)

### Selector drift from documented v1
The site has been rewritten since the classic Selenium docs. Current (2026-10):

- **Login error** is at `[data-test="error"]` (h3, `role="alert"`), NOT `#error`.
- Inputs have `data-test="username"` and `data-test="password"`.
- Cart entry button is `[data-test=shopping-cart-link]`, NOT `-button`.
- Checkout flow buttons change per step: `continue` on step 1, `finish` on step 2.
- Submit button uses `value="Login"` attribute, NOT `innerText` (empty string).
- `[data-test=add-to-cart-ITEM]` flips to `[data-test=remove-ITEM]` after click
  (React state update). Re-query every time.
- Cart state persists server-side per user; a fresh login may start with items.

### React input trap — verified
**`fill_input()` silently does nothing on SauceDemo.** The helper dispatches
synthetic `new Event('input', {bubbles:true})`, which React's document-level
event delegation does NOT recognize. Result: `value` stays `''` and the app
never sees the field change.

**Use `type_text()`** (fires real key events, React sees them) or the
native setter workaround:

```js
const setter = Object.getOwnPropertyDescriptor(
  Object.getPrototypeOf(el), 'value').set;
setter.call(el, val);
el.dispatchEvent(new Event('input', {bubbles:true}));
```

Note: `document.querySelector('#user-name').value = 'x'` sets the DOM value
but the React app NEVER registers it (form submits with empty state). This is
worse than fill_input because it *looks* like it worked.

### Verified flows
- Login (happy): `standard_user` / `secret_sauce` → `/inventory.html`, 6 items
- Login (bad pass): "Epic sadface: Username and password do not match any user"
- Login (locked out): "Epic sadface: Sorry, this user has been locked out."
- Login (empty pass): "Epic sadface: Password is required"
- Checkout complete: `/checkout-complete.html`, "Thank you for your order!",
  cart badge element removed from DOM
- Add to cart via `[data-test=add-to-cart-ITEM]` — flips to `remove-ITEM`
- Product sort: `[data-test=product-sort-container]` opens dropdown, click
  `.dropdown_list li` with matching innerText

## The Internet (https://the-internet.herokuapp.com/)

**The site has drifted substantially from the widely-cited Elementals
Selenium documentation.** Multiple scenarios now use different selectors,
frameworks, and behaviors than documented. Verified drift:

### Form Authentication (`/login`)
- Password is `SuperSecretPassword!` (NOT `SuperSecret!` — that's a
  documentation error that has circulated for years).
- Username field hint text: "Enter tomsmith for the username and
  SuperSecretPassword! for the password."
- Success: `flash success | You logged into a secure area!` → `/secure`
- Bad password: `flash error | Your password is invalid!`
- Empty user: `flash error | Your username is invalid!`

### Drag and Drop (`/drag_and_drop`)
- **Completely rewritten** as HTML5 DnD. No `<li>` elements; two `<div>`
  columns with `draggable="true"` and `<header>` children only.
- jQuery UI Sortable no longer in use.
- Mouse event clicks do NOT move items. HTML5 DnD requires `DragEvent`
  dispatch:
```js
const data = new DataTransfer();
src.dispatchEvent(new DragEvent('dragstart', {bubbles:true, dataTransfer:data}));
dst.dispatchEvent(new DragEvent('dragenter', {bubbles:true, dataTransfer:data}));
dst.dispatchEvent(new DragEvent('dragover',  {bubbles:true, dataTransfer:data}));
dst.dispatchEvent(new DragEvent('drop',      {bubbles:true, dataTransfer:data}));
src.dispatchEvent(new DragEvent('dragend',   {bubbles:true, dataTransfer:data}));
```
- Verified: A↔B headers swap after dispatch.

### Dynamic Controls (`/dynamic_controls`)
- Buttons have plain text (`Remove`, `Enable`), no CSS classes or IDs.
- No `#checkbox` id — target `input[type=checkbox]` positionally.
- Remove checkbox: works, count 1→0 after 3s.
- **Enable button appears unresponsive** — text input stays `disabled=true`
  after both `.click()` and CDP coordinate click. May be a real regression
  on this test page or a known broken state.

### Notification Messages (`/notification_message`)
- No `#message` button. Message shown in `.alert` above heading.
- Reload via `<a>Click here</a>` link (find by innerText).
- Close via `a[href="#"]` (note: bare `#` selector requires quotes).

## Cross-cutting lessons

### React controlled inputs
`fill_input()` works on most React apps because they typically listen on
the input element. On strict React (like SauceDemo), the document-root
delegation misses synthetic events. **Rule:** after any `fill_input()`,
verify `value !== ''` in a follow-up `js()` call. If empty, fall back to
`type_text()` or the native setter.

### Selector drift on public fixtures
Both SauceDemo and The Internet have rewritten their DOMs since the
classic documentation. **Never hardcode selectors from Stack Overflow /
blog posts.** Discover them on load:

```python
js("""(() => [...document.querySelectorAll('[data-test]')]
  .map(e => e.getAttribute('data-test'))
  .filter((v,i,a) => a.indexOf(v) === i)
  .sort())()""")
```

Then match by stable attributes or innerText, not by class name.

### Button label flips
React stateful buttons (Add↔Remove, Open↔Close, Ask↔Send) change their
`data-test` attribute or `innerText` when state changes. Always re-query
by attribute/text before acting; never cache a reference.

### HTML5 DnD vs jQuery UI
The Internet's drag-and-drop page migrated from jQuery UI Sortable to
HTML5 DnD. jQuery-style `Input.dispatchMouseEvent` with mouse moves does
NOT trigger HTML5 DnD handlers. Use `DragEvent` dispatch via `js()`.

### Async UI updates
The Internet's dynamic controls use ~2-3s delays. Poll for the expected
state rather than assuming a fixed sleep works; the delay varies.
