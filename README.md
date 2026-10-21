# Data Quality & Privacy Audit (SQL Server)

A hands-on data quality, anomaly detection, privacy and document review project built in **SQL Server (SSMS)**.

> **All data is 100% synthetic.** Names, emails, phones, national IDs and addresses are fake. The dataset was generated with deliberately planted errors so every check can be scored against a known answer key.

## Files

| File | Description |
|---|---|
| `customer_transactions.csv` | 50,250 rows, 21 columns: PII, sales fields and a free-text `support_note` column |
| `documents_for_review.csv` | 2,000 short documents with embedded PII (last column is the answer key) |
| `ANSWER_KEY_do_not_open_first.csv` | 3,530 planted issues by `record_id`, used for scoring in Step 6 |

## Skills demonstrated

Data profiling, data quality assurance, sampling methods, anomaly detection, PII identification, data privacy safeguards, document review, business-rule validation and financial impact analysis.

## Headline results

| Area | Result |
|---|---|
| Missingness | 0.2% to 0.6% per field |
| Duplicates | 150 exact, 100 near-duplicates (250 pairs found by fuzzy match) |
| Invoice reconciliation failures | 491 rows (about 1%) |
| Revenue overstatement | 9.64%, with 104 records causing 99% of it |
| Outlier detection | Z-score flagged 117 and caught 117 of 120 planted outliers |
| Rule scoring vs answer key | 98.3% recall, 79.2% precision overall |
| PII in free-text notes | Patterns flagged 491; allow-list redacted 602 |
| Document review | 1,402 of 2,000 documents flagged for PII; 280 with national IDs |

---

## Setup

1. Create a database, for example `PracticeDB`.
2. Import `customer_transactions.csv` into a table named `tx_raw` using **Tasks > Import Flat File**. Set `support_note`, `full_name`, `email`, `street`, `city` and `category` to `varchar(MAX)`, the rest to `varchar(50)`, and allow nulls on every column.
3. Check the load. It must return **50,250**:

```sql
SELECT COUNT(*) AS total_rows FROM tx_raw;
```

### Alternative: script-based import

Use this if the wizard fails (for example with a truncation error on `support_note`). Copy the CSV to `C:\Temp\` first.

```sql
DROP TABLE IF EXISTS tx_raw;

CREATE TABLE tx_raw (
 record_id varchar(50), customer_id varchar(50), full_name varchar(MAX),
 email varchar(MAX), phone varchar(MAX), national_id varchar(MAX),
 dob varchar(50), country varchar(50), city varchar(MAX), street varchar(MAX),
 card_last4 varchar(50), ip_address varchar(50), order_date varchar(50),
 category varchar(MAX), quantity varchar(50), unit_price varchar(50),
 discount_pct varchar(50), total_amount varchar(50), payment_method varchar(50),
 status varchar(50), support_note varchar(MAX)
);

BULK INSERT tx_raw
FROM 'C:\Temp\customer_transactions.csv'
WITH (FORMAT='CSV', FIELDQUOTE='"', FIRSTROW=2,
      ROWTERMINATOR='0x0a', CODEPAGE='65001', TABLOCK);
```

Every column is loaded as text on purpose, so `TRY_CAST` and `TRY_CONVERT` are used throughout to handle bad values safely.

---

## Step 1: Profiling and QA metrics

**Check 1: Missingness %**

```sql
SELECT
 100.0*SUM(CASE WHEN NULLIF(LTRIM(RTRIM(email)),'') IS NULL THEN 1 ELSE 0 END)/COUNT(*) AS email_pct,
 100.0*SUM(CASE WHEN NULLIF(LTRIM(RTRIM(phone)),'') IS NULL THEN 1 ELSE 0 END)/COUNT(*) AS phone_pct,
 100.0*SUM(CASE WHEN NULLIF(LTRIM(RTRIM(city)),'') IS NULL THEN 1 ELSE 0 END)/COUNT(*) AS city_pct,
 100.0*SUM(CASE WHEN NULLIF(LTRIM(RTRIM(category)),'') IS NULL THEN 1 ELSE 0 END)/COUNT(*) AS category_pct
FROM tx_raw;
```

**Check 2: Exact duplicates (ignoring `record_id`)**

```sql
SELECT COUNT(*) AS extra_duplicate_rows
FROM (
 SELECT ROW_NUMBER() OVER (
   PARTITION BY customer_id, order_date, category, quantity, unit_price,
                total_amount, card_last4, ip_address
   ORDER BY record_id) AS rn
 FROM tx_raw) x
