---
name: uber-eats
description: Browse restaurants, read menus, place orders, and track deliveries on Uber Eats. Use when the user asks about food delivery, nearby restaurants, ordering food, or anything related to Uber Eats.
allowed-tools: Bash(agent-browser:*)
---

# Uber Eats via agent-browser

Drive the Uber Eats website using `agent-browser`. There is no public API — all interaction happens through the browser.

## Configuration

- **Delivery address:** `2136 Oakvale Road, Walnut Creek, CA 94597` (edit this placeholder with the actual address)
- **Auth state file:** `uber-eats-auth.json` (saved to workspace, persists across sessions)

## First-time setup (login)

Uber Eats requires authentication. Log in once and save the auth state.

```bash
agent-browser open https://www.ubereats.com
agent-browser snapshot -i
# Click "Sign in" button
agent-browser click @eN  # ref for Sign In
agent-browser snapshot -i

# Enter email/phone
agent-browser fill @eN "user@example.com"
agent-browser click @eN  # "Next" button
agent-browser wait --load networkidle
agent-browser snapshot -i

# Enter password
agent-browser fill @eN "password"
agent-browser click @eN  # "Next" / "Log in" button
agent-browser wait --load networkidle
```

If Uber asks for SMS or email verification:

1. Snapshot the page to confirm the verification prompt
2. **Ask the user** for the verification code — never guess or skip this
3. Fill the code field and submit

```bash
agent-browser snapshot -i
# Ask user: "Uber Eats sent a verification code to your phone/email. What's the code?"
agent-browser fill @eN "123456"
agent-browser click @eN  # Verify / Submit
agent-browser wait --load networkidle
```

After successful login, save auth state:

```bash
agent-browser state save uber-eats-auth.json
```

## Starting a session

Always start by loading saved auth state, then open Uber Eats:

```bash
agent-browser state load uber-eats-auth.json
agent-browser open https://www.ubereats.com
agent-browser wait --load networkidle
agent-browser snapshot -i
```

If the snapshot shows a "Sign in" button instead of the user's account, auth has expired — run the login flow again and re-save state.

## Set delivery address

If the page shows the wrong address or asks for one:

```bash
agent-browser snapshot -i
# Click the address/delivery-location element near the top of the page
agent-browser click @eN
agent-browser snapshot -i
# Clear and fill the address input
agent-browser fill @eN "2136 Oakvale Road, Walnut Creek, CA 94597"
agent-browser wait 1500  # Wait for autocomplete suggestions
agent-browser snapshot -i
# Click the first matching suggestion
agent-browser click @eN
agent-browser wait --load networkidle
```

## Workflows

### Browse nearby restaurants

```bash
agent-browser open https://www.ubereats.com
agent-browser wait --load networkidle
agent-browser snapshot -i
```

The homepage shows nearby restaurants. Extract names, cuisines, ratings, delivery times, and fees from the snapshot. Scroll for more results (see "Dynamic loading" below).

### Search restaurants or cuisines

```bash
agent-browser snapshot -i
# Click the search input
agent-browser click @eN
agent-browser fill @eN "sushi"
agent-browser press Enter
agent-browser wait --load networkidle
agent-browser snapshot -i
```

Report results to the user: restaurant names, ratings, delivery time, delivery fee.

### Read a restaurant menu

```bash
# Click the restaurant card from search results or homepage
agent-browser click @eN
agent-browser wait --load networkidle
agent-browser snapshot -i
```

The restaurant page shows:
- **Categories** (e.g., "Popular Items", "Appetizers", "Entrees") — usually as tabs or sections
- **Menu items** with name, description, and price

Scroll down to see all categories. Click a category tab/link to jump to that section.

### View item details and customizations

```bash
# Click a menu item
agent-browser click @eN
agent-browser wait --load networkidle
agent-browser snapshot -i
```

The item detail modal/page shows:
- Full description
- Price
- Required and optional customizations (size, toppings, sides, etc.)
- Quantity selector

### Add item to cart

