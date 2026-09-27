# Project 01: The Messy Sales Report

**Clean a messy sales export, find which regions and products are slipping, and build a one-page dashboard.**

**Tool:** Excel or Google Sheets · **Time:** about 4 hours · **Stage:** [5 of the roadmap](../stage-5-portfolio-project/)

---

## 📋 The brief

Your manager at **Fernway Goods**, an online home and office goods retailer, drops a file on you Monday morning:

> "Sales export from the old system. Can you tell me which regions and products are slipping? Board meeting Thursday."

The file is [`datasets/fernway-sales-messy.csv`](datasets/fernway-sales-messy.csv). 812 rows. One financial year of orders, July 2025 to June 2026.

Open it and look before you touch anything. It's ugly on purpose. This is what real exported data looks like, and cleaning it is the actual job.

**Columns:** `order_id`, `order_date`, `customer_name`, `region`, `category`, `product`, `quantity`, `unit_price`, `sales_channel`

Fernway is a made-up company. The data is generated, not real customers.

---

## 🤖 Before you start: use AI like a working analyst

You are not going to use AI to do this project for you. If you do, you'll feel it in the interview, because you won't be able to explain your own work.

Use it the way a working analyst does. Four moves:

| Move | What it looks like |
|---|---|
| **1. Translate the ask** | Turn a vague question into a precise one before you touch the data. |
| **2. Unblock, then understand** | Your formula breaks. Paste it to AI, get unstuck, then ask **why** it broke. The why is the skill. |
| **3. Sanity-check the finding** | Before you tell anyone a number, ask AI to attack it. "What could make this number wrong?" |
| **4. Draft the story** | Give AI your real findings and ask for a first draft. Then rewrite it in your own words. |

Use whichever assistant you have: Claude, ChatGPT, Gemini. A free tier is fine.

**The rule that protects you:** never present a number you can't explain. Your job isn't done when the formula runs. It's done when you can explain it to another human without looking at the screen.

---

## Step 1 · Audit the mess (30 min)

Don't fix anything yet. Make a list of every problem you can see. Scroll. Sort columns. Look at the edges.

Then check yourself against AI:

```text
I'm a data analyst. I've been given a messy sales CSV export with columns: order_id, order_date, customer_name, region, category, product, quantity, unit_price, sales_channel. Before I clean it, what are the 8 most common data quality problems I should systematically check for in a file like this, and what's the fastest way to check for each one in Excel?
```

<details>
<summary><b>What you should have found</b> (open after your own audit)</summary>

- Exact duplicate rows (the same order exported twice)
- Three different date formats in one column: `2025-11-04`, `11/4/2025`, `Nov 4, 2025`
- Prices stored as text: `$129.00`, some with stray spaces
- Quantity cells that are blank, and a few written as words (`two`)
- Category casing chaos: `Office`, `OFFICE`, `office`, and leading spaces
- Region in lowercase in places, and some blank region cells
- Product names with trailing spaces (these silently break pivot grouping)

</details>

---

## Step 2 · Clean it, with a paper trail (60 to 90 min)

**Work on a copy.** Duplicate the raw sheet and name it `clean`. An analyst never destroys the original.

In your `clean` sheet, add helper columns instead of editing values in place. Your raw data is in columns A to I. Add these helper columns in this order, starting in column J:

| J | K | L | M | N | O | P | Q |
|---|---|---|---|---|---|---|---|
| `clean_date` | `clean_price` | `clean_qty` | `clean_category` | `clean_region` | `clean_product` | `revenue` | `fy_quarter` |

Every formula below assumes row 2 and fills down. If you put the columns somewhere else, adjust the references.

| Problem | The fix |
|---|---|
| **Duplicates** | Excel: select all columns → Data → Remove Duplicates. Google Sheets: Data → Data cleanup → Remove duplicates. Note how many it removed. You'll report this. |
| **Mixed date formats** | `clean_date` (J): `=DATEVALUE(TRIM(B2))`, then format the column as a date. This handles all three formats in a US-locale Excel. `DATEVALUE` is locale-sensitive in Excel and Google Sheets, so if it errors on your machine (likely outside the US), you haven't done anything wrong. Use the Move 2 prompt below. Diagnosing it is genuinely the job. |
| **Text prices** | `clean_price` (K): `=VALUE(SUBSTITUTE(TRIM(H2),"$",""))` |
| **Quantity blanks and words** | Business rule, confirmed with your manager: blank means 1, words mean the number they spell. Find & Replace the words first (`one` → `1`, `two` → `2`, `three` → `3`). Then `clean_qty` (L): `=IF(G2="",1,VALUE(G2))` |
| **Casing and spaces** | `clean_category` (M): `=PROPER(TRIM(E2))` · `clean_region` (N): `=IF(TRIM(D2)="","Unknown",PROPER(TRIM(D2)))` · `clean_product` (O): `=TRIM(F2)` |
| **Revenue** | `revenue` (P): `=L2*K2`, quantity times price. Your pivots live on this column. If you turned your range into an Excel Table first, use `=[@clean_qty]*[@clean_price]` instead. Typing `=clean_qty*clean_price` returns `#NAME?`, because column headers aren't names Excel can resolve. |
| **Fiscal quarter** | Fernway's year starts in July, so calendar quarters are wrong here. `fy_quarter` (Q): `="Q"&(INT(MOD(MONTH(J2)-7,12)/3)+1)`. Use this in your pivots, not Excel's automatic date grouping. |

