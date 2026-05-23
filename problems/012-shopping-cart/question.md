---
id: 12
title: Design a Shopping Cart Service
category: Session State
topics: [session, persistence, idempotency, inventory check, abandoned cart]
difficulty: Easy
solution: solution.md
---

## Scene

The interviewer slides their laptop towards you.

> *"We are building a small e-commerce site. Shoes, mostly. About 500 customers a day right now. We want a shopping cart. The kind that lets you add things, change quantities, leave for two hours, come back, and still have your cart. Design that. We expect to grow."*

They lean back. "Easy one. Or it should be."

This is the trap. The shopping cart looks like a CRUD problem with three endpoints and a database table. It is not. The real questions hide under the floorboards: where does the cart actually live (the cookie, the server, both), what happens when a guest user logs in and already has a cart in their account, what does "in stock" even mean when the cart is updated five minutes before checkout and inventory changes by the second, and how do you keep all of this fast for a million users without melting your database.

We are going to walk this from a 500-customer startup to a 1M-active-user marketplace. At each stage something cracks. Naming what cracks is half the exercise.

## Step 1: clarify before you design

Take 5 minutes. Do not start drawing. What would you ask the interviewer? Aim for 8 questions that meaningfully change the design.

<details>
<summary><b>Reveal: questions a strong candidate asks</b></summary>

1. **Guest vs logged-in.** "Do anonymous users have carts, or do we force login first?" Almost every real site allows guest carts. This single answer determines whether you have one cart model or two, and whether merging carts at login is part of the design.
2. **Cart persistence horizon.** "If I add an item and close the tab, how long does my cart live? An hour? A day? Forever?" Drives whether the cart lives in memory, in a cookie, or in durable storage. Forever-carts (Amazon-style) need a database. Session-only carts can live in Redis with a TTL.
3. **Multi-device sync.** "If I add a shoe on my phone, do I see it on my laptop when I log in?" This forces server-side state for logged-in users. Without it, you can get away with cookie-only carts.
4. **Inventory accuracy.** "When the cart says 'in stock', does that mean reserved for me, or just visible to me, or last-known-stock-from-five-minutes-ago?" Three very different answers, each with very different complexity. Reservation needs a TTL hold; optimistic is simpler but causes occasional checkout failures; eventual is the cheapest and most painful for users.
5. **Cart size limits.** "Is there a max items per cart? Max quantity per item?" Without limits, scrapers and bots stuff carts with 50,000 items and break your DB. 100 distinct items, 99 units per item, is a typical bound.
6. **Price drift.** "Item was $50 when added, store changed it to $55 by checkout. Whose price wins?" Almost always the current price wins, with a UI banner. But you need to capture the snapshot for fraud detection and customer disputes.
7. **Promotions.** "Are coupons and discounts applied at cart-time or checkout-time? Are they part of this service or a separate one?" Coupons are usually a separate service. The cart holds the coupon code, the pricing engine evaluates it.
8. **Checkout boundary.** "Is checkout part of this design or downstream? Where does the cart end and the order begin?" The cart usually ends when the user clicks 'Place Order'. After that, an order record is created and the cart is cleared (or marked converted).

Bonus questions a senior asks:

- **Abandoned cart recovery.** "Do we send 'you left this in your cart' emails? After how long?" Drives the need for an analytics path on cart state.
- **B2B / large carts.** "Any customers with carts of 500+ SKUs?" If yes, your storage decision changes significantly.

If you walked into the interview asking only "how many users" you would miss the merge logic, the inventory question, and the price drift question. Those three are where the design actually lives.

</details>

## Step 2: capacity estimates

Two scales. The interviewer gives you the small numbers; you derive the big ones from the growth target.

**Small startup (today):**

- 500 daily active users
- 30% of visits add at least one item to cart
- Average cart: 3 items, edited 2 times before checkout
- 60% of carts are abandoned (industry standard)

**One million active users (target):**

- 1M daily active users
- Same conversion behavior
- Peak hour: 3x sustained
- Cart can persist up to 30 days for logged-in users

Compute (do this on paper before revealing):

1. Cart writes per second at both scales
2. Cart reads per second at both scales (every page load with cart-icon shows count)
3. Active carts at any moment
4. Storage for cart data at the 1M scale
5. Inventory check QPS at peak