```bash
# After viewing item details:
# Select any required customizations
agent-browser click @eN  # e.g., size "Large"
agent-browser snapshot -i

# Adjust quantity if needed
agent-browser click @eN  # "+" button to increase

# Click "Add to Cart" / "Add to Order"
agent-browser click @eN
agent-browser wait --load networkidle
```

### View and modify cart

```bash
agent-browser snapshot -i
# Click the cart icon/button (usually shows item count and total)
agent-browser click @eN
agent-browser wait --load networkidle
agent-browser snapshot -i
```

From the cart view you can:
- Adjust item quantities (click +/- buttons)
- Remove items (click remove/trash icon)
- See subtotal, delivery fee, taxes, and total

### Apply promo code

```bash
# From the cart or checkout page:
agent-browser snapshot -i
# Click "Add promo code" or "Promotions"
agent-browser click @eN
agent-browser snapshot -i
agent-browser fill @eN "PROMOCODE"
agent-browser click @eN  # "Apply" button
agent-browser wait --load networkidle
agent-browser snapshot -i
# Confirm the discount was applied
```

### Checkout and place order

**IMPORTANT: Never place an order without explicit user confirmation.**

```bash
# From cart, click "Go to Checkout" / "Checkout"
agent-browser click @eN
agent-browser wait --load networkidle
agent-browser snapshot -i
```

Present the order summary to the user:
- Items and quantities
- Delivery address
- Estimated delivery time
- Payment method
- Subtotal, delivery fee, taxes, tip, total

**Ask the user:** "Here's your order summary: [details]. Total is $X.XX. Should I place the order?"

Only after the user explicitly confirms:

```bash
agent-browser click @eN  # "Place Order" button
agent-browser wait --load networkidle
agent-browser snapshot -i
# Confirm order was placed successfully
```

### Track active orders

```bash
agent-browser open https://www.ubereats.com/orders
agent-browser wait --load networkidle
agent-browser snapshot -i
```

Report the order status: preparing, picked up, en route, delivery ETA, driver name if available.

### View order history and reorder

```bash
agent-browser open https://www.ubereats.com/orders
agent-browser wait --load networkidle
agent-browser snapshot -i
# Past orders are listed — click one for details
agent-browser click @eN
agent-browser wait --load networkidle
agent-browser snapshot -i
```

To reorder, click "Reorder" or add the same items from the restaurant page.

## Dynamic loading (infinite scroll)

Uber Eats uses infinite scroll for restaurant and menu lists. To load more results:

```bash
agent-browser scroll down 800
agent-browser wait 2000
agent-browser snapshot -i
```

Repeat until no new content appears or you have enough results. Compare snapshot output between scrolls — if the content is identical, you've reached the end.

## Error recovery

### Dismiss popups and modals

Uber Eats may show popups (cookie consent, promotions, app download banners). Snapshot and dismiss them:

```bash
agent-browser snapshot -i
# Look for "X", "Close", "Dismiss", "No thanks", or "Accept" buttons
agent-browser click @eN
```

### Auth expired mid-session

If any page redirects to login or shows "Sign in":

1. Re-run the login flow
2. Save fresh auth state
3. Resume the interrupted workflow

### Page loading issues

```bash
agent-browser wait --load networkidle
agent-browser snapshot -i
# If the page looks empty or stuck, reload
agent-browser reload
agent-browser wait --load networkidle
agent-browser snapshot -i
```

### Element not found

If a ref from a previous snapshot no longer works, the page has changed. Take a fresh snapshot:

```bash
agent-browser snapshot -i
# Find the new ref for the element you need
```

## Safety rules

1. **Never place an order without explicit user confirmation.** Always present the full order summary (items, total, address, payment method) and wait for a clear "yes" before clicking "Place Order."
2. **Never store or log payment information.** Do not extract credit card numbers, CVVs, or billing details from the page.
3. **Never change the saved payment method** unless the user explicitly requests it.
4. **Never modify the delivery address** to anything other than what the user specifies.
5. **If anything looks wrong** (unexpected charges, wrong address, unfamiliar payment method), stop and ask the user before proceeding.