WHERE rn > 1;
```

**Check 3: Reconciliation (total = quantity x price x (1 - discount))**

```sql
SELECT COUNT(*) AS mismatched_rows
FROM tx_raw
WHERE ABS(TRY_CAST(total_amount AS decimal(18,2))
 - TRY_CAST(quantity AS int) * TRY_CAST(unit_price AS decimal(18,2))
 * (1 - TRY_CAST(discount_pct AS decimal(5,2))/100.0)) > 0.02;
```

**Check 3b: Break the mismatches into root-cause types**

```sql
WITH m AS (
 SELECT TRY_CAST(total_amount AS decimal(18,2)) AS tot,
        TRY_CAST(quantity AS int) AS qty,
        TRY_CAST(quantity AS int)*TRY_CAST(unit_price AS decimal(18,2))
          *(1-TRY_CAST(discount_pct AS decimal(5,2))/100.0) AS expected
 FROM tx_raw),
c AS (
 SELECT CASE WHEN tot < 0 THEN '1_negative_total'
             WHEN qty = 0 THEN '2_zero_quantity'
             WHEN tot >= 10*expected THEN '3_huge_vs_expected'
             ELSE '4_moderate_mismatch' END AS issue_type
 FROM m
 WHERE ABS(tot - expected) > 0.02)
SELECT issue_type, COUNT(*) AS n
FROM c GROUP BY issue_type ORDER BY issue_type;
```

Expected: 80 negative, 61 zero quantity, 200 huge vs expected, 150 moderate (491 total).

---

## Step 2: Sampling

Sample size formula: n = z^2 x p(1-p) / e^2. At 95% confidence and a 5% margin of error, n is about 384, or 381 after the finite-population correction.

**A. 1% simple random sample**

```sql
SELECT TOP 503 * INTO tx_sample_random FROM tx_raw ORDER BY NEWID();
SELECT COUNT(*) AS sample_rows FROM tx_sample_random;
```

**B. 1% stratified sample by country**

```sql
WITH r AS (
 SELECT *, ROW_NUMBER() OVER (PARTITION BY country ORDER BY NEWID()) AS rn,
        COUNT(*) OVER (PARTITION BY country) AS cnt
 FROM tx_raw)
SELECT * INTO tx_sample_strat FROM r WHERE rn <= CEILING(cnt*0.01);
SELECT COUNT(*) AS strat_rows FROM tx_sample_strat;
```

**C. Country mix: full data vs both samples**

```sql
SELECT f.country,
 100.0*f.n/SUM(f.n) OVER () AS full_pct,
 100.0*r.n/SUM(r.n) OVER () AS random_pct,
 100.0*s.n/SUM(s.n) OVER () AS strat_pct
FROM (SELECT country, COUNT(*) n FROM tx_raw GROUP BY country) f
JOIN (SELECT country, COUNT(*) n FROM tx_sample_random GROUP BY country) r ON r.country=f.country
JOIN (SELECT country, COUNT(*) n FROM tx_sample_strat GROUP BY country) s ON s.country=f.country;
```

**D. Error rate in the random sample** (change the table name to test the other samples)

```sql
SELECT COUNT(*) AS bad_in_sample,
 100.0*COUNT(*)/(SELECT COUNT(*) FROM tx_sample_random) AS bad_pct
FROM tx_sample_random
WHERE ABS(TRY_CAST(total_amount AS decimal(18,2))
 - TRY_CAST(quantity AS int)*TRY_CAST(unit_price AS decimal(18,2))
 *(1-TRY_CAST(discount_pct AS decimal(5,2))/100.0)) > 0.02;
```

**E. Systematic sample (every 100th row from a random start)**

```sql
DECLARE @k int = 100;
DECLARE @start int = 1 + ABS(CHECKSUM(NEWID())) % @k;

WITH n AS (
 SELECT *, ROW_NUMBER() OVER (ORDER BY record_id) AS rn FROM tx_raw)
SELECT * INTO tx_sample_sys FROM n
WHERE (rn - @start) % @k = 0 AND rn >= @start;

SELECT COUNT(*) AS sys_rows FROM tx_sample_sys;
```

Takeaway: stratified sampling matched the true country mix almost exactly. A 1% sample estimates the overall error rate but can miss rare issues, so automated checks run on 100% of the data.

---

## Step 3: Automated anomaly detection

**A. Z-score outliers per category (|z| > 3)**

```sql
WITH v AS (
 SELECT LTRIM(RTRIM(LOWER(category))) AS cat,
        TRY_CAST(total_amount AS decimal(18,2)) AS amt
 FROM tx_raw WHERE category IS NOT NULL),
