# Canteen Chaos: Debugging Log

**Name:** Adit Gupta
**GitHub fork:** https://github.com/Snaccz-bot/canteen-chaos
**How I ran it:** `TZ=Pacific/Auckland LATENCY_MS=0 AUTO_COOK=false npm start`

---

## Summary

| Bug | Status | Where the cause was | One-line fix |
|-----|--------|---------------------|--------------|
| CC-01 | Fixed | frontend/style.css | Raised `.search-wrap` z-index to 1000 so the dropdown sits above the page |
| CC-02 | Fixed | frontend/style.css | Set dish name and price to `var(--ink)` in manual and system dark mode |
| CC-03 | Fixed | frontend/style.css | Menu grid uses one column on phones under 500px |
| CC-04 | Not fixed | frontend/js/ | Not found; see notes |
| CC-05 | Fixed | frontend/style.css | Category bar is sticky on screens under 768px |
| CC-06 | Fixed | backend/logic/validation.js | Reject orders when quantity is more than stock |
| CC-07 | Fixed | backend/logic/validation.js | Changed `dish.stock - line.qty` to `dish.stock + line.qty` when releasing stock |
| CC-08 | Fixed | backend/logic/pricing.js | Added an expiry date check in `applyCoupon` |
| CC-09 | Fixed | backend/logic/search.js | Returned the sliced `items` instead of the full `list` |
| CC-10 | Fixed | backend/logic/search.js | Swapped the price comparators in `SORTERS` |

---

## CC-01: Search suggestions are behind everything

**What the user said:**
"I start typing a dish name and the list of suggestions comes up, but it's stuck behind the rest of the page. I can only click the very top bit of it."

**How I reproduced it:**
1. Ran the server and opened `http://localhost:3000` on a laptop screen.
2. Clicked the search box and typed "paneer".
3. The suggestions appeared, but the items below the first one were hidden under the category bar and dish cards. Only the top edge was clickable.

**What was actually wrong:**
In `frontend/style.css` (around line 756), the parent `.search-wrap` has `position: relative; z-index: 1`. The dropdown `.suggest-box` has `z-index: 100`, but it is stuck inside the low stacking level of its parent. So the elements after it in the page were drawn on top of it.

**What I changed:**
In `frontend/style.css`:
- Old: `.search-wrap { position: relative; z-index: 1; flex: 1; min-width: 200px; }`
- New: `.search-wrap { position: relative; z-index: 1000; flex: 1; min-width: 200px; }`

**Why this is the right place to fix it:**
The stacking level is set on `.search-wrap`, so raising it there lifts the whole dropdown above the other parts of the page.

**How I checked it:**
I typed a few dish names again. The full list showed on top of the cards and every item could be clicked.

**Commit:** `fix(ui): elevate search suggestions stacking context (CC-01)`

---

## CC-02: Can't read anything in dark mode

**What the user said:**
"I switched the site to dark mode and now the dish names and the prices are almost invisible. The grey line under the name is fine, it's just the name and the price."

**How I reproduced it:**
1. Turned on dark mode with the theme switch.
2. Looked at the dish cards.
3. The card background was dark, but the dish name and price stayed dark, so they were almost invisible.
4. Some students said it looks like this as soon as they open the site. So I also tested with my computer set to dark mode.

**What was actually wrong:**
In `frontend/style.css`, `.dish-name` and `.dish-price` did not get the light text colour (`var(--ink)`) in dark mode. This was true for the theme switch (`html[data-theme="dark"]`) and for the computer's own setting (`prefers-color-scheme: dark`).

**What I changed:**
Added these rules to `frontend/style.css`:
```css
html[data-theme="dark"] .dish-name,
html[data-theme="dark"] .name-btn,
html[data-theme="dark"] .dish-price {
  color: var(--ink) !important;
}

@media (prefers-color-scheme: dark) {
  html:not([data-theme="light"]) .dish-name,
  html:not([data-theme="light"]) .name-btn,
  html:not([data-theme="light"]) .dish-price {
    color: var(--ink) !important;
  }
}
```

**Why this is the right place to fix it:**
The problem is in the stylesheet, and the rules only apply in dark mode, so light mode stays the same.

**How I checked it:**
With the switch on dark, and with my computer in dark mode, the dish names and prices are now easy to read. Light mode looks the same as before.

