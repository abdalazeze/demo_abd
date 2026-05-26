# Interview Prep — Code Review Discussion
# مراجعة المقابلة — نقاش مراجعة الكود

Personal notes. Not pushed to GitHub.
ملاحظات شخصية. غير مرفوعة على GitHub.

---

## How to approach the interview | كيف تتعامل مع المقابلة

The reviewer already read the code. They are not testing whether you wrote it — they want to see that you understand every decision and can defend it, spot its weaknesses, and describe how you'd improve it. Answer confidently but be honest about trade-offs. Never pretend something is perfect.

المراجع قرأ الكود مسبقاً. لا يختبر إن كنت كتبته — يريد أن يرى أنك تفهم كل قرار وتستطيع الدفاع عنه، وتحديد نقاط ضعفه، وشرح كيف ستحسّنه. أجب بثقة مع الصدق في الحديث عن المقايضات. لا تتظاهر أن الكود مثالي.

---

## Questions they will almost certainly ask | أسئلة ستُسأل على الأرجح

---

### "Walk me through the concurrency solution."
### "اشرح لي حل التزامن."

**What to say (EN):**

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

**ما تقوله (AR):**

- المشكلة race condition كلاسيكية: طلبان يقرآن `quantity = 1` في نفس الوقت، كلاهما يتجاوز الفحص، كلاهما يطرح — النتيجة `-1`.
- الحل الساذج هو `SELECT FOR UPDATE` داخل transaction، يعمل لكنه قفل صريح يصبح عنق زجاجة تحت الضغط العالي.
- اخترت UPDATE الشرطي الذري: شرط `WHERE quantity >= ?` يدمج القراءة والتحقق والكتابة في عملية واحدة.
- قاعدة البيانات تُسلسل العمليات المتزامنة على نفس الصف على المستوى التخزيني — طلب يفوز (`affected_rows = 1`)، الآخر يخسر (`affected_rows = 0`).
- استخدمت `trans_begin/commit/rollback` وليس `trans_start/complete` لأنني أحتاج فحص `affected_rows()` بين الاستعلامات قبل القرار بالإتمام أو الإلغاء.

**If they push back — "but SELECT FOR UPDATE is safer":**
**إذا اعترضوا — "لكن SELECT FOR UPDATE أكثر أماناً":**

Both are safe. `SELECT FOR UPDATE` makes the intent more readable — you can see you're locking. The atomic UPDATE is one fewer round-trip per line, no separate SELECT, and no lock management. For this scale either works. At very high concurrency, the atomic UPDATE is actually better because it holds the row lock for a shorter time.

كلاهما آمن. `SELECT FOR UPDATE` يجعل النية أكثر وضوحاً — ترى القفل صراحةً. UPDATE الذري أقل رحلة إلى قاعدة البيانات، بدون SELECT منفصل، بدون إدارة قفل. في التزامن العالي جداً، UPDATE الذري أفضل لأنه يحتفظ بقفل الصف لوقت أقصر.

---

### "Why InnoDB? Why not MyISAM?"
### "لماذا InnoDB؟ لماذا ليس MyISAM؟"

**What to say (EN):**

InnoDB and MySQL are not alternatives — InnoDB is a storage engine that runs *inside* MySQL (or MariaDB). The choice is InnoDB vs MyISAM, which was MySQL's old default before version 5.5.

Three reasons InnoDB is the only realistic choice for this app:

1. **Transactions** — `trans_begin/commit/rollback` only works with InnoDB. MyISAM has no transaction support at all. Without InnoDB, the entire concurrency solution falls apart — there is nothing to rollback.
2. **Row-level locking** — InnoDB locks individual rows during UPDATEs. MyISAM locks the entire table. For our atomic stock UPDATE, row-level locking is what allows two concurrent requests on different products to proceed without blocking each other.
3. **Foreign key constraints** — InnoDB enforces FK constraints at the database level. MyISAM parses them but silently ignores them — you'd get orphaned `invoice_lines` with no referential integrity.

InnoDB has been MySQL's default engine since 5.5 (2010). There is no reason to use MyISAM for a transactional application in 2026.

**ما تقوله (AR):**

- InnoDB وMySQL ليسا بديلين — InnoDB هو محرك تخزين يعمل *داخل* MySQL. المقارنة الحقيقية هي InnoDB مقابل MyISAM، الإعداد الافتراضي القديم قبل MySQL 5.5.
- ثلاثة أسباب تجعل InnoDB الاختيار الوحيد المنطقي:
  1. **Transactions** — `trans_begin/commit/rollback` يعمل فقط مع InnoDB. MyISAM لا يدعم الـ transactions أصلاً. بدون InnoDB، حل التزامن بالكامل ينهار — لا يوجد شيء للـ rollback.
  2. **Row-level locking** — InnoDB يقفل الصفوف الفردية أثناء UPDATE. MyISAM يقفل الجدول كله. لـ UPDATE المخزون الذري، قفل الصف يسمح لطلبين متزامنين على منتجات مختلفة بالعمل دون حجب بعضهما.
  3. **Foreign key constraints** — InnoDB يطبق قيود FK على مستوى قاعدة البيانات. MyISAM يحللها لكن يتجاهلها صامتاً — ستحصل على `invoice_lines` يتيمة بدون تكاملية مرجعية.
