# Interactive Session: Prompt Engineering for Precision

**Surface:** Claude.ai (web)
**Duration:** 45–60 minutes
**Task Statements:** 4.1, 4.2, 4.3

---

## Learning Objectives

By the end of this session, you will be able to:
- Write explicit review criteria that reduce false positives
- Construct few-shot examples for ambiguous cases
- Understand why vague instructions fail to improve precision

---

## Prerequisites

- A Claude.ai account (Free, Pro, or Team)
- Familiarity with code review concepts
- No special setup required — all prompts are self-contained

---

## Session Steps

### Step 1 — Demonstrate the Vague Instruction Problem

**Prompt:**
```
Review this Python function for issues. Be conservative and only report high-confidence findings.

```python
def get_user_data(user_id):
    query = "SELECT * FROM users WHERE id = " + user_id
    result = db.execute(query)
    return result
```
```

**Observe:** Count how many findings Claude reports. Note what categories appear.

**Checkpoint 1:** Did Claude report issues beyond security? Were there style comments mixed in with real bugs?

---

### Step 2 — Add Explicit Categorical Criteria

**Prompt:**
```
Review this Python function for issues.

Report ONLY these categories:
- Security vulnerabilities (SQL injection, command injection, auth bypass)
- Bugs that cause incorrect behavior or runtime errors
- Data loss or corruption risks

SKIP these categories:
- Naming convention preferences
- Style choices where multiple valid options exist
- Comments about code clarity unless the comment contradicts actual behavior

```python
def get_user_data(user_id):
    query = "SELECT * FROM users WHERE id = " + user_id
    result = db.execute(query)
    return result
```
```

**Observe:** Compare the findings to Step 1. Did the number of findings change? Did the categories change?

**Checkpoint 2:** How did explicit categorical criteria change the output compared to "be conservative"?

---

### Step 3 — Test with a Style-Heavy File

**Prompt:**
```
Review this code using the same criteria (only Security, Bugs, Data risks — skip style):

```python
def Calc_Total(itemlist, disc=0):
    t = 0
    for i in itemlist:
        t = t + i['price']
    if disc:
        t = t - (t * disc/100)
    return t

def processOrder(orderdata):
    items = orderdata['items']
    total = Calc_Total(items, orderdata.get('discount', 0))
    db.execute("INSERT INTO orders VALUES (" + str(total) + ")")
    return total
```
```

**Observe:** Claude should skip the naming convention issues (Calc_Total, single-letter variables) and focus on the SQL injection in processOrder.

**Checkpoint 3:** Are naming issues skipped? Is the SQL injection correctly identified as the critical finding?

---

### Step 4 — Introduction to Few-Shot Examples

**Prompt:**
```
I need to review code for security issues. I want to train my review system with examples of what to report versus what to skip.

Here are my decision examples:

REPORT this:
Code: query = "SELECT * FROM orders WHERE id = " + order_id
Reason: Direct string concatenation enables SQL injection — attacker controls query structure

SKIP this:
Code: def calculate_total(items):
Reason: Uses camelCase-like naming — this is a style preference, not a bug or security issue

SKIP this:
Code: result = db.execute(parameterized_query, [order_id])
Reason: Parameterized query — this is correct SQL injection prevention, not an issue

REPORT this:
Code: if user.role = 'admin':
Reason: Assignment instead of comparison — will always evaluate to truthy, bypasses authorization check

Now review this code using the same judgment:

```python
def update_profile(user_id, new_email):
    validate_email(new_email)
    query = f"UPDATE users SET email='{new_email}' WHERE id={user_id}"
    db.execute(query)
    return True

def get_admin_panel(request):
    if request.user.authenticated:
        return render_template('admin.html')
```
```

**Observe:** Did Claude apply the judgment demonstrated in the examples? Did it catch the SQL injection while skipping style? Did it catch the missing authorization check in get_admin_panel?

**Checkpoint 4:** How did few-shot examples change the quality and consistency of output compared to instructions alone?

---

### Step 5 — Few-Shot for Format Consistency