s AS (SELECT cat, AVG(amt) AS mu, STDEV(amt) AS sd FROM v GROUP BY cat)
SELECT COUNT(*) AS flagged_z3
FROM v JOIN s ON s.cat = v.cat
WHERE ABS((v.amt - s.mu)/s.sd) > 3;
```

**B. IQR outliers per category (1.5 x IQR fences)**

```sql
WITH v AS (
 SELECT LTRIM(RTRIM(LOWER(category))) AS cat,
        TRY_CAST(total_amount AS decimal(18,2)) AS amt
 FROM tx_raw WHERE category IS NOT NULL),
q AS (
 SELECT DISTINCT cat,
  PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY amt) OVER (PARTITION BY cat) AS q1,
  PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY amt) OVER (PARTITION BY cat) AS q3
 FROM v)
SELECT COUNT(*) AS flagged_iqr
FROM v JOIN q ON q.cat = v.cat
WHERE v.amt > q.q3 + 1.5*(q.q3 - q.q1)
   OR v.amt < q.q1 - 1.5*(q.q3 - q.q1);
```

**C. Rule-based checks in one query**

```sql
SELECT
 SUM(CASE WHEN TRY_CAST(total_amount AS decimal(18,2)) < 0 THEN 1 ELSE 0 END) AS negative_total,
 SUM(CASE WHEN quantity = '0' THEN 1 ELSE 0 END) AS zero_qty,
 SUM(CASE WHEN TRY_CONVERT(date, order_date, 23) > CAST(GETDATE() AS date) THEN 1 ELSE 0 END) AS future_date,
 SUM(CASE WHEN TRY_CONVERT(date, order_date, 23) < '2000-01-01' THEN 1 ELSE 0 END) AS placeholder_date,
 SUM(CASE WHEN order_date LIKE '%/%' THEN 1 ELSE 0 END) AS mixed_date_format,
 SUM(CASE WHEN email IS NOT NULL AND email NOT LIKE '%_@_%._%' THEN 1 ELSE 0 END) AS invalid_email,
 SUM(CASE WHEN phone IS NOT NULL AND LEN(phone) < 10 THEN 1 ELSE 0 END) AS short_phone,
 SUM(CASE WHEN TRY_CONVERT(date, dob, 23) > DATEADD(YEAR,-10,GETDATE()) THEN 1 ELSE 0 END) AS dob_under_10,
 SUM(CASE WHEN country = 'USA' AND city = 'Lagos' THEN 1 ELSE 0 END) AS country_city_mismatch
FROM tx_raw;
```

**D. Near-duplicate detection (same customer, date and category, amount within 5 cents)**

```sql
SELECT COUNT(*) AS duplicate_pairs
FROM tx_raw a JOIN tx_raw b
  ON a.customer_id = b.customer_id
 AND a.order_date  = b.order_date
 AND a.category    = b.category
 AND a.record_id   < b.record_id
 AND ABS(TRY_CAST(a.total_amount AS decimal(18,2))
       - TRY_CAST(b.total_amount AS decimal(18,2))) <= 0.05;
```

Expected: z-score 117, IQR 487, near-duplicate pairs 250. The email format rule misses domain typos such as `.con`, which need a domain allow-list.

---

## Step 4: PII detection and privacy safeguards

**A. Column classification (no SQL)**

| Group | Columns |
|---|---|
| Direct identifiers | full_name, email, phone, national_id, street, ip_address, card_last4 |
| Quasi-identifiers | dob, city, country, order_date |
| Persistent ID (pseudonymise) | customer_id |
| Free text that may hide PII | support_note |
| Not PII | record_id, category, quantity, unit_price, discount_pct, total_amount, payment_method, status |

**B. Scan `support_note` for PII**

```sql
SELECT
 SUM(CASE WHEN support_note LIKE '%_@_%._%' THEN 1 ELSE 0 END) AS email_in_note,
 SUM(CASE WHEN support_note LIKE '%[0-9]%.[0-9]%.[0-9]%.[0-9]%' THEN 1 ELSE 0 END) AS ip_in_note,
 SUM(CASE WHEN support_note LIKE '%+[0-9][0-9]%' THEN 1 ELSE 0 END) AS phone_in_note,
 SUM(CASE WHEN support_note LIKE '%ID [0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9]%' THEN 1 ELSE 0 END) AS id_in_note,
 SUM(CASE WHEN support_note LIKE '%DOB [0-9][0-9][0-9][0-9]-%' THEN 1 ELSE 0 END) AS dob_in_note,
 SUM(CASE WHEN support_note LIKE '%ending [0-9][0-9][0-9][0-9]%' THEN 1 ELSE 0 END) AS card_in_note