- InnoDB هو المحرك الافتراضي لـ MySQL منذ الإصدار 5.5 (2010). لا سبب لاستخدام MyISAM في تطبيق معاملات في 2026.

---

### "Why CodeIgniter 3? It's legacy."
### "لماذا CodeIgniter 3؟ إنه إطار قديم."

**What to say (EN):**

It was specified in the brief. That said, I actually think it's a good choice for a test like this — it forces you to implement things yourself instead of reaching for a package. Auth from scratch, transactions explicitly managed, no ORM hiding the SQL. It makes the code easy to audit.

The main weakness of CI3 in 2026 is PHP 8.x compatibility — there are edge cases. We ran it on PHP 8.1 and it worked, but CI3 is not officially supported on 8.x.

**ما تقوله (AR):**

- كان محدداً في المهمة.
- في رأيي هو اختيار جيد لاختبار كهذا — يجبرك على تطبيق الأشياء بنفسك: authentication من الصفر، transactions بشكل صريح، بدون ORM يخفي SQL.
- الكود سهل المراجعة والتدقيق.
- نقطة الضعف الرئيسية في 2026 هي التوافق مع PHP 8.x — توجد حالات حافة. شغّلناه على PHP 8.1 وعمل، لكنه غير مدعوم رسمياً على 8.x.

---

### "Why did you roll your own auth instead of using Ion Auth?"
### "لماذا كتبت نظام المصادقة بنفسك بدلاً من استخدام Ion Auth؟"

**What to say (EN):**

Ion Auth adds around 20 files and a separate schema migration for what this app needs: login, logout, and a role check. Our auth is 50 lines:

- `attempt()` — `password_verify()` + set session
- `check()` — is user_id in session?
- `user()` — read session data back as an object
- `is_admin()` — role check

The only thing we lose is password reset emails, which the brief doesn't require. Adding a dependency that's bigger than the feature it provides is the wrong trade-off.

**ما تقوله (AR):**

- Ion Auth يضيف ~20 ملفاً و migration منفصل لميزة نحتاج فيها فقط: تسجيل دخول، خروج، وفحص دور.
- الـ auth عندنا 50 سطراً: `attempt()` للدخول، `check()` للتحقق من الجلسة، `user()` لقراءة البيانات، `is_admin()` لفحص الدور.
- الشيء الوحيد الذي نخسره هو إعادة تعيين كلمة المرور عبر البريد، وهو غير مطلوب في المهمة.
- إضافة اعتمادية أكبر من الميزة التي تقدمها هو مقايضة خاطئة.

---

### "Why `is_active` instead of soft-delete?"
### "لماذا استخدمت `is_active` بدلاً من الحذف الناعم (soft-delete)؟"

**What to say (EN):**

Soft-delete (`deleted_at`) is the right answer for a production ERP because products referenced by historical invoices should be hidden from UI but still satisfy FK constraints and remain queryable. `is_active` is simpler — one column, `WHERE is_active = 1` everywhere.

The practical difference: if you disable a product today, its name still shows correctly on invoices created last month. Both `is_active` and `deleted_at` handle this — the FK row still exists either way. The difference is queryability and semantics. `deleted_at` is the cleaner model.

I chose `is_active` to keep the schema simple for this scope. In a real ERP I'd use `deleted_at`.

**ما تقوله (AR):**

- `deleted_at` هو الإجابة الصحيحة في بيئة إنتاج لأنه يسمح بـ `WHERE deleted_at IS NULL` بشكل موحد ويعطي دلالة أوضح.
- `is_active` أبسط — عمود واحد، `WHERE is_active = 1` في كل مكان.
- الفرق العملي: إذا عطّلت منتجاً اليوم، اسمه لا يزال يظهر بشكل صحيح على الفواتير القديمة — كلا النهجين يحققان ذلك لأن صف FK لا يزال موجوداً.
- اخترت `is_active` لبساطة النطاق. في ERP حقيقي كنت سأستخدم `deleted_at`.

---

### "Why invoice-level discount instead of per-line?"
### "لماذا الخصم على مستوى الفاتورة بدلاً من كل سطر على حدة؟"

**What to say (EN):**

The brief said "percentage discount" without specifying. Invoice-level is the most common interpretation in simple ERPs — one field, one calculation. Per-line discount needs an extra column on `invoice_lines`, a more complex UI, and per-line rounding decisions (do you round each line or the sum?).

