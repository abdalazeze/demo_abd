# Interview Prep — Code Review Discussion

Personal notes. Not pushed to GitHub.

---

## How to approach the interview

The reviewer already read the code. They are not testing whether you wrote it — they want to see that you understand every decision and can defend it, spot its weaknesses, and describe how you'd improve it. Answer confidently but be honest about trade-offs. Never pretend something is perfect.

---

## Questions they will almost certainly ask

---

### "Walk me through the concurrency solution."

**What to say:**

The problem is a classic race condition: two requests both read `quantity = 1`, both pass the check, both decrement — one ends at `-1`.

The naive fix is `SELECT ... FOR UPDATE` inside a transaction. It works but it's an explicit lock — you need to manage acquiring it, and under high load it becomes a bottleneck.

I chose the atomic conditional UPDATE instead:

```sql
UPDATE stock
SET    quantity = quantity - ?
WHERE  product_id = ? AND warehouse_id = ? AND quantity >= ?;
```

The `WHERE quantity >= ?` predicate collapses the read, the check, and the write into a single atomic operation. The database engine serialises concurrent UPDATEs on the same row at the storage level — one wins (affected_rows = 1), one loses (affected_rows = 0). No explicit lock to acquire or release, works under any isolation level.

I wrap all lines in a `trans_begin / trans_commit / trans_rollback` block — not CI3's `trans_start/complete` — because I need to inspect `affected_rows()` between queries before deciding whether to commit.

**If they push back — "but SELECT FOR UPDATE is safer":**

Both are safe. `SELECT FOR UPDATE` makes the intent more readable — you can see you're locking. The atomic UPDATE is one fewer round-trip per line, no separate SELECT, and no lock management. For this scale either works. At very high concurrency, the atomic UPDATE is actually better because it holds the row lock for a shorter time.

---

### "Why CodeIgniter 3? It's legacy."

**What to say:**

It was specified in the brief. That said, I actually think it's a good choice for a test like this — it forces you to implement things yourself instead of reaching for a package. Auth from scratch, transactions explicitly managed, no ORM hiding the SQL. It makes the code easy to audit.

The main weakness of CI3 in 2026 is PHP 8.x compatibility — there are edge cases. We ran it on PHP 8.1 and it worked, but CI3 is not officially supported on 8.x.

---

### "Why did you roll your own auth instead of using Ion Auth?"

**What to say:**

Ion Auth adds around 20 files and a separate schema migration for what this app needs: login, logout, and a role check. Our auth is 50 lines:

- `attempt()` — `password_verify()` + set session
- `check()` — is user_id in session?
- `user()` — read session data back as an object
- `is_admin()` — role check

The only thing we lose is password reset emails, which the brief doesn't require. Adding a dependency that's bigger than the feature it provides is the wrong trade-off.

---

### "Why `is_active` instead of soft-delete?"

**What to say:**

Soft-delete (`deleted_at`) is the right answer for a production ERP because products referenced by historical invoices should be hidden from UI but still satisfy FK constraints and remain queryable. `is_active` is simpler — one column, `WHERE is_active = 1` everywhere.

The practical difference: if you disable a product today, its name still shows correctly on invoices created last month. Both `is_active` and `deleted_at` handle this — the FK row still exists either way. The difference is queryability and semantics. `deleted_at` is the cleaner model.

I chose `is_active` to keep the schema simple for this scope. In a real ERP I'd use `deleted_at`.

---

### "Why invoice-level discount instead of per-line?"

**What to say:**

The brief said "percentage discount" without specifying. Invoice-level is the most common interpretation in simple ERPs — one field, one calculation. Per-line discount needs an extra column on `invoice_lines`, a more complex UI, and per-line rounding decisions (do you round each line or the sum?).

If the business needs per-line negotiated pricing, that's a one-column schema change and a UI tweak. I noted it in DECISIONS.md as a next step.

---

### "How does the warehouse scoping work? Could a user_warehouse user see another warehouse's data?"

**What to say:**

There are two independent layers — either one alone would be enough, but both together makes it hard to break.

**Layer 1 — Controller:** `_scoped_warehouse_id()` in `MY_Controller` always returns the user's own `warehouse_id` for `user_warehouse` role, ignoring any GET/POST parameter. So even if someone crafts a request with `?warehouse_id=2`, the query uses their own warehouse ID.