FROM tx_raw;
```

**C. Privacy-safe view** (run it alone in its own query window)

```sql
CREATE OR ALTER VIEW v_tx_safe AS
SELECT
 record_id,
 CONVERT(varchar(64), HASHBYTES('SHA2_256', CONCAT(customer_id,'ChangeThisSecretSalt')), 2) AS customer_token,
 CASE WHEN CHARINDEX('@', email) > 0
      THEN LEFT(email,1) + '***' + SUBSTRING(email, CHARINDEX('@', email), 100)
      WHEN email IS NULL THEN NULL ELSE '[INVALID]' END AS email_masked,
 CASE WHEN phone IS NULL THEN NULL ELSE '***' + RIGHT(phone,3) END AS phone_masked,
 country, city,
 LEFT(dob,4) AS birth_year,
 order_date, category, quantity, unit_price, discount_pct,
 total_amount, payment_method, status,
 CASE WHEN support_note IS NULL OR support_note = ''
        OR support_note IN ('Delivered on time.','Customer satisfied.',
             'Item exchanged, size issue.','No issues reported.',
             'Awaiting courier pickup.','Promo code applied.')
      THEN support_note ELSE '[REDACTED - possible PII]' END AS support_note_safe
FROM tx_raw;
```

Use a real secret salt in practice and keep it out of version control.

**Count redactions**

```sql
SELECT COUNT(*) AS redacted_notes FROM v_tx_safe
WHERE support_note_safe = '[REDACTED - possible PII]';
```

**D. Verify the view is safe**

```sql
SELECT COUNT(*) AS leaks FROM v_tx_safe
WHERE support_note_safe LIKE '%@%' OR support_note_safe LIKE '%+[0-9][0-9]%';

SELECT (SELECT COUNT(DISTINCT customer_id) FROM tx_raw) AS real_ids,
       (SELECT COUNT(DISTINCT customer_token) FROM v_tx_safe) AS tokens;
```

Expected: scan flags 491 notes with PII, 602 notes redacted (the allow-list also catches names and addresses that patterns miss), 0 leaks, and 11,831 real IDs equal to 11,831 tokens.

---

## Step 5: Document review

Import `documents_for_review.csv` into `docs_raw` (`text` and `pii_types_present_KEY` as `varchar(MAX)`, the rest `varchar(50)`).

```sql
SELECT COUNT(*) AS docs FROM docs_raw;
```

**B. Automated first pass**

```sql
SELECT
 SUM(CASE WHEN text LIKE '%_@_%._%' THEN 1 ELSE 0 END) AS email_docs,
 SUM(CASE WHEN text LIKE '%Phone: +%' THEN 1 ELSE 0 END) AS phone_docs,
 SUM(CASE WHEN text LIKE '%ID number [0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9]%' THEN 1 ELSE 0 END) AS id_docs,
 SUM(CASE WHEN text LIKE '%Lives at%' THEN 1 ELSE 0 END) AS address_docs,
 SUM(CASE WHEN text LIKE '%_@_%._%' OR text LIKE '%Phone: +%' OR text LIKE '%Lives at%'
           OR text LIKE '%ID number [0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9]%'
      THEN 1 ELSE 0 END) AS docs_with_pii
FROM docs_raw;
```

**C. Random sample of 50 documents for manual review** (the KEY column is left out on purpose)

```sql
SELECT TOP 50 doc_id, doc_type, text FROM docs_raw ORDER BY NEWID();
```

Review guideline used: types are name, email, phone, national_id, address. Severity is High for national ID, Medium for email, phone or address, and Low for a name alone. Actions are Escalate (High), Redact (Medium or Low) and Clear (none). Unclear cases are escalated, never guessed. A blind second pass on 10 documents checks consistency, and the log is scored with precision and recall (recall weighted higher).

Expected: 931 email, 499 phone, 280 ID, 305 address, and 1,402 documents with PII out of 2,000.

---

## Step 6: QA scoring against the answer key

Import `ANSWER_KEY_do_not_open_first.csv` into `answer_key` (`detail` as `varchar(MAX)`).

```sql
SELECT COUNT(*) FROM answer_key;   -- expect 3,530
```

**B. Turn the rules into flags**

```sql
DROP TABLE IF EXISTS my_flags;
SELECT record_id, CAST('negative_amount' AS varchar(50)) AS flag_type
INTO my_flags FROM tx_raw WHERE TRY_CAST(total_amount AS decimal(18,2)) < 0;

