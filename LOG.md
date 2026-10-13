# Canteen Chaos: Debugging Log

**Name:** Adit Gupta
**GitHub fork:** https://github.com/Snaccz-bot/canteen-chaos
**How I ran it:** `TZ=Pacific/Auckland LATENCY_MS=0 AUTO_COOK=false npm start`

---

## Summary

| Bug | Status | Where the cause was | One-line fix |
|-----|--------|---------------------|--------------|
| CC-01 | Fixed | frontend/style.css | Raised `.search-wrap` z-index to 1000 so the dropdown sits above the page |
| CC-02 | Partly fixed | frontend/style.css | Set dish name and price to `var(--ink)` in dark mode; see notes |
| CC-03 | Not fixed | frontend/style.css | Suspected fixed grid widths; see notes |
| CC-04 | Not fixed | frontend/js/ | Not found yet; see notes |
| CC-05 | Not fixed | frontend/style.css | Suspected parent overflow breaking sticky; see notes |
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
3. The suggestion list appeared, but most of it was under the category bar and dish cards. Only the top edge was clickable.

**What was actually wrong:**
In `frontend/style.css`, the parent `.search-wrap` has `position: relative; z-index: 1`. The dropdown `.suggest-box` has `z-index: 100`, but it is stuck inside the low stacking level of its parent. So the elements after it in the page were drawn on top.

**What I changed:**
In `frontend/style.css`:
- Old: `.search-wrap { position: relative; z-index: 1; flex: 1; min-width: 200px; }`
- New: `.search-wrap { position: relative; z-index: 1000; flex: 1; min-width: 200px; }`

**Why this is the right place to fix it:**
The stacking level is set on `.search-wrap`, so raising it there lifts the whole dropdown above the other parts of the page.

**How I checked it:**
I typed a dish name again. The full list showed on top of the cards and every item was clickable.

**Commit:** `fix(ui): elevate search suggestions stacking context (CC-01)`

---

## CC-02: Can't read anything in dark mode

**What the user said:**
"I switched the site to dark mode and now the dish names and the prices are almost invisible. The grey line under the name is fine, it's just the name and the price."

**How I reproduced it:**
1. Switched the site to dark mode.
2. Looked at the dish cards in the menu.
3. The card background is dark, but the dish name and price stayed dark too, so they were hard to read.

**What was actually wrong:**
In `frontend/style.css`, `.dish-name` and `.dish-price` did not get a light text colour in dark mode.

**What I changed:**
Added this rule in `frontend/style.css`:
```css
html[data-theme="dark"] .dish-name,
html[data-theme="dark"] .dish-price {
  color: var(--ink);
}
```

**Why this is the right place to fix it:**
The rule is only for dark mode, so light mode stays the same.

**How I checked it:**
I switched between light and dark. The name and price are readable in dark mode when I use the theme switch.

**Commit:** `fix(ui): improve dish typography contrast in dark mode (CC-02)`

**Not finished:**
Some students said the problem shows up the moment they open the site, without changing anything. My rule only covers the theme switch. I did not check if the site also follows the computer's own dark mode setting (`prefers-color-scheme`), so that part may still be broken.

---

## CC-03: The menu is wider than my phone

**What the user said:**
"I have to scroll sideways to see the whole menu, and the Add to Cart buttons on the right-hand dishes are cut off the edge of the screen"

**How I reproduced it:**
1. Opened Chrome DevTools and switched to phone view (about 375px wide).
2. Scrolled through the menu.
3. A sideways scrollbar appeared and the buttons on the right were cut off.

**What was actually wrong (my best guess):**
In `frontend/style.css`, `.menu-grid` seems to use a fixed minimum column width, so the grid is wider than a small screen.

**What I tried:**
Setting `.menu-grid` to one column and removing the fixed minimum width.

**Status:** Not fixed. I did not find a version that I was happy to commit.

---

## CC-04: The buttons don't work on my tablet

**What the user said:**
"Add to Cart does nothing. The star to save a dish does nothing either. The buttons look completely normal. It works fine on my laptop, and it works on my friend's phone. Only my tablet?"

**How I reproduced it:**
Not reproduced yet. It seems to need a tablet-sized screen.

**Where I think the problem is:**
Something at tablet width may cover the buttons or stop the click. I looked at `frontend/js/ui.js` and `frontend/js/app.js` but did not find the cause.

**Status:** Not fixed.