**Layer 2 — Query:** Every stock and invoice query receives `$warehouse_id` and injects `WHERE warehouse_id = ?`. The value always comes from `_scoped_warehouse_id()`, never directly from user input.

There's also `_warehouse_guard()` for direct resource access — if a `user_warehouse` somehow gets the URL for another warehouse's resource, they get a 403.

**If they ask about the invoice pricing:**

Same pattern. The form shows the price input as `readonly` for `user_warehouse`. On the server, the posted value is completely ignored — we fetch `product.price` from the database directly. An attacker who removes the `readonly` attribute and posts a custom price gets the correct price anyway.

---

### "How does the invoice_no get generated? Why not a UUID or auto-increment?"

**What to say:**

We want a human-readable format: `INV-2026-000001`. The problem is we can't know the auto-increment ID before the INSERT.

The solution: INSERT with `uniqid('INV', true)` as a temporary placeholder (it's unique enough to not collide with the UNIQUE constraint), get `insert_id()`, compute `INV-YYYY-NNNNNN`, then UPDATE in the same transaction. Two queries, both inside the transaction, so if anything fails after the INSERT, the whole thing rolls back cleanly.

A UUID would be simpler but less readable on a printed invoice. A sequence table would add contention. This approach needs no extra infrastructure.

---

### "Why manual pagination instead of CI3's Pagination library?"

**What to say:**

CI3's Pagination library generates links based on URI segments — it wants URLs like `/products/page/2`. Our product list has search and category filters as GET params. Mixing URI segment pagination with GET params causes routing conflicts and messy URL construction.

Manual pagination with `?page=N` plus `http_build_query()` to preserve filters is three lines of code and produces clean, bookmarkable URLs. No library needed.

---

### "Why no Composer packages?"

**What to say:**

The brief said no external dependencies beyond what's necessary. There was nothing in the requirements that needed a package. Adding Composer for a CI3 project that doesn't need it adds noise and a `vendor/` directory that reviewers might question.

The one place I considered it was for a more robust input sanitiser, but CI3 has its own XSS filter and the risk surface here is small.

---

### "How would you test this properly?"

**What to say:**

Right now the only test is the concurrency script in `tests/concurrency_test.php` — it's a smoke test, not a suite. What I'd add:

1. **PHPUnit for the service layer** — `Invoice_service::create()` with a real test database. Cover: normal sale, insufficient stock, concurrent sales (using actual threads or process forking), discount rounding edge cases.
2. **Integration tests for the controllers** — test that `user_warehouse` can't see other warehouses' stock or invoices. These are the permission rules that are easy to break during a refactor.
3. **Contract tests for the pricing rule** — prove that posting a custom `unit_price` as `user_warehouse` always gets overridden server-side.

The concurrency test as written is useful but fragile — it depends on a specific product having exactly 1 unit, and the timing depends on the OS scheduler.

---

## What I would improve if I had more time

Be honest about these — the reviewer will respect it more than pretending the code is perfect.

### High priority

- **Audit log** — a `stock_movements` table recording every decrement (sale) and manual adjustment with user ID, timestamp, delta, and reason. Essential for any real ERP. Currently you can't tell why stock is 0.
- **Soft-delete** — replace `is_active` with `deleted_at` on products and categories. The FK history is intact either way, but `deleted_at` is the standard and allows `WHERE deleted_at IS NULL` across the board.
- **PHPUnit suite** — the service layer has real business logic (concurrency, pricing, discount rounding) that deserves proper tests.

### Medium priority

- **Invoice cancel / return** — add a `status` column (active/cancelled), on cancel restore stock inside a transaction using the same atomic pattern.
- **Stock transfer** — move qty between warehouses: decrement source, increment destination, both in one transaction.
- **VAT** — per-product tax rate, separate line on invoice total, stored on the invoice row at creation time (not recalculated — the rate might change later).

### Lower priority

- **Per-line discount** — extra column on `invoice_lines`, UI change, rounding decision.
- **Background CSV export** — for large datasets, queue the export and notify via email instead of streaming synchronously.
- **User management UI** — currently users are seed-only. A simple admin CRUD for users with warehouse assignment.
- **Password reset** — token-based, `password_resets` table with expiry column.

---

---

## Code walkthrough — module by module

Use these when the reviewer says "walk me through this file" or "explain how X works."

---

### Database schema

Seven tables, all InnoDB:

```
warehouses → users (FK)
categories → products (FK)
products   → stock (FK, composite unique on product_id + warehouse_id)
customers  → invoices (FK)
warehouses → invoices (FK)
users      → invoices (FK)
invoices   → invoice_lines (FK)
products   → invoice_lines (FK)
```

Key design choices worth knowing:

- **`stock` has a composite unique key** `(product_id, warehouse_id)`. One row per product-per-warehouse. This is what makes the atomic UPDATE safe — there's exactly one row to lock.
- **`invoices.invoice_no` is `UNIQUE VARCHAR(30)`**, not the primary key. The PK is an auto-increment INT. This lets us INSERT first, get the ID, then compute and UPDATE the formatted number.
- **`discount_amount` is stored alongside `discount_percent`**. Storing both means you never need to recalculate the discount if the percentage rule later changes. Historical invoices remain accurate.
- **`unit_price` is stored on `invoice_lines`**, not joined from `products.price`. Prices change — the line must record what was charged at sale time.
- **`users.warehouse_id` is nullable** — `admin` has no warehouse assignment, `user_warehouse` must have one. The schema doesn't enforce this with a constraint (that would need a check constraint on the ENUM + nullable combo), but the seed data and the UI enforce it in practice.
- **No `deleted_at` anywhere** — using `is_active` instead. Simple for this scope; in production you'd want `deleted_at` so you can query `WHERE deleted_at IS NULL` uniformly. Noted in DECISIONS.md.

---

### Auth system — `Auth_lib` + `Auth` controller

**`application/libraries/Auth_lib.php`**

```php
public function attempt($username, $password) { ... }   // login
public function logout()                       { ... }   // destroy session
public function check()                        { ... }   // is user_id in session?
public function user()                         { ... }   // return session data as stdClass
public function is_admin()                     { ... }   // role check shorthand
```

Four methods, 50 lines total. No library dependency — just CI3 session + `password_verify()`.

`attempt()` loads the user row by username, then `password_verify()` against `password_hash`. On success it writes `user_id`, `username`, `role`, and `warehouse_id` into session. **Why write `warehouse_id` into session?** So every subsequent request can scope queries without another DB hit. The session is the source of truth for the request lifetime.

`user()` reconstructs a `stdClass` from session data — it doesn't re-query the database. This means if an admin changes a user's role mid-session, the session still holds the old role until re-login. Acceptable trade-off for this scope.

**`application/controllers/Auth.php`**

Does not extend `MY_Controller` — it extends `CI_Controller` directly. If it extended `MY_Controller`, the auth check in `MY_Controller::__construct()` would redirect unauthenticated users away from the login page itself.

The login action: validate form → call `auth_lib->attempt()` → if true, redirect to products. The form validation check (`form_validation->run()`) short-circuits before `attempt()` on empty input, so we never hit the DB with blank credentials.

---

### Permission architecture — `MY_Controller`

**`application/core/MY_Controller.php`**

Every authenticated controller extends this. Three methods you'll be asked about:

**`_scoped_warehouse_id($method)`**

```php
protected function _scoped_warehouse_id($method = 'get')
{
    if ($this->user->role === 'user_warehouse') {
        return (int) $this->user->warehouse_id;  // always from session, never from request
    }
    return $method === 'post'
        ? (int) $this->input->post('warehouse_id')
        : (int) $this->input->get('warehouse_id');
}
```

For `user_warehouse`, the return value is **always** the session warehouse, regardless of what's in the request. An attacker can craft `?warehouse_id=2` all day — this method ignores it. For admin, it reads the request normally.

**`_warehouse_guard($warehouse_id)`**

For direct URL access — e.g. `/invoices/view/42` — we need to check if invoice 42 belongs to this user's warehouse. `_warehouse_guard()` receives the owner warehouse_id of the fetched resource and 403s if it doesn't match the user's warehouse.

**`Admin_Controller`**

A subclass of `MY_Controller` that adds a role check in its constructor. Any controller that extends `Admin_Controller` is admin-only. Used by Categories, Warehouses, Customers.

---

### Invoice system — the core feature

Three layers: controller → service → model.

**`application/controllers/Invoices.php` — `save()` method**

This is where the pricing enforcement happens, before the service is called:

```php
if ($is_admin) {
    $unit_price = round((float) ($line['unit_price'] ?? 0), 2);
    if ($unit_price <= 0) { continue; }
} else {
    // user_warehouse: force price from product table — ignore posted value
    $product    = $this->product_model->get($pid);
    $unit_price = $product ? round((float) $product->price, 2) : 0;
}
```

The `readonly` HTML attribute on the price input is cosmetic — it's trivially removed via browser dev tools. The server throws away the posted price for `user_warehouse` and fetches it from the database. The `is_admin` check uses the session role, not anything posted.

The `warehouse_id` for the invoice is taken from `_scoped_warehouse_id('post')` — same pattern: `user_warehouse` always gets their own, admin gets whatever they posted.

**`application/libraries/Invoice_service.php` — `create()` method**

The service handles everything inside a single transaction:

1. **Stock decrement** (the atomic UPDATE loop)
2. **Total calculation** — done server-side, `$subtotal` is computed from the cleaned `$lines` array, not from anything the client sent
3. **Invoice INSERT** with a `uniqid()` placeholder
4. **Invoice UPDATE** to set the formatted `invoice_no` after getting `insert_id()`
5. **Invoice lines INSERT** (one per line)
6. **`trans_commit()`**

If any stock decrement returns `affected_rows() === 0`, the method calls `trans_rollback()` and returns an `['error' => ...]` array immediately — no further inserts happen.

The client receives either `['invoice_id' => ..., 'invoice_no' => ...]` on success or `['error' => '...']` on failure. The controller checks for the `error` key and redirects accordingly.

**`application/models/Invoice_model.php`**

All read queries. `get_list()` and `get()` both accept an optional `$warehouse_id` — when provided they inject `WHERE i.warehouse_id = ?`. This is the second layer of scoping (the first being `_scoped_warehouse_id()` in the controller). If somehow the controller passed the wrong warehouse_id, the model query would return no rows.

**`assets/js/invoice.js` — the create form**

The JS handles: product autocomplete (debounced fetch to `/invoices/search_product`), adding rows to the lines table, live total calculation as quantities/prices change, and removing lines. The totals it shows are display-only — the server recomputes everything. `INVOICE_CFG` is inlined in the view to avoid hardcoding the base URL in JS.

---

### Stock management

**`application/models/Stock_model.php`**

Three methods that matter:

`get_low_stock()` — the `where('s.quantity <=', 'p.alert_quantity', false)` call: the `false` third argument tells CI3's query builder not to quote `p.alert_quantity` as a string. Without it, the SQL would be `WHERE s.quantity <= 'p.alert_quantity'` — comparing an integer to a literal string.

`adjust()` — upsert pattern: if a stock row exists, update it; if not, insert it. Uses `max(0, ...)` so manual adjustments can't push quantity below zero. This is a non-transactional convenience operation for admins only — no concurrency risk since it's not called during invoice creation.

**`application/controllers/Stock.php`**

`index()` — passes `$warehouse_id = 0` for admin (show all) or the session warehouse_id for `user_warehouse`. `get_list()` treats falsy as "no filter".

`adjust()` and `add_entry()` both call `_admin_only()` immediately. `add_entry()` is an admin shortcut: pick a product + warehouse, get redirected to the adjust form for that combination.

---

### Product model — pagination + search

**`application/models/Product_model.php`**

`_apply_filters()` is a private method called by both `get_list()` and `count_list()`. The key point: both the paginated query and the count query must apply the same filters, otherwise the page count is wrong. Extracting them into one method guarantees they stay in sync.

`search()` is the autocomplete endpoint — it returns only `id, code, name, price` and limits to 10 results. Keeping it minimal matters because it's called on every keystroke (debounced).

`code_exists()` — the `$exclude_id` parameter is used on edit: when updating a product, it should be allowed to keep its own code. Without excluding itself, the uniqueness check would always fail on edit.

`toggle_active()` — reads the current value then flips it. Two queries instead of one, but no raw SQL needed. The alternative would be `UPDATE products SET is_active = NOT is_active WHERE id = ?` — cleaner but bypasses the query builder.

---

### Reports — low stock + CSV export

**`application/controllers/Reports.php`**

`low_stock()` — applies both warehouse scoping and a search filter (product name or code). The query orders by `shortage DESC` — products furthest below their alert threshold appear first, which is the most actionable ordering.

`low_stock_csv()` — same query, different output:

```php
while (ob_get_level()) {
    ob_end_clean();
}
```

CI3 wraps everything in output buffers (sometimes nested). The `while` loop clears them all before we start streaming. If you used `ob_end_clean()` only once and there were two buffer levels, the outer buffer would still capture and delay the response.

The `\xEF\xBB\xBF` BOM at the top: Excel on Windows uses the BOM to detect UTF-8. Without it, Excel opens the file with Windows-1252 encoding and Arabic characters become garbage. Every other modern tool (Google Sheets, LibreOffice) handles UTF-8 correctly with or without the BOM.

Headers are also translated via `lang()` — so a CSV downloaded when Arabic is active has Arabic column headers.

---

### i18n system

**`application/helpers/app_helper.php`**

One function: `current_lang()`. Reads from session, defaults to `'ar'`. The default being Arabic is intentional — the business is Arabic-first.

**`application/controllers/Lang.php`**

```php
public function set($lang = 'en')
{
    if (in_array($lang, ['en', 'ar'], true)) {  // strict whitelist
        $this->session->set_userdata('lang', $lang);
    }
    $ref = $this->input->server('HTTP_REFERER') ?: base_url();
    redirect($ref);
}
```

The `in_array(..., true)` is strict type comparison — prevents values like `0` from matching. The referer redirect keeps the user on the same page after switching language. Fallback to `base_url()` handles direct URL access where there's no referer header.

**`MY_Controller::_load_language()`**

Loads the correct CI3 language file based on session lang. This runs on every request in every authenticated controller. The files are `application/language/english/ui_lang.php` and `application/language/arabic/ui_lang.php`.

**Layout**

```php
<html dir="<?= $this->session->userdata('lang') === 'ar' ? 'rtl' : 'ltr' ?>" lang="...">
```

The `dir` attribute switches the entire layout direction. Bootstrap's RTL stylesheet (`bootstrap.rtl.min.css`) is loaded conditionally — it mirrors margins, paddings, and flex directions for RTL. Without it, an RTL layout looks broken even if the text renders correctly.

---

## Things that are non-obvious in the code — be ready to explain

- **`trans_begin` vs `trans_start`** — `trans_start/complete` is CI3's convenience wrapper that automatically decides commit/rollback. We use `trans_begin/commit/rollback` explicitly because we need to check `affected_rows()` between queries to decide ourselves. If we used `trans_start`, we'd have no way to inspect the affected rows result mid-transaction.

- **`uniqid('INV', true)` in the INSERT** — the `true` second argument adds microseconds to make it more unique. It's a throwaway value — just needs to be unique enough to not collide with the UNIQUE constraint while we wait for `insert_id()`.

- **`while (ob_get_level()) ob_end_clean()`** in the CSV export — CI3 wraps output in its own buffer. Before we can stream a file directly to `php://output`, we need to flush all active buffers. The `while` loop handles nested buffers (not just one level).

- **UTF-8 BOM in CSV** — `\xEF\xBB\xBF` at the start of the file tells Excel it's UTF-8. Without it, Excel opens Arabic text with the wrong encoding and shows garbage characters. This is a known Excel quirk that has existed for 20 years.

- **`where($key, $val, false)` in Stock_model::get_low_stock** — the `false` third argument tells CI3's query builder not to escape the value as a string literal. Without it, `p.alert_quantity` would be quoted as `'p.alert_quantity'` and the comparison would be against a string, not the column value.

- **`language` in autoload helpers** — CI3's `lang()` function comes from `system/helpers/language_helper.php`. It's not loaded by default. Forgetting this causes a "Call to undefined function lang()" error in every view — which is exactly what happened when first running on WAMP.