INSERT INTO my_flags SELECT record_id,'zero_quantity' FROM tx_raw WHERE quantity='0';
INSERT INTO my_flags SELECT record_id,'future_date' FROM tx_raw WHERE TRY_CONVERT(date,order_date,23) > CAST(GETDATE() AS date);
INSERT INTO my_flags SELECT record_id,'placeholder_date' FROM tx_raw WHERE TRY_CONVERT(date,order_date,23) < '2000-01-01';
INSERT INTO my_flags SELECT record_id,'mixed_date_format' FROM tx_raw WHERE order_date LIKE '%/%';
INSERT INTO my_flags SELECT record_id,'invalid_email' FROM tx_raw WHERE email IS NOT NULL AND email NOT LIKE '%@%';
INSERT INTO my_flags SELECT record_id,'typo_email_domain' FROM tx_raw WHERE email LIKE '%.con';
INSERT INTO my_flags SELECT record_id,'truncated_phone' FROM tx_raw WHERE phone IS NOT NULL AND LEN(phone) < 10;
INSERT INTO my_flags SELECT record_id,'country_city_mismatch' FROM tx_raw WHERE country='USA' AND city='Lagos';
INSERT INTO my_flags SELECT record_id,'implausible_dob' FROM tx_raw WHERE TRY_CONVERT(date,dob,23) > DATEADD(YEAR,-10,GETDATE());
INSERT INTO my_flags SELECT record_id,'total_mismatch' FROM tx_raw
WHERE ABS(TRY_CAST(total_amount AS decimal(18,2))
 - TRY_CAST(quantity AS int)*TRY_CAST(unit_price AS decimal(18,2))
 *(1-TRY_CAST(discount_pct AS decimal(5,2))/100.0)) > 0.02;
```

Add the z-score outliers in a **separate batch**:

```sql
WITH v AS (
 SELECT record_id, LTRIM(RTRIM(LOWER(category))) AS cat,
        TRY_CAST(total_amount AS decimal(18,2)) AS amt
 FROM tx_raw WHERE category IS NOT NULL),
s AS (SELECT cat, AVG(amt) AS mu, STDEV(amt) AS sd FROM v GROUP BY cat)
INSERT INTO my_flags
SELECT v.record_id, 'outlier_amount'
FROM v JOIN s ON s.cat = v.cat
WHERE ABS((v.amt - s.mu)/s.sd) > 3;
```

**C. Score every rule (TP, FP, FN, precision, recall)**

```sql
SELECT issue, tp, fp, fn,
 CAST(100.0*tp/NULLIF(tp+fp,0) AS decimal(5,1)) AS precision_pct,
 CAST(100.0*tp/NULLIF(tp+fn,0) AS decimal(5,1)) AS recall_pct
FROM (
 SELECT COALESCE(f.flag_type, k.planted_issue) AS issue,
  SUM(CASE WHEN f.record_id IS NOT NULL AND k.record_id IS NOT NULL THEN 1 ELSE 0 END) AS tp,
  SUM(CASE WHEN k.record_id IS NULL THEN 1 ELSE 0 END) AS fp,
  SUM(CASE WHEN f.record_id IS NULL THEN 1 ELSE 0 END) AS fn
 FROM (SELECT DISTINCT record_id, flag_type FROM my_flags) f
 FULL JOIN answer_key k
   ON k.record_id = f.record_id AND k.planted_issue = f.flag_type
 WHERE COALESCE(f.flag_type, k.planted_issue) IN (SELECT DISTINCT flag_type FROM my_flags)
 GROUP BY COALESCE(f.flag_type, k.planted_issue)) t
