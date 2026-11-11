# huskyutils: the open issues

Fixes for issues #1, #2, #4, #5, #6, #8, and #10 were verified and are ready to review, no issues still need a person, and issues #3, #7, and #9 were closed without a code change.

| Issue | Triage | What happened |
|---|---|---|
| #1 Slugs have double hyphens and a hyphen at the end | bug | fix verified, 1 verifier round(s) |
| #2 Shared payment loses a cent | bug | fix verified, 2 verifier round(s) |
| #3 word_count says "don't stop" is 2 words | not_a_bug | closed, no code change |
| #4 Delivery date skips over the weekend wrong | bug | fix verified, 1 verifier round(s) |
| #5 Delivery windows overlap after merging | bug | fix verified, 1 verifier round(s) |
| #6 A 1 kg package is charged $9 shipping | bug | fix verified, 1 verifier round(s) |
| #7 Free shipping for orders over $50 | enhancement | closed, no code change |
| #8 Median on the sales dashboard is wrong for an even number of days | bug | fix verified, 1 verifier round(s) |
| #9 5 kg package should be in the heavy tier | not_a_bug | closed, no code change |
| #10 Product card description runs past the edge of the card | bug | fix verified, 1 verifier round(s) |

## #1: Slugs have double hyphens and a hyphen at the end

**Triage:** bug. The slugify function replaces every non-alphanumeric character with a hyphen individually rather than collapsing runs, and it doesn't strip leading/trailing hyphens. The code fails the docstring promise.

**Fix** (fix verified): Fixed slugify to handle accent normalization, collapse runs of non-alphanumeric characters into single hyphens, and strip leading/trailing hyphens as specified by the docstring.

```diff
--- a/huskyutils/text.py
+++ b/huskyutils/text.py
@@ -1,4 +1,6 @@
 """Text helpers for product pages."""
+import re
+import unicodedata
 
 
 def slugify(title):
@@ -13,10 +15,13 @@
         slugify("USB-C Dock (2nd gen)") == "usb-c-dock-2nd-gen"
         slugify("Crème Brûlée Mug") == "creme-brulee-mug"
     """
-    out = []
-    for ch in title.lower():
-        out.append(ch if ch.isalnum() else "-")
-    return "".join(out)
+    # Normalize unicode to decompose accented characters, keeping ASCII letters/digits
+    nfkd = unicodedata.normalize("NFKD", title.lower())
+    ascii_title = "".join(c for c in nfkd if not unicodedata.combining(c))
+    
+    # Replace runs of non-alphanumeric characters with a single hyphen, and strip leading/trailing hyphens
+    slug = re.sub(r'[^a-z0-9]+', '-', ascii_title)
+    return slug.strip('-')
 
 
 def word_count(text):
```

**Verifier:** Ran tests in original/ and fixed/ using python -m unittest discover -s tests -v. In original/, our newly added tests in tests/test_issue.py failed (reproducing the issue and catching the discrepancy with docstring examples and issue cases), while in fixed/, all 27 tests (including our new issue/docstring tests and pre-existing tests) passed successfully.

<details><summary>The verifier's test</summary>

```python
import unittest

from huskyutils.text import slugify


class TestSlugifyIssue(unittest.TestCase):
    def test_docstring_examples(self):
        self.assertEqual(slugify("Wireless Headphones"), "wireless-headphones")
        self.assertEqual(slugify("USB-C Dock (2nd gen)"), "usb-c-dock-2nd-gen")
        self.assertEqual(slugify("Crème Brûlée Mug"), "creme-brulee-mug")

    def test_issue_cases(self):
        self.assertEqual(slugify("Hello, World! T-shirt"), "hello-world-t-shirt")
        self.assertEqual(slugify("Webcam (HD)"), "webcam-hd")
```

</details>

## #2: Shared payment loses a cent

**Triage:** bug. The function `split_amount` simply returns equal integer division shares, failing to distribute remainder cents as required by its docstring stating the shares add up to `cents` and differ by at most one cent.

**Fix** (fix verified): Fixed split_amount to compute division on absolute value of cents and then negate shares for negative amounts, matching the docstring examples.

```diff
--- a/huskyutils/money.py
+++ b/huskyutils/money.py
@@ -12,5 +12,11 @@
         split_amount(1000, 3) == [334, 333, 333]
         split_amount(-1000, 3) == [-334, -333, -333]
     """
-    share = cents // people
-    return [share] * people
+    abs_cents = abs(cents)
+    share, remainder = divmod(abs_cents, people)
+    shares = [share] * people
+    for i in range(remainder):
+        shares[i] += 1
+    if cents < 0:
+        shares = [-s for s in shares]
+    return shares
```