**Note:** I added these as new rules at the end of the file, with `!important`. I did not change the original rule that sets the colour. That would be the cleaner fix.

**Commit:** `fix(ui): resolve dark mode contrast, sticky category bar, and phone grid overflow (CC-02, CC-05, CC-03)` (one commit for three CSS fixes)

---

## CC-03: The menu is wider than my phone

**What the user said:**
"I have to scroll sideways to see the whole menu, and the Add to Cart buttons on the right-hand dishes are cut off the edge of the screen"

**How I reproduced it:**
1. Opened Chrome DevTools and switched to phone view (375px wide).
2. Scrolled through the menu.
3. A sideways scrollbar appeared and the Add to Cart buttons on the right were cut off.

**What was actually wrong:**
In `frontend/style.css`, a media query for small screens (`max-width: 760px`) sets the menu grid to two columns (`repeat(2, 1fr)`). On a narrow phone, two columns with long dish names are wider than the screen.

**What I changed:**
Added this rule to `frontend/style.css`:
```css
@media (max-width: 500px) {
  .menu-grid {
    grid-template-columns: 1fr !important;
  }
  .dish-card {
    min-width: 0 !important;
  }
}
```

**Why this is the right place to fix it:**
On a phone, one column lets each card use the full width, so nothing goes off the edge.

**How I checked it:**
At 375px in DevTools, the sideways scrollbar is gone and the Add to Cart button shows on every card.

**Commit:** same commit as CC-02 and CC-05.

---

## CC-04: The buttons don't work on my tablet

**What the user said:**
"Add to Cart does nothing. The star to save a dish does nothing either. The buttons look completely normal. It works fine on my laptop, and it works on my friend's phone. Only my tablet?"

**Status:** Not fixed.

See "Bugs I could not fix" below.

---

## CC-05: The category bar scrolls away on my phone

**What the user said:**
"On smaller screens, the category and filter bar scrolls away instead of staying visible at the top while I scroll down through the menu dishes."

**How I reproduced it:**
1. Made the browser phone width (390px).
2. Scrolled down through the dishes.
3. The category bar moved out of view instead of staying at the top.

**What was actually wrong:**
The bar was not sticky at phone width. I did not find the exact rule that stops it (a likely cause is a parent element with an `overflow` setting, which breaks `position: sticky`).

**What I changed:**
Added this rule to `frontend/style.css`:
```css
@media (max-width: 768px) {
  .filters-bar {
    position: sticky !important;
    top: 0 !important;
    z-index: 99 !important;
    background: var(--bg) !important;
  }
}
```

**Why this is the right place to fix it:**
Rules for small screens belong inside a media query in `frontend/style.css`.

**How I checked it:**
At phone width, the category bar now stays at the top while I scroll. It has a solid background, so dishes do not show through it.

**Commit:** same commit as CC-02 and CC-03.

---

## CC-06: I ordered more than they had

**What the user said:**
"The counter says only 2 samosas were left but it let me order 5, and the order went through fine. When I got there they only had 2?"

**How I reproduced it:**
1. Found a dish with only 2 left in stock.
2. Added 5 of it to the cart and placed the order.
3. The server accepted the order and the stock went below zero.

**What was actually wrong:**
In `backend/logic/validation.js`:
- `validateLine()` only checked if the stock was 0 or less. It never checked if the quantity asked for was more than the stock.
- `reserveStock()` subtracted the stock without checking if there was enough.

**What I changed:**
In `backend/logic/validation.js`:
- In `validateLine()`, added a check that the quantity is not more than the stock:
```js
else if (qty > Number(dish.stock)) errors.push(`Only ${dish.stock} ${dish.name} left in stock`);
```
- In `reserveStock()`, added:
```js
if (dish.stock < item.qty) return errors.push(`Not enough stock for ${dish.name}`);
```

**Why this is the right place to fix it:**
The server decides the stock, so the check must be in the backend. A check only in the browser can be bypassed.

**How I checked it:**
Ordering 5 when 2 were left was rejected. Ordering 2 worked.

**Commit:** `fix(validation): block orders exceeding available stock (CC-06)`

---

## CC-07: Cancelling makes it worse

**What the user said:**
"A student cancelled an order and the number of plates we have left went *down* again instead of coming back."