---

## CC-05: The category bar scrolls away on my phone

**What the user said:**
"On smaller screens, the category and filter bar scrolls away instead of staying visible at the top while I scroll down through the menu dishes."

**How I reproduced it:**
1. Switched DevTools to a phone width.
2. Scrolled down the menu.
3. The category bar moved out of view instead of staying at the top.

**What was actually wrong (my best guess):**
`position: sticky` stops working if a parent element has an `overflow` setting (like `hidden`). I think a parent of the bar has one at small screen sizes.

**What I tried:**
Looking for the parent with `overflow` in `frontend/style.css`.

**Status:** Not fixed.

---

## CC-06: I ordered more than they had

**What the user said:**
"The counter says only 2 samosas were left but it let me order 5, and the order went through fine. When I got there they only had 2?"

**How I reproduced it:**
1. Found a dish with only 2 left in stock.
2. Added 5 of it to the cart and placed the order.
3. The server accepted it and the stock went below zero.

**What was actually wrong:**
In `backend/logic/validation.js`:
- `validateLine()` only checked if stock was 0 or less. It never checked if the quantity asked for was more than the stock.
- `reserveStock()` subtracted the stock without checking if there was enough.

**What I changed:**
In `backend/logic/validation.js`:
- In `validateLine()`, added a check: if `qty > Number(dish.stock)`, add the error `Only ${dish.stock} ${dish.name} left in stock`.
- In `reserveStock()`, added a check: if `dish.stock < item.qty`, return the error `Not enough stock for ${dish.name}`.

**Why this is the right place to fix it:**
The server decides stock, so the check must be in the backend. The browser can be bypassed.

**How I checked it:**
Ordering 5 when 2 were left was rejected. Ordering 2 worked.

**Commit:** `fix(validation): block orders exceeding available stock (CC-06)`

---

## CC-07: Cancelling makes it worse

**What the user said:**
"A student cancelled an order and the number of plates we have left went *down* again instead of coming back."

**How I reproduced it:**
1. Placed an order for 2 plates of a dish. Stock went down by 2.
2. Cancelled the order.
3. Stock went down again instead of coming back.

**What was actually wrong:**
In `backend/logic/validation.js`, `releaseStock()` used `stock: dish.stock - line.qty`. It took away the plates a second time instead of giving them back.

**What I changed:**
- Old: `return { ...dish, stock: dish.stock - line.qty };`
- New: `return { ...dish, stock: dish.stock + line.qty };`

**Why this is the right place to fix it:**
`releaseStock()` is the function that runs when an order is cancelled to return the plates.

**How I checked it:**
I ordered, cancelled, and the stock went back to the starting number.

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
`FRESHERS24` now says it has expired. A coupon that is still valid still works.

**Commit:** `fix(coupons): validate coupon expiration date (CC-08)`

---

## CC-09: The menu shows more dishes than it should

**What the user said:**
"The page is supposed to show a few dishes at a time and load more as you scroll. It just dumps a much longer list straight away!"

**How I reproduced it:**
1. Opened the site on a fresh load.
2. The whole menu showed at once instead of one page of dishes.

**What was actually wrong:**
In `backend/logic/search.js`, `paginate()` cuts the list into a page (`items`), but then returned the full `list` instead.

**What I changed:**
- Old: `return { items: list, page: pageNum, ... }`
- New: `return { items, page: pageNum, ... }`

**Why this is the right place to fix it:**
`paginate()` is the function that builds each page for the menu.

**How I checked it:**
After reloading, only the first page of dishes showed, and more loaded as I scrolled.

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

**Commit:** `fix(search): correct price sorting comparators (CC-10)`

---

## Bugs I could not fix

**CC-03:** I tried one column and no fixed minimum width. I think the cause is the fixed column width in `.menu-grid`. I did not find a change that fixed it without making the cards look wrong.

**CC-04:** I read the click code in `ui.js` and `app.js` but did not find what stops the buttons on a tablet. My guess is something covering the buttons at that screen width.

**CC-05:** I think a parent element has an `overflow` setting that breaks `position: sticky`. I did not finish finding and fixing it.

---

## Honest note

In `backend/logic/validation.js`, `reserveStock()` checks and changes stock in memory before the file is saved by `backend/store.js`. If two people order the last plate at the same moment, both might pass the check. I am not sure how to fully fix this, because the data is stored in plain JSON files with no database locking.