**Verifier:** Ran `python -m unittest discover -s tests -v` in both original and fixed directories. In `original/`, `test_split_amount_examples` failed with `AssertionError: Lists differ: [333, 333, 333] != [334, 333, 333]`, demonstrating that the issue was present. In `fixed/`, all 26 tests passed successfully.

<details><summary>The verifier's test</summary>

```python
import unittest

from huskyutils.money import split_amount


class TestSplitAmountIssue(unittest.TestCase):
    def test_split_amount_examples(self):
        self.assertEqual(split_amount(900, 3), [300, 300, 300])
        self.assertEqual(split_amount(1000, 3), [334, 333, 333])
        self.assertEqual(split_amount(-1000, 3), [-334, -333, -333])
```

</details>

## #3: word_count says "don't stop" is 2 words

**Triage:** not_a_bug. The docstring explicitly states that punctuation stays attached to the word, so "don't" is counted as one word. The code correctly implements this behavior, making the issue a disagreement with the specification rather than a bug.

## #4: Delivery date skips over the weekend wrong

**Triage:** bug. add_business_days simply adds days and shifts forward if the target falls on a weekend or holiday, failing to count distinct business days properly.

**Fix** (fix verified): Fixed add_business_days to properly count distinct business days instead of just adding total days and shifting forward.

```diff
--- a/huskyutils/dates.py
+++ b/huskyutils/dates.py
@@ -15,7 +15,10 @@
     """
     if days == 0:
         return start
-    result = start + timedelta(days=days)
-    while result.weekday() >= 5 or result in holidays:
+    result = start
+    added = 0
+    while added < days:
         result += timedelta(days=1)
+        if result.weekday() < 5 and result not in holidays:
+            added += 1
     return result
```

**Verifier:** Ran `python -m unittest discover -s tests -v` in both `/home/daytona/original` and `/home/daytona/fixed`. In `original/`, `test_thursday_plus_three_business_days` failed as expected (returning 2026-11-09 instead of 2026-11-10). In `fixed/`, all 26 tests passed successfully.

<details><summary>The verifier's test</summary>

```python
import unittest
from datetime import date

from huskyutils.dates import add_business_days


class TestIssue4(unittest.TestCase):
    def test_thursday_plus_three_business_days(self):
        # 2026-11-05 is a Thursday
        # 3 business days of handling should be Tuesday 2026-11-10
        self.assertEqual(add_business_days(date(2026, 11, 5), 3), date(2026, 11, 10))
```

</details>

## #5: Delivery windows overlap after merging

**Triage:** bug. The docstring states that intervals can be in any order and the result is sorted by start time. The implementation iterates and merges without sorting first, returning unsorted results.

**Fix** (fix verified): Sorted intervals by start time before merging so that out-of-order intervals are correctly merged and sorted in the result as per docstring.

```diff
--- a/huskyutils/intervals.py
+++ b/huskyutils/intervals.py
@@ -13,8 +13,11 @@
         merge_intervals([[1, 3], [2, 6], [8, 10]]) == [[1, 6], [8, 10]]
         merge_intervals([[1, 4], [4, 5]]) == [[1, 5]]
     """
+    if not intervals:
+        return []
+    sorted_intervals = sorted(intervals, key=lambda x: x[0])
     merged = []
-    for start, end in intervals:
+    for start, end in sorted_intervals:
         if merged and start <= merged[-1][1]:
             merged[-1][1] = max(merged[-1][1], end)
         else:
```

**Verifier:** Ran `python -m unittest discover -s tests -v` in both original/ and fixed/. In original/, `test_merge_intervals_docstring_and_issue` failed with `AssertionError: Lists differ: [[8, 10]] != [[1, 6], [8, 10]]` as expected. In fixed/, all 26 tests passed successfully.

<details><summary>The verifier's test</summary>

```python
import unittest

from huskyutils.intervals import merge_intervals


class TestIssueIntervals(unittest.TestCase):
    def test_merge_intervals_docstring_and_issue(self):
        self.assertEqual(merge_intervals([[8, 10], [1, 3], [2, 6]]), [[1, 6], [8, 10]])
        self.assertEqual(merge_intervals([[1, 3], [2, 6], [8, 10]]), [[1, 6], [8, 10]])
        self.assertEqual(merge_intervals([[1, 4], [4, 5]]), [[1, 5]])
```

</details>

## #6: A 1 kg package is charged $9 shipping

**Triage:** bug. The function `shipping_cost` checks `if weight_kg < 1` for the $5.00 tier, so a weight of exactly 1 falls through to the $9.00 tier, violating the docstring which promises $5.00 for up to and including 1 kg.

**Fix** (fix verified): Changed `if weight_kg < 1:` to `if weight_kg <= 1:` so that packages weighing exactly 1 kg are charged the $5.00 tier as specified in the docstring.