ORDER BY issue;
```

**D. Investigate the gaps**

Missed typo emails (all are on `proton.me`, where the planted `.com` to `.con` change had no effect, so the answer key overstates this issue):

```sql
SELECT TOP 10 k.record_id, t.email FROM answer_key k
JOIN tx_raw t ON t.record_id = k.record_id
WHERE k.planted_issue='typo_email_domain' AND t.email NOT LIKE '%.con';
```

Issue types with no rule yet:

```sql
SELECT planted_issue, COUNT(*) AS n FROM answer_key
WHERE planted_issue NOT IN (SELECT DISTINCT flag_type FROM my_flags)
GROUP BY planted_issue ORDER BY n DESC;
```

Findings: 98.3% recall and 79.2% precision overall. The broad `total_mismatch` rule has only 30.5% precision because it also catches negatives, zero quantities, outliers and wrong unit prices, so each record needs a single root cause (rule precedence). The 12 rules cover 38% of the 22 issue types.

---

## Step 7: Business-logic validation and financial impact

Business rules:

1. Line total = quantity x unit price x (1 - discount), within 2 cents.
2. A negative total is only valid on a refunded order.
3. Quantity must be at least 1.
4. No order may be dated in the future.
5. Discount must be 0, 5, 10, 15 or 20 percent.

**B. Check each rule**

```sql
SELECT
 SUM(CASE WHEN ABS(tot-expd) > 0.02 THEN 1 ELSE 0 END) AS r1_line_total,
 SUM(CASE WHEN tot < 0 AND st <> 'refunded' THEN 1 ELSE 0 END) AS r2_negative_not_refund,
 SUM(CASE WHEN qty < 1 THEN 1 ELSE 0 END) AS r3_qty_lt_1,
 SUM(CASE WHEN od > CAST(GETDATE() AS date) THEN 1 ELSE 0 END) AS r4_future_dated,
 SUM(CASE WHEN disc NOT IN (0,5,10,15,20) THEN 1 ELSE 0 END) AS r5_bad_discount
FROM (
 SELECT LOWER(status) AS st,
  TRY_CAST(total_amount AS decimal(18,2)) AS tot,
  TRY_CAST(quantity AS int) AS qty,
  TRY_CONVERT(date, order_date, 23) AS od,
  TRY_CAST(discount_pct AS decimal(5,2)) AS disc,
  TRY_CAST(quantity AS int)*TRY_CAST(unit_price AS decimal(18,2))
   *(1-TRY_CAST(discount_pct AS decimal(5,2))/100.0) AS expd
 FROM tx_raw) x;
```

**C. Financial impact on completed orders**

```sql
WITH x AS (
 SELECT TRY_CAST(total_amount AS decimal(18,2)) AS tot,
  TRY_CAST(quantity AS int)*TRY_CAST(unit_price AS decimal(18,2))
   *(1-TRY_CAST(discount_pct AS decimal(5,2))/100.0) AS expd
 FROM tx_raw WHERE LOWER(status) = 'completed')
SELECT COUNT(*) AS completed_rows,
 SUM(tot) AS reported_revenue,
 SUM(expd) AS expected_revenue,
 SUM(tot-expd) AS net_overstatement,
 100.0*SUM(tot-expd)/SUM(expd) AS overstatement_pct
FROM x;
```

**D. What drives the overstatement**

```sql
WITH x AS (
 SELECT TRY_CAST(total_amount AS decimal(18,2))
  - TRY_CAST(quantity AS int)*TRY_CAST(unit_price AS decimal(18,2))
   *(1-TRY_CAST(discount_pct AS decimal(5,2))/100.0) AS diff
 FROM tx_raw WHERE LOWER(status) = 'completed')
SELECT COUNT(*) AS rows_over_1000, SUM(diff) AS their_total_diff
FROM x WHERE diff > 1000;
```

**E. How refunds are recorded**

```sql
SELECT LOWER(status) AS st,
 SUM(CASE WHEN TRY_CAST(total_amount AS decimal(18,2)) < 0 THEN 1 ELSE 0 END) AS negative_rows,
 SUM(CASE WHEN TRY_CAST(total_amount AS decimal(18,2)) > 0 THEN 1 ELSE 0 END) AS positive_rows
FROM tx_raw GROUP BY LOWER(status);
```

Findings: 491 rule-1 failures, 77 negatives on non-refunded orders, 61 zero quantities, 72 future dates and 0 bad discounts. Reported revenue on 42,801 completed orders was 9,779,111 against 8,919,154 expected, a 9.64% overstatement, and 104 records account for 99% of it. Refunds are stored as positive amounts with a refunded status, so the negative totals are treated as errors.

---

## Notes

- Results marked "expected" come from a reference run of the same logic on this dataset. Counts that depend on today's date (`GETDATE()`) can shift slightly over time, and random samples differ on every run.
- Because the data is synthetic, precision and recall are measured against a planted answer key. Real data has no key, so spot-checks and escalation matter even more.
- Pseudonymised data is still personal data under laws such as the Nigeria Data Protection Act 2023 and GDPR. Never apply these techniques to real personal data without proper authorisation.