<details>
<summary><b>Reveal: the math</b></summary>

**Small startup:**

- Carts created: 500 × 30% = 150 carts/day. About 6/hour.
- Edits: 150 × 2 = 300 cart writes/day. ~0.003 writes/sec. *Negligible.*
- Reads: every page load shows cart count. Average 10 pages/visit × 500 = 5000 reads/day. ~0.06 reads/sec.
- Active carts at any time: ~50 (most are inactive for hours).
- Storage: 150 carts/day × 3 items × ~200 bytes = 90KB/day. After a year, ~33MB.

You could build this with one Postgres table, one app server, and walk away. A laptop running locally would handle the load.

**One million users:**

- Carts created: 1M × 30% = 300k/day. Sustained ~3.5/sec. Peak ~10/sec.
- Edits per cart: 2. So 600k cart writes/day. Sustained ~7/sec. Peak ~21/sec.
- Reads: 10M page loads/day × cart-count-on-every-page = 10M reads/day. Sustained ~115/sec. Peak ~350/sec.
- Active carts at any moment: 1M × 30% × (avg 2 hour active window / 24 hour day) ≈ 25k active carts at any moment.
- Storage at 30-day retention: 300k × 30 × 3 items × 200 bytes = ~5.5GB of cart_items. Plus carts header: 300k × 30 × ~150 bytes = ~1.3GB. Total ~7GB live. Easy.
- Inventory check QPS: every cart add and every cart-page-load re-checks stock. ~7/sec for add, ~350/sec for read. Peak inventory check load: ~400/sec.

**Key insights from the math:**

- Even at 1M users, write QPS is *tiny* (~20/sec peak). The cart is a low-write system.
- Reads dominate by 20x. Caching the read path is the whole game.
- Active carts at any moment (25k) is the metric that matters for memory sizing of a cache layer.
- The big cost is not the cart store. It is the *inventory check* fan-out to a downstream inventory service, which probably handles dozens of other read patterns. The cart becomes a noisy neighbor on inventory.
- Storage is trivial. 7GB fits on any modern Postgres. The system is small.

The architecture exists not for throughput but for **correctness around merges, inventory consistency, and price drift**.

</details>

## Step 3: where does the cart live?

This is the single most-debated design decision in cart systems. Four candidates: in-browser cookie only, in-memory server session, durable database, and the hybrid Redis+DB pattern. Each has a real trade-off.

Sketch a comparison table. For each storage option, name: who can see it, what happens on device change, what happens on server restart, what happens at high scale, and what it costs.

<details>
<summary><b>Reveal: comparison and recommended choice</b></summary>

| Storage | How it works | Pros | Cons | When to use |
|---------|--------------|------|------|-------------|
| **Cookie only** | Cart serialized into a cookie (JSON or signed token), sent on every request | Zero server state. Survives server restart. Works for anonymous users. | Limited to ~4KB cookie size (~30 items max). No cross-device sync. Cart visible to the user (so signing required to prevent tampering). Lost if browser cookies cleared. | Truly small sites, prototype only. |
| **In-memory server session** | Cart in app server memory keyed by session ID | Fastest possible reads. No external dependency. | Lost on server restart. Sticky sessions required. Does not scale past one server. | Never in production. Mentioned to be dismissed. |
| **Database (Postgres)** | Cart rows in a `carts` and `cart_items` table | Durable. Queryable for analytics ("abandoned carts > 24h"). ACID transactions. Survives anything. | One DB read per cart-icon load. At 350 reads/sec, the DB notices. Not as fast as memory. | Default choice for small-to-medium sites and as the source of truth for any scale. |
| **Redis cache + DB write-through** | Active cart in Redis hash per user. DB holds source-of-truth. Read from Redis, write to both. | Sub-ms cart reads. Scales reads horizontally. DB still has the durable record for analytics and recovery. | Two systems to keep in sync. Cache invalidation logic. More moving parts. | The right answer at 100k+ DAU or whenever DB reads start being noticeable. |

**The recommendation:**