**Move 2 prompt, if `DATEVALUE` errors:**

```text
My =DATEVALUE(TRIM(B2)) formula returns #VALUE! for some rows. I'm in [EXCEL or GOOGLE SHEETS]. The column mixes these formats: "2025-11-04", "11/4/2025", "Nov 4, 2025". My locale is [YOUR COUNTRY]. Give me one robust helper-column approach that converts all three to real dates, and explain how it handles each format so I can explain it to someone else.
```

<details>
<summary><b>Self-check</b> (open before you build anything)</summary>

- After removing duplicates you should have **800 unique orders**. The raw file has 812 rows of data, so 12 duplicates.
- Total revenue across all clean rows: **$70,507**.

Within a few dollars? Your cleaning is right. If not, the usual suspects are quantity blanks or text prices that slipped through.

**The two judgement calls, because they change the number:** 13 rows have a blank quantity and 21 have a blank region. The business rule says blank quantity counts as 1, and blank region becomes `Unknown`. That's a decision, not a fact, and **$2,288** of revenue ends up in an `Unknown` region because of it. Say so in your write-up. Being explicit about the assumptions behind a number is the difference between an analyst and a calculator.

</details>

---

## Step 3 · Find the story (45 min)

Now the question your manager actually asked: **which regions and products are slipping?**

1. Insert a PivotTable from your clean data (Google Sheets: Insert → Pivot table).
2. Use your `fy_quarter` column, not Excel's automatic date grouping. Fernway's year runs July to June, so auto-grouping would label your periods Q3 2025 to Q2 2026 and nothing below would line up.
3. **Pivot 1:** revenue by region by fiscal quarter. Look for the region that falls every quarter.
4. **Pivot 2:** revenue by category by fiscal quarter. Look for the category that ends the year well below where it started.

<details>
<summary><b>Check your findings</b> (open after you've built both pivots)</summary>

- **South is the slipping region.** It falls every single quarter: $5,447 → $5,173 → $5,014 → $3,396. Down about 38% across the year, and the only region that declines in all four quarters.
- **Office is the slipping category.** Not a straight line (Q4 ticks up against Q3), but the year trend is down about 15% from Q1 to Q4: $6,953 → $6,400 → $5,424 → $5,905. Reporting that honestly, "down 15% on the year, small Q4 recovery", is what separates a real analyst from a chart-maker.
- Your `Unknown` region holds about **$2,288**. That's the 21 blank-region rows, not a real place. Note it rather than hiding it.

</details>

**Move 3, sanity-check before you present:**

```text
I analyzed a year of sales data (Jul 2025 to Jun 2026) and found: the South region declined every quarter, from $5,447 to $3,396, and the Office category is down ~15% year over year with a small Q4 recovery. About 3% of revenue sits in an "Unknown" region because those cells were blank in the source. Before I present this: what could make these findings wrong or misleading? What should I check or caveat? Keep it to the 5 most important checks.
```

---

## Step 4 · The one-page dashboard (60 min)

One sheet, named `Dashboard`. Not fancy. Legible.

- **A headline cell** in large text: the single most important finding, in one sentence.
- **A line or column chart:** revenue by quarter for the four regions. South's decline should be visible without squinting.
- **A column chart:** revenue by category by quarter.
- **Three number cells:** total revenue, revenue this quarter, and this quarter vs Q1 as a percentage.

Then write a three-sentence summary under the headline. Draft it with AI (Move 4), then rewrite it in your own words:

```text
Draft a 3-sentence executive summary of these findings for a retail sales dashboard, written for a manager who has 30 seconds: [PASTE YOUR ACTUAL NUMBERS: total revenue, the South decline by quarter, the Office trend]. Plain business English, no jargon, lead with the most important number.
```

---

## ✅ Portfolio-ready when

- [ ] My raw sheet is untouched, and my clean sheet documents every fix in helper columns
- [ ] My totals match the self-check (800 unique orders, about $70,507 revenue)
- [ ] My one-page dashboard shows the South decline and the Office trend at a glance
- [ ] My three-sentence summary is in my own words, and I can defend every number in it
- [ ] I saved it as `fernway-sales-analysis.xlsx` and it opens cleanly on someone else's machine (from Google Sheets: File → Download → Microsoft Excel)

## 🚀 Make it your portfolio

1. **Fork this repo.**
2. Add `fernway-sales-analysis.xlsx` to this folder in your fork.
3. Fill in [`write-up-template.md`](write-up-template.md). This is what a hiring manager actually reads.
4. Link your fork on your CV.

Finished? Post it on Threads and tag [@stanleywangg](https://www.threads.com/@stanleywangg).

---

Built by [Stanley Wang](https://stanleywangg.com/?utm_source=github&utm_medium=readme&utm_campaign=data-analyst-roadmap). Back to the [roadmap](../README.md).