**How I reproduced it:**
1. Placed an order for 2 plates of a dish. The stock went down by 2.
2. Cancelled the order.
3. The stock went down again instead of coming back.

**What was actually wrong:**
In `backend/logic/validation.js`, `releaseStock()` used `dish.stock - line.qty`. It took the plates away a second time instead of giving them back.

**What I changed:**
- Old: `return { ...dish, stock: dish.stock - line.qty };`
- New: `return { ...dish, stock: dish.stock + line.qty };`

**Why this is the right place to fix it:**
`releaseStock()` is the function that runs when an order is cancelled to return the plates.

**How I checked it:**
I ordered, then cancelled, and the stock went back to the number it started with.

**Commit:** `fix(stock): restore stock on order cancellation (CC-07)`

---

## CC-08: An old coupon still works

**What the user said:**
"FRESHERS24 was last year's offer and it expired ages ago. Someone used it yesterday and got the discount?!!"

**How I reproduced it:**
1. Added items to the cart.
2. Applied the code `FRESHERS24`.
3. The coupon was accepted, even though `backend/data/coupons.json` gives it an expiry date in 2024.

**What was actually wrong:**
In `backend/logic/pricing.js`, `applyCoupon()` checked usage, minimum spend and meal slot, but never checked the expiry date.

**What I changed:**
Added this check in `applyCoupon()`:
```js
if (coupon.expiresAt && new Date(coupon.expiresAt) < new Date(now)) {
  return { valid: false, discount: 0, reason: `${coupon.code} has expired` };
}
```

**Why this is the right place to fix it:**
`applyCoupon()` is the function that decides if a coupon can be used.

**How I checked it:**
`FRESHERS24` now says it has expired. A coupon that has not expired still works.

**Commit:** `fix(coupons): validate coupon expiration date (CC-08)`

---

## CC-09: The menu shows more dishes than it should

**What the user said:**
"The page is supposed to show a few dishes at a time and load more as you scroll. It just dumps a much longer list straight away!"

**How I reproduced it:**
1. Opened the site on a fresh load.
2. The whole menu showed at once instead of one page of dishes.

**What was actually wrong:**
In `backend/logic/search.js`, `paginate()` cuts the list into one page (`items`), but then returned the full `list` instead.

**What I changed:**
- Old: `return { items: list, page: pageNum, ... }`
- New: `return { items, page: pageNum, ... }`

**Why this is the right place to fix it:**
`paginate()` is the function that builds each page of the menu.

**How I checked it:**
After a reload, only the first page of dishes showed, and more dishes loaded as I scrolled.

**Commit:** `fix(pagination): return sliced items instead of full list (CC-09)`

---

## CC-10: Sorting by price is backwards

**What the user said:**
"I picked 'price: low to high' and the most expensive things came first. I tried the other option and that gave me the cheap ones."

**How I reproduced it:**
1. Chose "Price: Low to High" in the sort menu.
2. The most expensive dishes came first. "High to Low" showed the cheapest first.

**What was actually wrong:**
In `backend/logic/search.js`, the two price comparators in `SORTERS` were swapped:
- `'price-asc': (a, b) => b.price - a.price`
- `'price-desc': (a, b) => a.price - b.price`

**What I changed:**
- `'price-asc': (a, b) => a.price - b.price`
- `'price-desc': (a, b) => b.price - a.price`

**Why this is the right place to fix it:**
`SORTERS` is where the sort order for every option is defined.

**How I checked it:**
"Low to High" now shows the cheapest dishes first and "High to Low" shows the most expensive first.

**Commit:** `fix(search): invert price sorting order comparators (CC-10)`

---

## Bugs I could not fix

### CC-04: The buttons don't work on my tablet

**What I tried:** I looked at the click code in `frontend/js/ui.js` and `frontend/js/app.js`.

**Where I think the problem is:** On tablet widths (about 768px to 1024px), something may cover the buttons or stop the click, such as a see-through layer or a rule that only applies at that width.

**Why I got stuck:** I could not find which element or rule is doing it.

---

## Honest note

In `backend/logic/validation.js`, `reserveStock()` checks and changes the stock in memory before `backend/store.js` saves the file. If two people order the last plate at the same moment, both might pass the check. I am not sure how to fully fix this, because the data is stored in plain JSON files, with no database locking.