- Anonymous users: cart stored server-side keyed by an anonymous cart token (UUID in a `cart_token` cookie). Not just in the cookie itself, because we want multi-device potential and bigger carts.
- Logged-in users: cart stored server-side keyed by user ID.
- Backend storage: start with Postgres only. Add Redis as a read-through cache when DB read load becomes noticeable (in practice, ~10k DAU and up).

**Why not cookie-only:**

Cookies are limited to ~4KB. They are sent on every request, so a fat cookie slows every page. They are visible to the user (which is fine, but means you must sign them to prevent tampering). They do not sync across devices. They get wiped when users clear cookies. The only thing they are good for is the anonymous cart *token* (a 36-byte UUID pointing at a server-side record).

**Why not in-memory session:**

This is the answer a junior engineer gives because it is fast and easy in local dev. It dies the moment you have two app servers and a load balancer. Mention it once, then move on.

</details>

## Step 4: incomplete architecture diagram

Here is an intentionally incomplete diagram. Fill in the five `[ ? ]` placeholders. Think about: who sits at the edge, where the cart actually lives, what speeds up reads, what the cart depends on, and what consumes its events.

```
                  Client (web, mobile)
                          │
                          ▼
                  ┌─────────────────┐
                  │    [ ? ]        │  (sits at edge, terminates TLS, auth)
                  └────────┬────────┘
                           │
                           ▼
                  ┌─────────────────┐
                  │  Cart Service   │  (stateless, horizontally scalable)
                  └──┬──────┬──┬────┘
                     │      │  │
              read   │      │  │ writes events
                     │      │  │
                     ▼      │  │
              ┌──────────┐  │  │
              │  [ ? ]   │  │  │  (fast read cache for active carts)
              └─────┬────┘  │  │
                    │       │  │
                    ▼       ▼  │
              ┌────────────────┐│
              │   [ ? ]        ││  (source of truth for cart data)
              └────────────────┘│
                                │
                                ▼
                         ┌────────────┐
                         │  [ ? ]     │  (event stream for downstream)
                         └────────────┘

           Sync calls out:
                  ┌────────────┐
                  │  [ ? ]     │  (tells the cart whether items are still in stock)
                  └────────────┘
```

<details>
<summary><b>Reveal: complete architecture</b></summary>

```
                  Client (web, mobile)
                          │
                          ▼
                  ┌──────────────────┐
                  │  API Gateway     │  (TLS, auth, rate limit,
                  │  + Auth          │   anonymous cart_token cookie
                  │                  │   handling)
                  └────────┬─────────┘
                           │
                           ▼
                  ┌──────────────────┐
                  │  Cart Service    │  (stateless, horizontally
                  │                  │   scalable; merge logic;
                  │                  │   price snapshot; size limits)
                  └──┬──────┬──┬─────┘
                     │      │  │
              read   │      │  │ writes events
                     │      │  │
                     ▼      │  │
              ┌──────────┐  │  │
              │  Redis   │  │  │  (cart per user: hash of items;
              │  (active │  │  │   TTL 30 days; primary read path)
              │  carts)  │  │  │
              └─────┬────┘  │  │
                    │       │  │
                    │write- │  │
                    │through│  │
                    ▼       ▼  │
              ┌────────────────┐│
              │  Postgres      ││  (source of truth: carts,
              │  (carts,       ││   cart_items, carts_merged
              │   cart_items)  ││   audit table)
              └────────────────┘│
                                │
                                ▼
                         ┌──────────────┐
                         │  Kafka       │  cart.item.added
                         │  cart.*      │  cart.item.removed
                         │              │  cart.merged
                         │              │  cart.abandoned
                         └──────┬───────┘
                                │
                  Consumers: abandoned-cart-notifier,
                             analytics, recommendation engine,
                             fraud detection

           Sync calls out:
                  ┌─────────────────┐
                  │ Inventory       │  (returns availability per SKU;
                  │ Service         │   cart calls on add and on read;
                  │                 │   own DB, own cache, own scaling)
                  └─────────────────┘
                  ┌─────────────────┐
                  │ Catalog /       │  (returns name, image, price;
                  │ Pricing Service │   cart joins this for display)
                  └─────────────────┘
```

Why each piece is here:

- **API Gateway + Auth.** First because the cart is exposed to the web. Handles TLS, rate limits abusive clients, and resolves the user (either authenticated user_id or anonymous cart_token from cookie).
- **Cart Service.** Stateless, scales horizontally. Holds the merge logic, size limits, and orchestrates inventory/pricing calls.
- **Redis (active carts).** The active read tier. Stored as a hash per user: `cart:{user_id}` → `{sku_a: 2, sku_b: 1}`. Sub-millisecond reads. TTL is the cart retention period.
- **Postgres (source of truth).** Durable. Used on Redis miss, for analytics queries (abandoned carts), and recovery if Redis dies.
- **Kafka.** The cart emits events but does not block on them. Downstream consumers handle abandoned-cart emails, analytics, recommendation training, and fraud signals.
- **Inventory Service.** Separately owned. The cart asks "is SKU X available in qty Y?" Cart never writes to inventory directly (that happens at checkout, in a different service). Inventory check is the bottleneck once you hit scale.
- **Catalog / Pricing Service.** Returns display data (name, image) and current price. The cart stores SKU + quantity + the snapshot price at add-time, and resolves the current price on read.

</details>

## Step 5: anonymous to logged-in cart merge

This is the trickiest part of cart design and the one most candidates flunk.

A guest user adds 3 items to their cart over 20 minutes. They then click "Log in" and authenticate. The system discovers this user *already has* a cart in their account from a session two days ago: 2 different items. What does the merged cart contain? What if there is overlap (same SKU in both carts at different quantities)?

Sketch the merge logic. Consider these cases:

- Anonymous cart has SKU A (qty 1), SKU B (qty 2). Account cart is empty.
- Anonymous cart has SKU A (qty 1). Account cart has SKU C (qty 1).
- Anonymous cart has SKU A (qty 2). Account cart has SKU A (qty 1). What is the final quantity?
- Anonymous cart has SKU A. Account cart has SKU A but A is now discontinued.
- Anonymous cart is empty. Account cart has SKU C. What shows up after login?

What do you do with the anonymous cart's `cart_token` after the merge?

<details>
<summary><b>Reveal: merge rules and the algorithm</b></summary>

**The merge policy (take a stance, defend it):**

For overlapping SKUs, the right default is **max(anonymous_qty, account_qty)**, not sum.

Why max and not sum: if a user added 2 of SKU A on their phone (anonymous), and they remembered they already had 1 in their account cart from yesterday, they almost certainly wanted "make sure I have 2", not "I want 3". Summing surprises users; max is conservative. The exception: items that are explicitly "additive" (digital downloads, gift cards) sum, but those are special-cased per category.

Some teams choose "anonymous wins" (overwrite account cart with the just-built session). Defensible: the user's last intent is most recent. Easier to explain. But you lose the older session's items if it had things the user still wanted.

Whichever you pick, write it down somewhere user-visible: "We combined your guest cart with your saved cart." Surface what changed.

**The full algorithm:**

```python
def merge_carts(anonymous_token, user_id):
    """Called on successful login when an anonymous cart_token cookie is present."""
    with db.transaction():
        anon_cart = cart_store.get_by_token(anonymous_token)
        user_cart = cart_store.get_by_user(user_id)

        if anon_cart is None:
            return user_cart                    # nothing to merge

        if user_cart is None:
            # Just rebind the anonymous cart to the user
            cart_store.rebind(anon_cart.id, user_id=user_id, clear_token=True)
            audit_merge(user_id, anon_cart.id, source="rebind")
            return cart_store.get_by_user(user_id)

        # Both exist. Merge.
        merged_items = {}
        for sku, item in user_cart.items.items():
            merged_items[sku] = item.copy()

        for sku, anon_item in anon_cart.items.items():
            if not catalog.is_available(sku):
                continue                        # skip discontinued items
            if sku in merged_items:
                merged_items[sku].qty = max(anon_item.qty, merged_items[sku].qty)
                merged_items[sku].qty = min(merged_items[sku].qty, MAX_QTY_PER_ITEM)
            else:
                merged_items[sku] = anon_item

        # Enforce cart size limit (100 distinct items)
        if len(merged_items) > MAX_CART_ITEMS:
            merged_items = trim_to_limit(merged_items, MAX_CART_ITEMS)

        cart_store.replace(user_cart.id, merged_items)
        cart_store.delete(anon_cart.id)         # anonymous cart consumed
        audit_merge(user_id, anon_cart.id, source="merge",
                    rules={"qty": "max", "trimmed": ...})

        return cart_store.get_by_user(user_id)
```