**Prompt:**
```
Review the following code. Output each finding in exactly this format:

FILE: [filename]
LINE: [line number or range]
SEVERITY: [CRITICAL / HIGH / MEDIUM / LOW]
ISSUE: [one sentence describing the problem]
FIX: [one sentence describing the fix]

Examples of correct output format:
---
FILE: auth.py
LINE: 47
SEVERITY: CRITICAL
ISSUE: Token expiration uses local time instead of UTC, allowing replay attacks from different timezones.
FIX: Replace datetime.now() with datetime.utcnow() and compare against UTC expiration claim.
---
FILE: payment.py
LINE: 23
SEVERITY: HIGH
ISSUE: Missing null check on order.customer_id before database lookup will throw TypeError for guest orders.
FIX: Add guard: if not order.customer_id: raise ValidationError("Order has no customer association")
---

Now review:
FILE: refund.py
```python
def process_refund(order_id, amount, reason):
    order = db.query("SELECT * FROM orders WHERE id=" + order_id)
    if order.status != 'delivered':
        return False
    db.execute("UPDATE orders SET status='refunded', amount=" + str(amount))
    send_email(order.customer.email, f"Refund of {amount} approved")
    return True
```
```

**Observe:** Does Claude follow the exact format from the examples? Are line numbers included? Is severity correctly applied?

**Checkpoint 5:** Compare the output format to the examples. Is it consistent? What happened when Claude had to apply the format to a case not explicitly covered by the examples?

---

### Step 6 — Severity Calibration with Concrete Examples

**Prompt:**
```
Define these severity levels with concrete examples:

CRITICAL: Code that can be exploited right now with no user action
Example: SQL injection with direct string concatenation in a query

HIGH: Bug that affects multiple users or corrupts data
Example: Missing transaction wrapper — if the second write fails, data is partially updated

MEDIUM: Bug that affects specific edge cases
Example: Integer overflow only when order quantity exceeds 10,000 items

LOW: Best practice violation that could become a bug with future changes
Example: Using a mutable default argument in Python (def func(data=[]))

Now classify these findings:
1. Token stored in localStorage (accessible to XSS)
2. No rate limiting on password reset endpoint
3. Function raises unhandled exception when input list is empty
4. Variable name shadows built-in 'list' function
```

**Observe:** Does Claude apply the concrete definitions consistently? Is "token in localStorage" CRITICAL, HIGH, or MEDIUM? Does the concrete example anchor change the classification?

**Checkpoint 6:** Which classification was most surprising? Did the concrete severity examples improve calibration?

---

### Step 7 — Variation: Ambiguous Case Handling

Try this variation to see how few-shot examples handle edge cases:

**Prompt:**
```
I need to review test coverage quality. Here are my decision examples for what counts as a meaningful test:

MEANINGFUL TEST:
Test: test_login_with_wrong_password_returns_401
Reason: Tests a specific security-relevant behavior with a concrete expected outcome

NOT MEANINGFUL:
Test: test_user_model_exists
Reason: Only verifies instantiation — doesn't test any behavior

MEANINGFUL TEST:
Test: test_refund_blocked_for_non_delivered_orders
Reason: Tests a business rule with a specific condition

NOT MEANINGFUL:
Test: test_calculate_total_returns_number
Reason: Type-checking rather than behavior — doesn't test the calculation logic

Review this test file and identify which tests are meaningful vs not:

```python
def test_payment_gateway_connects():
    gateway = PaymentGateway()
    assert gateway is not None

def test_refund_amount_exceeds_limit_raises_error():
    with pytest.raises(RefundLimitError):
        process_refund(order_id="123", amount=600)

def test_order_has_status_field():
    order = Order(items=[], customer_id="C1")
    assert hasattr(order, 'status')

def test_discount_applied_before_tax():
    total = calculate_total(items=[{"price": 100}], discount=10, tax_rate=0.2)
    assert total == 108  # (100 - 10) * 1.2, not (100 * 1.2) - 10
```
```

---

## Session Debrief

**Key Takeaways:**

1. **"Be conservative" does not reduce false positives** — only explicit categorical criteria (report X, skip Y) work
2. **Few-shot examples do two things:** demonstrate format AND demonstrate judgment for ambiguous cases
3. **Severity calibration requires concrete examples** — abstract severity labels produce inconsistent classification
4. **High false positives in one category undermine trust in all categories** — even if security findings are accurate, noisy style findings erode developer trust in the whole system

**Exam Connection:**
- Task Statement 4.1: Explicit criteria over vague instructions
- Task Statement 4.2: Few-shot examples for consistent format and ambiguous-case handling
- Task Statement 4.3: The role of examples in enabling generalization to novel patterns