If the business needs per-line negotiated pricing, that's a one-column schema change and a UI tweak. I noted it in DECISIONS.md as a next step.

**ما تقوله (AR):**

- المهمة ذكرت "خصم نسبة مئوية" دون تحديد المستوى.
- الخصم على مستوى الفاتورة هو التفسير الأكثر شيوعاً في ERPs البسيطة — حقل واحد، حساب واحد.
- الخصم لكل سطر يحتاج عموداً إضافياً في `invoice_lines`، واجهة أكثر تعقيداً، وقرارات تقريب لكل سطر.
- إذا احتاج العمل خصماً لكل سطر، هذا تغيير عمود واحد في الـ schema وتعديل بسيط في الواجهة. ذكرته في DECISIONS.md كخطوة تالية.

---

### "How does the warehouse scoping work? Could a user_warehouse user see another warehouse's data?"
### "كيف يعمل تحديد نطاق المستودع؟ هل يمكن لمستخدم المستودع رؤية بيانات مستودع آخر؟"

**What to say (EN):**

There are two independent layers — either one alone would be enough, but both together makes it hard to break.

**Layer 1 — Controller:** `_scoped_warehouse_id()` in `MY_Controller` always returns the user's own `warehouse_id` for `user_warehouse` role, ignoring any GET/POST parameter. So even if someone crafts a request with `?warehouse_id=2`, the query uses their own warehouse ID.

**Layer 2 — Query:** Every stock and invoice query receives `$warehouse_id` and injects `WHERE warehouse_id = ?`. The value always comes from `_scoped_warehouse_id()`, never directly from user input.

There's also `_warehouse_guard()` for direct resource access — if a `user_warehouse` somehow gets the URL for another warehouse's resource, they get a 403.

**If they ask about the invoice pricing:**

Same pattern. The form shows the price input as `readonly` for `user_warehouse`. On the server, the posted value is completely ignored — we fetch `product.price` from the database directly. An attacker who removes the `readonly` attribute and posts a custom price gets the correct price anyway.

**ما تقوله (AR):**

- طبقتان مستقلتان — كل واحدة كافية وحدها، لكن معاً يصعب كسرهما.
- **الطبقة الأولى — Controller:** `_scoped_warehouse_id()` في `MY_Controller` يُرجع دائماً `warehouse_id` الخاص بالمستخدم لدور `user_warehouse`، ويتجاهل أي معامل GET أو POST. حتى لو صنع المهاجم طلباً بـ `?warehouse_id=2`، الاستعلام يستخدم مستودعه هو.
- **الطبقة الثانية — Query:** كل استعلام stock وinvoice يستقبل `$warehouse_id` ويُضيف `WHERE warehouse_id = ?`، والقيمة دائماً من `_scoped_warehouse_id()`، وليس مباشرة من المدخلات.
- `_warehouse_guard()` للوصول المباشر للرابط — إذا وصل `user_warehouse` لرابط مورد مستودع آخر، يحصل على 403.
- نفس النمط للتسعير: حقل السعر `readonly` في الواجهة فقط للزينة. في السيرفر، القيمة المُرسلة تُتجاهل تماماً ونجلب `product.price` من قاعدة البيانات مباشرة.

---

### "How does the invoice_no get generated? Why not a UUID or auto-increment?"
### "كيف يتم توليد رقم الفاتورة؟ لماذا لم تستخدم UUID أو auto-increment مباشرة؟"

**What to say (EN):**

We want a human-readable format: `INV-2026-000001`. The problem is we can't know the auto-increment ID before the INSERT.