**Key implementation details:**

1. **Idempotency.** A user might double-click "Log in". The merge endpoint must be idempotent: if the anonymous cart has already been consumed (deleted), the second call is a no-op. Achieve this by checking `anon_cart is None` early; the cart_token is cleared from the cookie after first merge.

2. **Audit trail.** Insert a row into `carts_merged` capturing `(merge_id, user_id, anonymous_token, anonymous_items, account_items, merged_items, rule_applied, occurred_at)`. This rescues you from "my cart is wrong after I logged in" support tickets.

3. **Cookie cleanup.** After merge, clear the `cart_token` cookie (`Set-Cookie: cart_token=; Max-Age=0`). Otherwise the next page load may attempt to merge again (and find nothing, but generate a log line).

4. **Discontinued items.** Skip them silently in the merge, but surface a banner: "Some items in your guest cart are no longer available."

5. **Size limit.** A user with 80 items in their account cart and 80 in their guest cart can blow the 100-item limit. Trim with a deterministic rule (favor most-recently-added) and tell them.

6. **Race condition.** User logs in on two devices nearly simultaneously. Both call merge. Solve with a unique `cart_token` value: the first merge consumes the token (delete row), the second merge finds nothing to merge.

**Common mistake:** doing the merge client-side. The client cannot be trusted; it does not see the account cart history; it cannot enforce limits. The merge must be a single server-side operation in one transaction.

</details>

## Step 6: inventory consistency

The cart shows "in stock" when the user adds an item. Twenty minutes later, they click checkout. The item is gone. Or worse: the user successfully checks out, takes payment, and then your warehouse tells you there is no shoe to ship.

There are three approaches. None of them is perfect.

**A. No reservation (optimistic).** The cart shows the last-known inventory state. Checkout re-checks. If gone, surface "this item is no longer available" and require the user to adjust their cart.

**B. Soft reservation with TTL.** Adding to cart places a temporary hold on the inventory (15 minutes). If the user does not check out, the hold expires and the inventory is released. If they do check out, the hold is converted to a permanent commitment.

**C. No checks at all (overselling accepted).** Take the order. Charge the card. If the warehouse cannot fulfill, refund and apologize.

For each approach, walk through:
- What does the user experience look like in the common case?
- What goes wrong in the rare case?
- What does this cost engineering-wise?
- For what kind of business is each appropriate?

<details>
<summary><b>Reveal: the comparison and recommended approach</b></summary>

| Approach | Common case | Rare case | Engineering cost | Right for |
|----------|-------------|-----------|------------------|-----------|
| **Optimistic (re-check at checkout)** | Cart shows available; checkout succeeds. | Cart says available but item ran out between add and checkout. User sees "no longer available", has to remove and retry. ~1-3% of checkouts in normal commerce; higher on hot drops. | Low. No write to inventory at cart time. Just a read. | Most general-purpose e-commerce. Default choice. |
| **Soft reservation (TTL hold)** | Add to cart, item is reserved for 15 minutes. Checkout converts hold to commit. User never sees "out of stock" mid-checkout. | If user abandons, hold expires and other users see the item again. Inventory shows artificially low during the hold window. Hot products get worse: many holds, real users locked out. | High. Inventory service must support holds with TTLs. State synchronization. Holds must be released on cart deletion, cart-item removal, hold expiry, and checkout failure. | Live-event ticketing, limited drops (sneaker drops, concert tickets), B2B carts where checkout is a multi-step approval. |
| **No checks (oversell, refund later)** | Always succeeds at checkout. Warehouse fulfills or does not. | Customer gets an "out of stock, refund issued" email the next day. Brand damage, support cost, possibly chargebacks. | Almost zero from the cart's perspective. | Pre-orders. Print-on-demand. Anything with effectively infinite supply. |

**The recommendation: optimistic with TTL holds only for explicitly-marked SKUs.**