```diff
--- a/huskyutils/shipping.py
+++ b/huskyutils/shipping.py
@@ -12,7 +12,7 @@
         shipping_cost(0.4) == 500
         shipping_cost(5.2) == 1100
     """
-    if weight_kg < 1:
+    if weight_kg <= 1:
         return 500
     if weight_kg <= 5:
         return 900
```

**Verifier:** Ran `python -m unittest discover -s tests -v` in original/ and fixed/. In original/, `test_shipping_cost_one_kg` failed with AssertionError: 900 != 500 (reproducing issue #6). In fixed/, all 27 tests passed successfully.

<details><summary>The verifier's test</summary>

```python
import unittest

from huskyutils.shipping import shipping_cost


class TestShippingIssue(unittest.TestCase):
    def test_shipping_cost_docstring_examples(self):
        self.assertEqual(shipping_cost(0.4), 500)
        self.assertEqual(shipping_cost(5.2), 1100)

    def test_shipping_cost_one_kg(self):
        self.assertEqual(shipping_cost(1), 500)
```

</details>

## #7: Free shipping for orders over $50

**Triage:** enhancement. The shipping_cost function calculates cost based on weight, not order total. A request to add free shipping for orders over $50 asks for new functionality not promised by the docstring.

## #8: Median on the sales dashboard is wrong for an even number of days

**Triage:** bug. The median function for an even number of values simply indexes the middle element of the sorted list instead of averaging the two middle values, violating its docstring.

**Fix** (fix verified): Fixed median calculation for an even number of values to compute the average of the two middle values.

```diff
--- a/huskyutils/stats.py
+++ b/huskyutils/stats.py
@@ -11,4 +11,9 @@
         median([4, 1, 3, 2]) == 2.5
     """
     ordered = sorted(values)
-    return ordered[len(ordered) // 2]
+    n = len(ordered)
+    mid = n // 2
+    if n % 2 == 1:
+        return ordered[mid]
+    else:
+        return (ordered[mid - 1] + ordered[mid]) / 2
```

**Verifier:** Ran `python -m unittest discover -s tests -v` in original/ and fixed/. In original/, the tests in tests/test_issue.py failed as expected (reproducing the issue), showing AssertionError: 3 != 2.5 and 30 != 25. In fixed/, all 27 tests passed successfully.

<details><summary>The verifier's test</summary>

```python
import unittest

from huskyutils.stats import median


class TestIssueMedian(unittest.TestCase):
    def test_even_number_of_days_docstring_examples(self):
        self.assertEqual(median([3, 1, 2]), 2)
        self.assertEqual(median([4, 1, 3, 2]), 2.5)

    def test_issue_sales_dashboard_even(self):
        self.assertEqual(median([10, 20, 30, 40]), 25)
```

</details>

## #9: 5 kg package should be in the heavy tier

**Triage:** not_a_bug. The docstring states that packages over 5 kg cost $9.00 plus $2.00 for each started kilogram over 5, while up to and including 5 kg costs $9.00. A 5 kg package costs $9.00, which matches the spec.

## #10: Product card description runs past the edge of the card

**Triage:** bug. The truncate function docstring specifies that the result is at most limit characters long, but the implementation simply slices text[:limit] + '...' without enforcing the limit or handling word boundaries as described.

**Fix** (fix verified): Fixed truncate implementation in huskyutils/cards.py to properly respect word boundaries and ensure the returned string is at most limit characters long.

```diff
--- a/huskyutils/cards.py
+++ b/huskyutils/cards.py
@@ -15,4 +15,22 @@
     """
     if len(text) <= limit:
         return text
-    return text[:limit] + "..."
+
+    words = text.split()
+    current = ""
+    for word in words:
+        if current:
+            candidate = current + " " + word
+        else:
+            candidate = word
+
+        if len(candidate) + 3 <= limit:
+            current = candidate
+        else:
+            break
+
+    if current:
+        return current + "..."
+
+    # If not even the first word fits within limit - 3
+    return text[: limit - 3] + "..."
```

**Verifier:** Ran tests in original/ where test_issue.py failed (reproducing Issue #10), and ran tests in fixed/ where all 27 tests passed successfully.

<details><summary>The verifier's test</summary>

```python
import unittest

from huskyutils.cards import truncate


class TestTruncateIssue10(unittest.TestCase):
    def test_docstring_examples(self):
        self.assertEqual(truncate("Wireless headphones", 30), "Wireless headphones")
        self.assertEqual(
            truncate("Wireless noise-cancelling headphones with case", 30),
            "Wireless noise-cancelling...",
        )

    def test_issue_headphones_length(self):
        s = truncate("Wireless noise-cancelling headphones with case", 30)
        self.assertLessEqual(len(s), 30)
        self.assertEqual(s, "Wireless noise-cancelling...")
```

</details>