The solution: INSERT with `uniqid('INV', true)` as a temporary placeholder (it's unique enough to not collide with the UNIQUE constraint), get `insert_id()`, compute `INV-YYYY-NNNNNN`, then UPDATE in the same transaction. Two queries, both inside the transaction, so if anything fails after the INSERT, the whole thing rolls back cleanly.

A UUID would be simpler but less readable on a printed invoice. A sequence table would add contention. This approach needs no extra infrastructure.

**ما تقوله (AR):**

- نريد تنسيقاً مقروءاً للإنسان: `INV-2026-000001`. المشكلة أننا لا نعرف الـ ID التلقائي قبل INSERT.
- الحل: INSERT بـ `uniqid('INV', true)` كـ placeholder مؤقت (فريد بما يكفي لعدم التصادم مع UNIQUE constraint)، ثم نأخذ `insert_id()`، نحسب `INV-YYYY-NNNNNN`، ثم UPDATE — كل ذلك داخل نفس الـ transaction.
- UUID أبسط لكن أقل قابلية للقراءة على الفاتورة المطبوعة.
- جدول sequence يضيف تنافساً غير ضروري.
- هذا النهج لا يحتاج بنية تحتية إضافية.

---

### "Why manual pagination instead of CI3's Pagination library?"
### "لماذا تصفح الصفحات يدوياً بدلاً من استخدام مكتبة CI3 للتصفح؟"

**What to say (EN):**

CI3's Pagination library generates links based on URI segments — it wants URLs like `/products/page/2`. Our product list has search and category filters as GET params. Mixing URI segment pagination with GET params causes routing conflicts and messy URL construction.

Manual pagination with `?page=N` plus `http_build_query()` to preserve filters is three lines of code and produces clean, bookmarkable URLs. No library needed.

**ما تقوله (AR):**

- مكتبة Pagination في CI3 تولد روابط بناءً على URI segments — تريد URLs مثل `/products/page/2`.
- قائمة المنتجات لديها فلاتر بحث وفئة كـ GET params. دمج التصفح القائم على URI segments مع GET params يسبب تعارضات في التوجيه وبناء URLs فوضوي.
- التصفح اليدوي بـ `?page=N` مع `http_build_query()` للحفاظ على الفلاتر هو ثلاثة أسطر من الكود وينتج URLs نظيفة قابلة للحفظ كـ bookmark.

---

### "Why no Composer packages?"
### "لماذا لم تستخدم حزم Composer؟"

**What to say (EN):**

The brief said no external dependencies beyond what's necessary. There was nothing in the requirements that needed a package. Adding Composer for a CI3 project that doesn't need it adds noise and a `vendor/` directory that reviewers might question.

The one place I considered it was for a more robust input sanitiser, but CI3 has its own XSS filter and the risk surface here is small.

**ما تقوله (AR):**

- المهمة نصّت على عدم الاعتماديات الخارجية غير الضرورية. لم يكن في المتطلبات ما يستدعي حزمة.
- إضافة Composer لمشروع CI3 لا يحتاجه يضيف ضوضاء ومجلد `vendor/` قد يتساءل عنه المراجع.
- المكان الوحيد الذي فكرت فيه باستخدام حزمة كان لـ input sanitiser أقوى، لكن CI3 لديه فلتر XSS خاص به وحجم المخاطر هنا صغير.

---

### "How would you test this properly?"
### "كيف ستختبر هذا المشروع بشكل صحيح؟"

**What to say (EN):**

Right now the only test is the concurrency script in `tests/concurrency_test.php` — it's a smoke test, not a suite. What I'd add:

1. **PHPUnit for the service layer** — `Invoice_service::create()` with a real test database. Cover: normal sale, insufficient stock, concurrent sales (using actual threads or process forking), discount rounding edge cases.
2. **Integration tests for the controllers** — test that `user_warehouse` can't see other warehouses' stock or invoices. These are the permission rules that are easy to break during a refactor.
3. **Contract tests for the pricing rule** — prove that posting a custom `unit_price` as `user_warehouse` always gets overridden server-side.

The concurrency test as written is useful but fragile — it depends on a specific product having exactly 1 unit, and the timing depends on the OS scheduler.

**ما تقوله (AR):**

- حالياً الاختبار الوحيد هو سكريبت التزامن في `tests/concurrency_test.php` — هو smoke test وليس suite كاملة.
- ما سأضيفه:
  1. **PHPUnit لطبقة الخدمة** — `Invoice_service::create()` مع قاعدة بيانات اختبار حقيقية: بيع عادي، مخزون غير كافٍ، بيع متزامن، حالات تقريب الخصم.
  2. **Integration tests للـ controllers** — اختبار أن `user_warehouse` لا يرى مخزون أو فواتير مستودع آخر. هذه قواعد الصلاحيات التي يسهل كسرها أثناء إعادة الهيكلة.
  3. **Contract tests لقاعدة التسعير** — إثبات أن إرسال `unit_price` مخصص كـ `user_warehouse` يُتجاهل دائماً من السيرفر.
- اختبار التزامن الحالي مفيد لكنه هشّ — يعتمد على منتج بعينه له وحدة واحدة بالضبط، والتوقيت يعتمد على جدولة نظام التشغيل.

---

## What I would improve if I had more time | ما الذي سأحسّنه لو كان لديّ وقت أكثر

Be honest about these — the reviewer will respect it more than pretending the code is perfect.
كن صادقاً في هذه — المراجع سيحترمك أكثر من التظاهر بأن الكود مثالي.

### High priority | أولوية عالية

- **Audit log | سجل التدقيق** — a `stock_movements` table recording every decrement (sale) and manual adjustment with user ID, timestamp, delta, and reason. Essential for any real ERP. Currently you can't tell why stock is 0. / جدول `stock_movements` يسجل كل تخفيض (بيع) وتعديل يدوي مع معرف المستخدم والوقت والفرق والسبب. ضروري لأي ERP حقيقي. حالياً لا تستطيع معرفة لماذا المخزون صفر.
- **Soft-delete | الحذف الناعم** — replace `is_active` with `deleted_at` on products and categories. / استبدال `is_active` بـ `deleted_at` على المنتجات والفئات.
- **PHPUnit suite** — the service layer has real business logic (concurrency, pricing, discount rounding) that deserves proper tests. / طبقة الخدمة تحتوي على منطق أعمال حقيقي يستحق اختبارات صحيحة.

### Medium priority | أولوية متوسطة

- **Invoice cancel / return | إلغاء/إرجاع الفاتورة** — add a `status` column (active/cancelled), on cancel restore stock inside a transaction using the same atomic pattern. / إضافة عمود `status`، عند الإلغاء يُعاد المخزون داخل transaction بنفس النمط الذري.
- **Stock transfer | نقل المخزون** — move qty between warehouses: decrement source, increment destination, both in one transaction. / تحريك الكمية بين المستودعات: تخفيض المصدر، زيادة الوجهة، كلاهما في transaction واحدة.
- **VAT | ضريبة القيمة المضافة** — per-product tax rate, stored on the invoice row at creation time. / معدل ضريبة لكل منتج، يُخزن على صف الفاتورة وقت الإنشاء لا يُعاد حسابه لاحقاً.

### Lower priority | أولوية منخفضة

- **Per-line discount | خصم لكل سطر** — extra column on `invoice_lines`, UI change, rounding decision.
- **Background CSV export | تصدير CSV في الخلفية** — for large datasets, queue the export and notify via email.
- **User management UI | واجهة إدارة المستخدمين** — currently users are seed-only. A simple admin CRUD for users with warehouse assignment.
- **Password reset | إعادة تعيين كلمة المرور** — token-based, `password_resets` table with expiry column.

---

## Code walkthrough — module by module
## مراجعة الكود — وحدة بوحدة

Use these when the reviewer says "walk me through this file" or "explain how X works."
استخدم هذه عندما يقول المراجع "اشرح لي هذا الملف" أو "كيف يعمل X؟"

---

### Database schema
### هيكل قاعدة البيانات

Seven tables, all InnoDB — سبعة جداول، كلها InnoDB:

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

Key design choices | قرارات التصميم الرئيسية:

- **`stock` has a composite unique key** `(product_id, warehouse_id)`. One row per product-per-warehouse. This is what makes the atomic UPDATE safe — there's exactly one row to lock. / مفتاح فريد مركب — صف واحد لكل منتج لكل مستودع. هذا ما يجعل UPDATE الذري آمناً — يوجد صف واحد بالضبط للقفل عليه.
- **`invoices.invoice_no` is `UNIQUE VARCHAR(30)`**, not the primary key. The PK is auto-increment INT. This lets us INSERT first, get the ID, then compute and UPDATE the formatted number. / ليس المفتاح الأساسي. PK هو INT auto-increment يسمح لنا بالإدراج أولاً، أخذ الـ ID، ثم حساب وتحديث الرقم المنسق.
- **`discount_amount` is stored alongside `discount_percent`**. Storing both means historical invoices remain accurate if the rule changes. / تخزين كلاهما يعني أن الفواتير التاريخية تبقى دقيقة إذا تغيرت قاعدة الخصم.
- **`unit_price` is stored on `invoice_lines`**, not joined from `products.price`. Prices change — the line records what was charged at sale time. / الأسعار تتغير — السطر يسجل ما تم تحصيله وقت البيع.
- **`users.warehouse_id` is nullable** — `admin` has no warehouse, `user_warehouse` must have one. / `admin` ليس لديه مستودع، `user_warehouse` يجب أن يكون له واحد.
- **No `deleted_at`** — using `is_active` for simplicity. In production would use `deleted_at`. / استخدام `is_active` للبساطة. في الإنتاج كنت سأستخدم `deleted_at`.

---

### Auth system — `Auth_lib` + `Auth` controller
### نظام المصادقة

**`application/libraries/Auth_lib.php`**

```php
public function attempt($username, $password) { ... }   // login | تسجيل الدخول
public function logout()                       { ... }   // destroy session | تدمير الجلسة
public function check()                        { ... }   // is user_id in session? | هل user_id في الجلسة؟
public function user()                         { ... }   // return session data as stdClass
public function is_admin()                     { ... }   // role check shorthand | فحص الدور
```

Four methods, 50 lines total. No library dependency — just CI3 session + `password_verify()`.
أربعة دوال، 50 سطراً. لا اعتمادية على مكتبة — فقط CI3 session + `password_verify()`.

`attempt()` loads the user row by username, then `password_verify()` against `password_hash`. On success it writes `user_id`, `username`, `role`, and `warehouse_id` into session. **Why write `warehouse_id` into session?** So every subsequent request can scope queries without another DB hit.

`attempt()` يحمّل صف المستخدم بالاسم ثم `password_verify()` على `password_hash`. عند النجاح يكتب في الجلسة. **لماذا نكتب `warehouse_id` في الجلسة؟** حتى يستطيع كل طلب تالٍ تحديد نطاق الاستعلامات بدون رحلة إضافية لقاعدة البيانات.

`user()` reconstructs a `stdClass` from session data — doesn't re-query the database. If an admin changes a user's role mid-session, the session still holds the old role until re-login. Acceptable trade-off for this scope.

`user()` يُعيد بناء `stdClass` من بيانات الجلسة — لا يعيد الاستعلام من قاعدة البيانات. مقايضة مقبولة لهذا النطاق.

**`application/controllers/Auth.php`**

Does not extend `MY_Controller` — extends `CI_Controller` directly. If it extended `MY_Controller`, the auth check in `MY_Controller::__construct()` would redirect unauthenticated users away from the login page itself.

لا يمتد من `MY_Controller` — يمتد من `CI_Controller` مباشرة. لو امتد من `MY_Controller`، فحص المصادقة في `__construct()` كان سيحوّل المستخدمين غير المسجلين بعيداً عن صفحة تسجيل الدخول نفسها.

---

### Permission architecture — `MY_Controller`
### بنية الصلاحيات

**`application/core/MY_Controller.php`**

Every authenticated controller extends this. Three methods you'll be asked about:
كل controller مصادق عليه يمتد من هذا. ثلاثة دوال ستُسأل عنها:

**`_scoped_warehouse_id($method)` — تحديد نطاق المستودع**

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

For `user_warehouse`, return value is **always** the session warehouse regardless of what's in the request. An attacker can craft `?warehouse_id=2` all day — this method ignores it.

لـ `user_warehouse`، القيمة المُرجعة **دائماً** مستودع الجلسة بغض النظر عن الطلب. المهاجم يمكنه صياغة `?warehouse_id=2` طوال اليوم — هذه الدالة تتجاهله.

**`_warehouse_guard($warehouse_id)` — حارس المستودع**

For direct URL access — e.g. `/invoices/view/42` — checks if the resource belongs to this user's warehouse. 403 if not.
للوصول المباشر للرابط — يفحص إن كان المورد ينتمي لمستودع هذا المستخدم. 403 إذا لم يكن كذلك.

**`Admin_Controller` — Controller للمسؤول فقط**

A subclass of `MY_Controller` that adds a role check in its constructor. Used by Categories, Warehouses, Customers.
فئة فرعية من `MY_Controller` تضيف فحص الدور في المُنشئ. تستخدمها: Categories، Warehouses، Customers.

---

### Invoice system — the core feature
### نظام الفواتير — الميزة الأساسية

Three layers: controller → service → model.
ثلاث طبقات: controller → service → model.

**`application/controllers/Invoices.php` — `save()` method**

This is where the pricing enforcement happens, before the service is called:
هنا يتم تطبيق قاعدة التسعير، قبل استدعاء الخدمة:

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

The `readonly` HTML attribute on the price input is cosmetic — trivially removed via browser dev tools. The server throws away the posted price for `user_warehouse` and fetches from the database. The `is_admin` check uses the session role, not anything posted.

خاصية `readonly` في HTML مجرد زينة — يمكن إزالتها بسهولة من أدوات المطور. السيرفر يتجاهل السعر المُرسل لـ `user_warehouse` ويجلبه من قاعدة البيانات. فحص `is_admin` يستخدم دور الجلسة وليس أي شيء مُرسل.

**`application/libraries/Invoice_service.php` — `create()` method**

The service handles everything inside a single transaction:
الخدمة تتعامل مع كل شيء داخل transaction واحدة:

1. **Stock decrement** — the atomic UPDATE loop | **تخفيض المخزون** — حلقة UPDATE الذري
2. **Total calculation** — server-side, from cleaned `$lines` array | **حساب الإجماليات** — في السيرفر، من مصفوفة `$lines` المنظفة
3. **Invoice INSERT** with `uniqid()` placeholder | **إدراج الفاتورة** بـ placeholder مؤقت
4. **Invoice UPDATE** to set formatted `invoice_no` | **تحديث الفاتورة** لضبط رقم الفاتورة المنسق
5. **Invoice lines INSERT** (one per line) | **إدراج أسطر الفاتورة** (سطر واحد لكل منتج)
6. **`trans_commit()`**

If any stock decrement returns `affected_rows() === 0`, the method calls `trans_rollback()` and returns `['error' => ...]` immediately.
إذا أعاد أي تخفيض مخزون `affected_rows() === 0`، الدالة تستدعي `trans_rollback()` وتُرجع `['error' => ...]` فوراً.

**`application/models/Invoice_model.php`**

All read queries. `get_list()` and `get()` accept optional `$warehouse_id` — when provided inject `WHERE i.warehouse_id = ?`. Second layer of scoping — if the controller somehow passed the wrong warehouse_id, the model query returns no rows.

كل الاستعلامات للقراءة. الطبقة الثانية من تحديد النطاق — إذا مرّر الـ controller بطريقة ما `warehouse_id` خاطئاً، الاستعلام يُرجع صفراً من النتائج.

---

### Stock management
### إدارة المخزون

**`application/models/Stock_model.php`**

`get_low_stock()` — `where('s.quantity <=', 'p.alert_quantity', false)`: the `false` third argument tells CI3's query builder not to quote `p.alert_quantity` as a string. Without it: `WHERE s.quantity <= 'p.alert_quantity'` — comparing an integer to a string literal.

المعامل الثالث `false` يخبر query builder في CI3 بعدم وضع `p.alert_quantity` بين علامات اقتباس كنص. بدونه: `WHERE s.quantity <= 'p.alert_quantity'` — مقارنة عدد صحيح بنص حرفي.

`adjust()` — upsert pattern: if stock row exists update it, if not insert it. Uses `max(0, ...)` so manual adjustments can't push quantity below zero. Non-transactional, admin-only.

نمط upsert: إذا وُجد صف المخزون يُحدّثه، وإلا يُدرجه. `max(0, ...)` يمنع الكمية من النزول تحت الصفر. غير transaction، للمسؤول فقط.

**`application/controllers/Stock.php`**

`index()` — passes falsy `$warehouse_id` for admin (show all). `get_list()` treats falsy as "no filter".
يمرر `warehouse_id` بقيمة falsy للمسؤول (عرض الكل). `get_list()` يعامل القيمة الـ falsy كـ "بدون فلتر".

`adjust()` and `add_entry()` both call `_admin_only()` immediately.
`adjust()` و`add_entry()` كلاهما يستدعيان `_admin_only()` فوراً.

---

### Product model — pagination + search
### موديل المنتجات — تصفح الصفحات + البحث

**`application/models/Product_model.php`**

`_apply_filters()` — private method called by both `get_list()` and `count_list()`. Both the paginated query and the count query must apply the same filters, otherwise the page count is wrong. Extracting into one method guarantees they stay in sync.

دالة خاصة تستدعيها كل من `get_list()` و`count_list()`. كلا استعلام البيانات واستعلام العدد يجب أن يطبقا نفس الفلاتر، وإلا عدد الصفحات سيكون خاطئاً. استخراجها في دالة واحدة يضمن بقاءهما متزامنين.

`search()` — autocomplete endpoint, returns only `id, code, name, price`, limit 10. Keeping it minimal matters because it's called on every keystroke (debounced).

نقطة نهاية الإكمال التلقائي، تُرجع `id, code, name, price` فقط، حد 10 نتائج. الحفاظ على الحد الأدنى مهم لأنها تُستدعى عند كل ضغطة مفتاح (مع debouncing).

`code_exists($exclude_id)` — the `$exclude_id` parameter is used on edit: a product should be allowed to keep its own code. Without excluding itself, the uniqueness check would always fail on edit.

معامل `$exclude_id` يُستخدم عند التعديل: المنتج يجب أن يُسمح له بالاحتفاظ بكوده. بدون استبعاد نفسه، فحص التفرد سيفشل دائماً عند التعديل.

---

### Reports — low stock + CSV export
### التقارير — المخزون المنخفض + تصدير CSV

**`application/controllers/Reports.php`**

`low_stock()` — orders by `shortage DESC` — products furthest below their alert threshold appear first, most actionable ordering.

يرتب بـ `shortage DESC` — المنتجات الأبعد عن حد التنبيه تظهر أولاً، وهو الترتيب الأكثر قابلية للتنفيذ.

`low_stock_csv()`:

```php
while (ob_get_level()) {
    ob_end_clean();
}
```

CI3 wraps everything in output buffers (sometimes nested). The `while` loop clears them all before streaming. One `ob_end_clean()` alone might leave an outer buffer that captures and delays the response.

CI3 يلف كل شيء في output buffers (أحياناً متداخلة). حلقة `while` تمسحها كلها قبل البث. استخدام `ob_end_clean()` مرة واحدة قد يترك buffer خارجي يلتقط الاستجابة ويؤخرها.

The `\xEF\xBB\xBF` BOM: Excel on Windows uses the BOM to detect UTF-8. Without it, Excel opens Arabic text with Windows-1252 encoding and shows garbage. This Excel quirk has existed for 20 years.

Excel على Windows يستخدم BOM لاكتشاف UTF-8. بدونه، Excel يفتح النص العربي بترميز Windows-1252 ويظهر أحرفاً غير مفهومة. هذا خطأ Excel موجود منذ 20 سنة.

---

### i18n system
### نظام التعريب

**`application/helpers/app_helper.php`**

One function: `current_lang()`. Reads from session, defaults to `'ar'`. The default being Arabic is intentional — the business is Arabic-first.

دالة واحدة: `current_lang()`. تقرأ من الجلسة، الافتراضي `'ar'`. الافتراضي عربي مقصود — العمل أولوية عربية.

**`application/controllers/Lang.php`**

```php
if (in_array($lang, ['en', 'ar'], true)) {  // strict whitelist | قائمة بيضاء صارمة
```

`in_array(..., true)` is strict type comparison — prevents values like `0` from matching. The referer redirect keeps the user on the same page after switching language.

مقارنة نوع صارمة — تمنع قيماً مثل `0` من المطابقة. إعادة التوجيه عبر المرجع تُبقي المستخدم في نفس الصفحة بعد تغيير اللغة.

**Layout | التخطيط**

```php
<html dir="<?= $this->session->userdata('lang') === 'ar' ? 'rtl' : 'ltr' ?>" lang="...">
```

The `dir` attribute switches the entire layout direction. Bootstrap's RTL stylesheet (`bootstrap.rtl.min.css`) is loaded conditionally — it mirrors margins, paddings, and flex directions for RTL. Without it, an RTL layout looks broken even if the text renders correctly.

خاصية `dir` تحول اتجاه التخطيط بالكامل. ملف CSS الخاص بـ RTL في Bootstrap يُحمّل بشكل شرطي — يعكس margins وpaddings واتجاهات flex. بدونه، التخطيط RTL يبدو مكسوراً حتى لو الخط يُعرض بشكل صحيح.

---

## Things that are non-obvious in the code — be ready to explain
## أشياء غير واضحة في الكود — كن مستعداً لشرحها


- **`trans_begin` vs `trans_start`** — `trans_start/complete` is CI3's convenience wrapper that automatically decides commit/rollback. We use `trans_begin/commit/rollback` explicitly because we need to check `affected_rows()` between queries to decide ourselves. If we used `trans_start`, we'd have no way to inspect the affected rows result mid-transaction.
  **`trans_begin` مقابل `trans_start`** — `trans_start/complete` هو wrapper في CI3 يقرر commit/rollback تلقائياً. نستخدم `trans_begin/commit/rollback` صراحةً لأننا نحتاج فحص `affected_rows()` بين الاستعلامات لنقرر بأنفسنا.

- **`uniqid('INV', true)` in the INSERT** — the `true` second argument adds microseconds to make it more unique. It's a throwaway value — just needs to be unique enough to not collide with the UNIQUE constraint while we wait for `insert_id()`.
  المعامل الثاني `true` يضيف microseconds لجعله أكثر تفرداً. هو قيمة مؤقتة — يكفي أن تكون فريدة لعدم التصادم مع UNIQUE constraint ريثما نحصل على `insert_id()`.

- **`while (ob_get_level()) ob_end_clean()`** in the CSV export — CI3 wraps output in its own buffer. Before we can stream a file directly to `php://output`, we need to flush all active buffers. The `while` loop handles nested buffers.
  في تصدير CSV — CI3 يلف المخرجات في buffer خاص. قبل أن نبث ملفاً مباشرة لـ `php://output`، نحتاج تفريغ كل الـ buffers النشطة. حلقة `while` تتعامل مع الـ buffers المتداخلة.

- **UTF-8 BOM in CSV** — `\xEF\xBB\xBF` at the start of the file tells Excel it's UTF-8. Without it, Excel opens Arabic text with the wrong encoding and shows garbage characters. This is a known Excel quirk that has existed for 20 years.
  `\xEF\xBB\xBF` في بداية الملف يخبر Excel أنه UTF-8. بدونه، Excel يفتح النص العربي بترميز خاطئ ويظهر أحرفاً غير مفهومة. هذا خطأ Excel معروف موجود منذ 20 سنة.

- **`where($key, $val, false)` in Stock_model::get_low_stock** — the `false` third argument tells CI3's query builder not to escape the value as a string literal. Without it, `p.alert_quantity` would be quoted as `'p.alert_quantity'` and the comparison would be against a string, not the column value.
  المعامل الثالث `false` يخبر query builder في CI3 بعدم الهروب من القيمة كنص حرفي. بدونه، `p.alert_quantity` ستوضع بين اقتباسات وتُقارن بنص وليس بقيمة العمود.

- **`language` in autoload helpers** — CI3's `lang()` function comes from `system/helpers/language_helper.php`. It's not loaded by default. Forgetting this causes a "Call to undefined function lang()" error in every view — which is exactly what happened when first running on WAMP.
  دالة `lang()` في CI3 تأتي من `system/helpers/language_helper.php`. لا تُحمّل افتراضياً. نسيان هذا يسبب خطأ "Call to undefined function lang()" في كل view — وهذا بالضبط ما حدث عند أول تشغيل على WAMP.