The default is optimistic. The cart reads inventory at add-time (shows green check) and again at cart-page-load (refreshes the badge: "in stock", "only 2 left", "out of stock"). At checkout, the order service does the authoritative re-check and atomic deduction.

For high-contention SKUs (limited drops, event tickets), the catalog marks the SKU as `requires_reservation=true`. The cart calls the inventory service to place a TTL hold; the hold token is stored in the cart_item row. If the user removes the item or the hold expires, the cart releases the hold. This adds engineering cost only where it earns its keep.

**Why not soft-reservation by default:**

Every cart abandonment becomes an artificial inventory shortage. Industry average is 60-70% abandonment. If every add-to-cart held inventory for 15 minutes, you would show "out of stock" to many real buyers while ghost carts sit on the inventory. This is the right behavior for a Taylor Swift concert (limited supply, high intent) and wrong for shoes (deep supply, casual browsing).

**Why not overselling by default:**

Refunds and "we cannot fulfill your order" emails destroy customer trust quickly. Only acceptable when supply is genuinely unlimited (digital goods, print-on-demand) or when over-commitments are expected as a business model (airline overselling with bumping policies).

**Where the check actually happens:**

- **On add-to-cart:** read-only check against the Inventory Service. Display state to user. No writes.
- **On cart load:** re-check (cached briefly, ~30s). User sees current state.
- **On checkout:** authoritative. The order service places an atomic `try_reserve(sku, qty)` against inventory. This is a *write*. If it fails, the order does not complete; user sees the failure.

The atomic reserve at checkout is the only consistency guarantee. Everything before it is best-effort display.

**Common mistake:** trying to make the cart "guarantee" the user gets the item. That guarantee belongs at checkout, not at the cart. The cart's job is to show good information; the order service's job is to commit.

</details>

## Follow-up questions

Try answering each in 2-3 sentences before reading the solution.

1. **Cart bloat from bots.** A scraper adds 10,000 items to a single anonymous cart to stress-test your endpoints. What goes wrong, and how do you prevent it?

2. **Multi-device sync delay.** User adds an item on their phone. Opens laptop 5 seconds later. Cart shows the old state. How long is acceptable? What is the mechanism?

3. **Redis dies mid-cart.** All active carts in Redis are lost. What does the user see? How do you recover without anyone noticing (much)?

4. **Price drift.** User added a shoe at $50 last week. Today it is $55. They click checkout. What price do they pay? What do they see?

5. **Cart abandonment detection.** You want to send "you left this in your cart" emails after 6 hours of inactivity. How do you detect this efficiently without scanning all carts every minute?

6. **Anonymous cart TTL.** Anonymous carts pile up forever. How and when do you garbage collect? What if a user comes back after 90 days with their old cart_token cookie still set?

7. **Cart shared with a partner.** Two people log into the same shared account from different cities at the same time. Both add items. How does the cart behave?

8. **Currency and locale.** User adds an item priced in USD. Switches their site language to EUR. What happens to the cart?

9. **Item becomes restricted.** User adds an item legal in their state. Later the item is restricted from shipping there (regulation change). They go to check out. What does your system do?

10. **Save for later.** User wants to "move to wishlist" from cart. Is this part of the cart service? Where does the wishlist live?

## Related problems

- **[Approval Management (011)](../011-approval-management/question.md)**. Different domain, same patterns: state per user, event stream on changes, audit trail of modifications. The cart's `carts_merged` audit table follows the same logic as approval's `audit_log`.
- **[Coupon Redemption (014)](../014-coupon-redemption/question.md)**. The cart holds the coupon code; the coupon service validates and applies discounts. The integration boundary is similar to inventory.
- **[Read-Heavy System Patterns (017)](../017-read-heavy-patterns/question.md)**. The cart-icon read on every page load is the read-heavy edge of this design. The Redis + DB tiering pattern there applies directly.
- **[Write-Heavy System Patterns (018)](../018-write-heavy-patterns/question.md)**. At scale, the cart-event Kafka topic and the abandoned-cart analytics consumer become a write-heavy stream. The patterns from that problem describe how to keep them durable.
- **[Help Desk Ticketing (019)](../019-helpdesk-ticketing/question.md)**. "My cart is wrong" support tickets are common; the `carts_merged` audit table is what helps the agent resolve them.
