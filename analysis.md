# Fashion sales dataset — full analysis

Source file: `fashion_sales.csv` (sha256 `1be02313a44994d7d85966300f9bb660e53878e8a2cdb150ac6273afbd3a3fa1`)
Charts: `charts/` (10 PNGs) · Analysis date: November 2026

---

## 1. What the file contains

A two-year sales summary for a clothing retailer. It records, for each month, how each product
category sold through each sales channel: how many units, at what price, with how much advertising
spend, how customers rated it, and how much came back as returns.

Totals across the file:

| | Value |
|---|---:|
| Rows | 240 |
| Columns | 14 |
| Months covered | 24 (Jan 2024 – Dec 2025) |
| Revenue | $13,220,482 |
| Units sold | 202,500 |
| Advertising spend | $2,337,530 |
| Blended price per unit | $65.29 |

## 2. How it is organised

**One row = one month × one category × one channel.** That is the "grain" of the file.

- **5 categories:** Dresses, Denim, Outerwear, Footwear, Accessories
- **2 channels:** Online, Store
- **24 months:** `2024-01` … `2025-12`

24 × 5 × 2 = **240 rows**, and the file is a complete "balanced panel": every category-channel
combination appears in every month, with no duplicates and no gaps. The unique key is
`(month, category, channel)`. Nothing is missing — no blank cells anywhere.

Because the panel is balanced, all comparisons are like-for-like: each category has exactly 48 rows
(24 months × 2 channels), each channel exactly 120, each month exactly 10 (5 categories × 2 channels).

## 3. Column reference

| Column | Type | Meaning | Distinct values / range |
|---|---|---|---|
| `month` | text | Month label `YYYY-MM` | 24 values |
| `year` | int | Year | 2024, 2025 |
| `month_num` | int | Month number 1–12 | 1–12 (always consistent with `month`) |
| `category` | text | Product line | 5 values |
| `channel` | text | Where it sold | Online, Store |
| `avg_temp_F` | float | Average temperature for that month | 31.3–80.4, **24 distinct** (one per month) |
| `list_price` | float | Full (pre-discount) price for the category | 35, 70, 85, 110, 180 (**5 distinct**) |
| `discount_pct` | int | Discount applied | 0, 10, 15, 20, 25, 30, 40 |
| `sale_price` | float | Actual price per unit sold | 19.57–197.18 |
| `ad_spend` | float | Advertising spend for that month-category-channel | 2,050–19,270, all multiples of 10 |
| `avg_rating` | float | Average customer rating (1–5 scale) | 3.4–5.0 in 0.1 steps |
| `units_sold` | int | Units sold | 220–2,488 |
| `revenue` | float | Revenue | 17,257.20–154,512.49 |
| `return_rate` | float | Share of units returned (0–1) | 0.010–0.356 |

**Four columns are attributes, not measurements:** `avg_temp_F` repeats the same value for all 10 rows
of a month, and `list_price` repeats for all rows of a category. They are joined-in lookup values.

## 4. How the columns depend on each other

### 4.1 Revenue is derived, not measured — exactly

```
revenue = sale_price × units_sold
```

This holds for **all 240 rows with maximum error 0.00000000**. `revenue` therefore carries no
independent information: any analysis of revenue is really an analysis of price × volume, and a model
predicting revenue from both `sale_price` and `units_sold` would be circular.

### 4.2 Sale price is list price minus discount, plus random noise

The intended relationship is `sale_price = list_price × (1 − discount_pct/100)`, and within a category
that is almost exactly what we see (correlation −0.93 to −0.96; the fitted slope equals −list_price/100
for every category). But a random wobble is added on top:

```
sale_price = list_price × (1 − discount_pct/100) × (1 + ε),
ε ≈ uniform on [−10%, +10%]   (observed mean −0.23%, sd 5.9%, range −9.8% … +9.9%)
```

Consequences that matter:

- **21 of 240 rows have a sale price *above* list price** (worst case +9.5%; e.g. Outerwear, list $180,
  discounted 0%, sold at $197.18).
- **All 43 rows marked `discount_pct = 0` still deviate from list price**, by −9.5% to +9.3%, so a
  "0% discount" row is not actually sold at list.
- Across the whole file `sale_price` and `discount_pct` correlate only −0.27 (because list prices differ
  by category and because of the noise), so price cannot be reconstructed from discount except within a
  single category.

### 4.3 What is *not* derived

`units_sold`, `ad_spend`, `avg_rating` and `return_rate` are independent columns — nothing computes them
from the others. They behave like generated random variables with structure built in (see §15).

## 5. Missing values and data quality

- **No missing values:** all 14 columns are complete in all 240 rows.
- All numeric columns parse cleanly — no currency symbols, commas, or stray text.
- `month`, `year` and `month_num` agree in all 240 rows.
- File is pure ASCII with CRLF line endings and a final newline (241 physical lines).
- Issues worth knowing: the 21 above-list sale prices, the non-zero deviations on `discount_pct = 0`
  rows, and `ad_spend` limited to multiples of 10 (240/240) — none of which look like real measurements.

## 6. Summary statistics

| Column | Min | Max | Mean | Median | Std dev |
|---|---:|---:|---:|---:|---:|
| `avg_temp_F` | 31.30 | 80.40 | 55.64 | 56.20 | 15.63 |
| `list_price` | 35 | 180 | 96.00 | 85 | 48.62 |
| `discount_pct` | 0 | 40 | 22.56 | 25.00 | 13.51 |
| `sale_price` | 19.57 | 197.18 | 73.71 | 68.31 | 39.46 |
| `ad_spend` | 2,050 | 19,270 | 9,739.71 | 9,430 | 4,615.69 |
| `avg_rating` | 3.40 | 5.00 | 4.219 | 4.20 | 0.282 |
| `units_sold` | 220 | 2,488 | 843.75 | 819.50 | 349.39 |
| `revenue` | 17,257.20 | 154,512.49 | 55,085.34 | 48,837.68 | 27,365.24 |
| `return_rate` | 0.010 | 0.356 | 0.173 | 0.175 | 0.086 |

### By category

| Category | Revenue | Share | Units | Share | Revenue/unit | Median return | Avg rating | ROAS |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| Outerwear | $4,045,992 | 30.6% | 31,905 | 15.8% | $126.81 | 13.8% | 4.23 | 8.52 |
| Dresses | $2,995,076 | 22.7% | 47,760 | 23.6% | $62.71 | 22.8% | 4.23 | 6.57 |
| Footwear | $2,794,483 | 21.1% | 33,765 | 16.7% | $82.76 | 22.7% | 4.20 | 6.18 |
| Denim | $2,078,650 | 15.7% | 39,984 | 19.7% | $51.99 | 13.8% | 4.18 | 4.25 |
| Accessories | $1,306,281 | 9.9% | 49,086 | 24.2% | $26.61 | 14.2% | 4.25 | 2.81 |

Revenue and unit rankings are almost inverted: Outerwear earns the most on the fewest units;
Accessories sell the most units on the least revenue.

### By channel

| Channel | Revenue | Share | Units | Share | Revenue/unit | Mean return | Avg rating | ROAS |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| Online | $6,966,152 | 52.7% | 105,687 | 52.2% | $65.91 | 0.247 | 4.149 | 5.20 |
| Store | $6,254,329 | 47.3% | 96,813 | 47.8% | $64.60 | 0.100 | 4.288 | 6.27 |

## 7. Seasonal patterns

Total revenue by calendar month (average of the two years, which isolates seasonality from the trend):

| Month | Jan | Feb | Mar | Apr | May | Jun | Jul | Aug | Sep | Oct | Nov | Dec |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| Revenue | $533,923 | $493,782 | $533,940 | $501,943 | $529,505 | $488,620 | $544,831 | $517,242 | $493,908 | $525,636 | **$719,079** | **$727,831** |
| Index (100 = average) | 97 | 90 | 97 | 91 | 96 | 89 | 99 | 94 | 90 | 95 | **131** | **132** |

- Peak/trough across calendar months: **1.49×**.
- November + December = **21.9%** of revenue (16.7% would be flat).
- Overall, calendar month explains only **η² = 0.084** of monthly revenue variance — but that is an
  averaging artefact: the categories peak in opposite seasons and cancel each other out.

**Per-category seasonality** (η² = share of that category's month-to-month variance explained by month):

| Category | η²(month) | Peak month | Trough month | Peak/trough | Nov+Dec share of its revenue |
|---|---:|---|---|---:|---:|
| Outerwear | 0.888 | Dec ($285,130) | Jul ($88,410) | 3.23× | 27.0% (1.62× its average month) |
| Dresses | 0.783 | Jul ($210,656) | Feb ($64,109) | 3.29× | 15.8% (**0.94×** — dips) |
| Footwear | 0.544 | Dec ($162,237) | Feb ($92,249) | 1.76× | 22.3% |
| Denim | 0.542 | Dec ($108,495) | Apr ($64,995) | 1.67× | 20.8% |
| Accessories | 0.417 | Nov ($70,666) | May ($45,794) | 1.54× | 21.1% |
| All pooled | 0.084 | Dec | Jun | 1.49× | 21.9% |

Key points:

- The business is **two opposing seasonal businesses**: Outerwear is winter (Dec/Jul = 3.23×),
  Dresses is summer (Jul/Feb = 3.29×). November–February is **48.0%** of Outerwear's revenue;
  May–August is **43.0%** of Dresses'.
- The holiday peak is therefore **a cold-weather event, not a general one**. Dresses actually trade
  *below* their monthly average in Nov–Dec (0.94×).
- **Seasonality is not the same thing as temperature.** For Footwear, Denim and Accessories, month
  dummies explain far more unit variation than temperature does — Footwear R² = 0.644 (month) versus
  0.046 (temperature); Denim 0.428 vs 0.075; Accessories 0.615 vs 0.063. Their seasonality is
  calendar/holiday-driven. Only Dresses (0.717 vs 0.386) and Outerwear (0.836 vs 0.595) are genuinely
  weather businesses.
- Within every category, high-volume months are cheaper months — corr(monthly units, monthly average
  price) = −0.80 Footwear, −0.72 Accessories, −0.64 Denim, −0.63 Outerwear, −0.40 Dresses.

## 8. Differences between 2024 and 2025

| Metric | 2024 | 2025 | Change |
|---|---:|---:|---:|
| Revenue | $6,406,734 | $6,813,748 | **+6.4%** (+$407,014) |
| Units | 97,712 | 104,788 | +7.2% |
| Revenue per unit | $65.57 | $65.02 | −0.8% |
| Avg discount | 22.9% | 22.3% | −2.7% |
| Ad spend | $1,098,180 | $1,239,350 | +12.9% |
| ROAS | 5.83 | 5.50 | −5.7% |
| Return rate | 17.2% | 17.4% | +1.2% |
| Avg rating | 4.238 | 4.200 | −0.9% |

**Growth decomposition (price vs volume):**

| Component | Amount | Share of the increase |
|---|---:|---:|
| Volume effect (Δunits × 2024 price) | **+$463,956** | **114%** |
| Price effect (Δprice × 2024 units) | −$53,097 | −13% |
| Cross term | −$3,845 | −1% |

Growth was **entirely volume-led**; average prices slipped. On its own, the year explains only
η² = 0.004 of monthly revenue variance — the change is small and not a shift in level so much as more
units moved at slightly lower prices.

**By channel:**

| Channel | Revenue change | Volume part | Price part | Units change | Revenue/unit 24 → 25 |
|---|---:|---:|---:|---:|---|
| Online | **+$333,019 (+10.0%)** | +$441,178 | −$95,460 | +13.3% | $66.94 → $65.01 |
| Store | +$73,994 (+2.4%) | +$31,117 | +$42,450 | +1.0% | $64.16 → $65.04 |

Online produced **82% of all growth**, by selling more at lower prices; the Store grew barely at all,
and its small gain was price-driven. Advertising spend grew almost twice as fast as revenue overall
(+12.9% vs +6.4%), so efficiency fell (ROAS 5.83 → 5.50).

**By category** (revenue change): Dresses +$124,979, Outerwear +$108,357, Footwear +$69,926,
Denim +$61,419, Accessories +$42,334. Only Outerwear (+$39,843) and Footwear (+$7,602) had a positive
price contribution.

**Same-month year-over-year is uneven, and the two biggest months declined:**

| Month | Jan | Feb | Mar | Apr | May | Jun | Jul | Aug | Sep | Oct | Nov | Dec |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| YoY change | +10.0% | +6.3% | +8.5% | +12.1% | +0.2% | +15.6% | +9.1% | **+23.5%** | +1.1% | −5.1% | **−4.7%** | +7.7% |

Part of that is weather: 2025-01 was colder than 2024-01 (31.3°F vs 36.2°F) and 2025-08 was cooler
(72.8°F vs 77.5°F), which shifts demand between Outerwear and Dresses.

## 9. How temperature affects each category

| Category | Slope (units per °F) | r | R² | p | Warm (≥60°F) vs cold mean units |
|---|---:|---:|---:|---|---|
| Dresses | **+18.59** | +0.621 | 0.386 | <0.001 | 1,240 vs 820 |
| Outerwear | **−16.30** | −0.771 | 0.595 | <0.001 | 397 vs 856 |
| Denim | −4.37 | −0.274 | 0.075 | 0.054 | 782 vs 870 |
| Accessories | −3.92 | −0.251 | 0.063 | 0.079 | 955 vs 1,071 |
| Footwear | −3.18 | −0.215 | 0.046 | 0.135 (n.s.) | 658 vs 736 |
| All rows pooled | −0.08 | −0.082 | 0.007 | 0.203 (n.s.) | — |

A joint **temperature × category** regression (R² = 0.438, adjusted 0.416) confirms the slopes really
differ — every interaction is significant against the Dresses baseline:

| Term | Coefficient | t |
|---|---:|---:|
| temp (Dresses slope) | +18.59 | 7.5 |
| temp × Denim | −22.95 | −6.6 |
| temp × Outerwear | −34.89 | −10.0 |
| temp × Footwear | −21.77 | −6.2 |
| temp × Accessories | −22.51 | −6.4 |

Net effect of a 20°F swing: Dresses ±372 units, Outerwear ∓326 units. Note the pooled correlation is
essentially zero because the two large effects cancel — weather looks irrelevant unless you segment.

## 10. Do bigger discounts lead to more units sold? Yes

| Discount | n | Mean units | Median units | vs 0% | Mean revenue | Revenue vs 0% |
|---|---:|---:|---:|---:|---:|---:|
| 0% | 43 | 586 | 577 | — | $48,291 | — |
| 10% | 12 | 607 | 562 | +4% | $44,764 | −7% |
| 15% | 13 | 812 | 773 | +38% | $56,744 | +18% |
| 20% | 45 | 784 | 755 | +34% | $50,519 | +5% |
| 25% | 28 | 802 | 768 | +37% | $50,389 | +4% |
| 30% | 46 | 911 | 917 | +55% | $61,826 | +28% |
| 40% | 53 | **1,129** | 1,124 | **+92%** | $63,035 | +31% |

Robustness checks:

- Raw: corr(discount, units) = **0.513** (p < 0.001), R² = 0.263, slope **+13.26 units per
  percentage point** (t = 9.2).
- Holds **within every category and channel**: Denim r = 0.748, Footwear 0.694, Outerwear 0.569,
  Dresses 0.550, Accessories 0.552; Online 0.587, Store 0.454 (all p < 0.001).
- Holds **against seasonality**: month dummies alone give R² = 0.228, discount alone 0.263, and
  discount + month gives R² = 0.349 with the discount slope still **+10.59 (t = 6.5)**.
- Holds **against category and weather**: discount + temperature + category gives R² = 0.459 with
  discount **+13.80/pp (t = 10.9)** while temperature becomes insignificant (t = −0.2).
- Full model (discount + channel + temperature + month dummies): R² = 0.373, discount **+11.27
  (t = 6.9)**, Online **+106 units (t = 2.9)**.

**Does discounting pay?** Each +1pp of discount cuts price by about 1% of list (−$0.86 for Dresses,
−$1.95 for Outerwear — i.e. exactly −list_price/100) and adds roughly 13 units. Those two effects
nearly cancel:

- corr(discount, revenue) = 0.202 (p = 0.001); controlled slope **+$406 per pp (t = 3.2)**, but the
  revenue model's R² is only 0.084.
- Revenue per +10pp **within** category: Outerwear +$7,139, Footwear +$3,863, Dresses +$3,613,
  Denim +$2,052, and **Accessories −$523 (p = 0.368)**.

So discounts reliably buy **volume**, but buy **revenue** only in higher-ticket categories.
Accessories (list $35, actual ~$27) are the exception: markdowns there move units without adding money.

## 11. Does ad spending pay off? Weakly, and implausibly

| Test | Result |
|---|---|
| corr(ad_spend, revenue) | 0.208 (p = 0.001) — **R² = 0.043** |
| corr(ad_spend, units) | 0.252 (p < 0.001) — R² = 0.063 |
| OLS revenue ~ spend | +**$1.23** revenue per $1 spent (t = 3.3) |
| Diminishing returns (spend² term) | t = 0.4 — **no detectable curvature** |
| Log-log elasticity | **0.175** (t = 3.3) → 10% more spend ≈ +1.75% revenue |
| ROAS | mean 7.25, median 5.53, **range 1.43 – 38.34**, sd 5.65 |

Spend quartiles:

| Quartile | Spend range | Mean revenue | Mean units | Mean discount | Mean rating | Mean return |
|---|---|---:|---:|---:|---:|---:|
| Q1 | $2,050–5,990 | $49,150 | 773 | 23.5% | 4.26 | 0.161 |
| Q2 | $6,100–9,410 | $50,519 | 796 | 21.6% | 4.22 | 0.170 |
| Q3 | $9,450–13,070 | $57,462 | 791 | 22.6% | 4.23 | 0.151 |
| Q4 | $13,250–19,270 | **$63,211** | **1,016** | 22.6% | 4.17 | 0.211 |

By category:

| Category | ROAS | corr(spend, revenue) | Spend as % of its revenue |
|---|---:|---:|---:|
| Outerwear | 8.52 | 0.18 | 11.7% |
| Dresses | 6.57 | 0.37 | 15.2% |
| Footwear | 6.18 | 0.61 | 16.2% |
| Denim | 4.25 | 0.56 | 23.5% |
| Accessories | **2.81** | 0.38 | **35.6%** |

Findings: spending is associated with more revenue in the right direction and statistically
significantly, but it explains under 5% of revenue variation, shows no diminishing returns (unlike real
media), and produces a 27× spread in ROAS — signs that this column is not real. Spend is also
uncorrelated with discounting (r = −0.002), so media and markdowns are not coordinated.
Accessories absorb 35.6% of their own revenue in advertising for the weakest returns.
The apparent link between spend and returns (r = +0.228, p < 0.001) is a **Simpson's paradox**: Online
receives both more spend (mean $11,168 vs $8,311) and more returns; within channel it disappears
(Online r = −0.127, p = 0.163; Store r = −0.006, p = 0.946).

## 12. What is related to returns? The channel, almost exclusively

| Group | Mean | Median | Std dev | Min–max |
|---|---:|---:|---:|---|
| Online | **0.247** | 0.235 | 0.045 | 0.165–0.356 |
| Store | **0.100** | 0.092 | 0.046 | 0.010–0.211 |

Welch test: difference **0.147, t = 25.0, df ≈ 238, p < 0.001**. The gap is present in every category,
all overwhelmingly significant: Denim +0.157 (t = 21.2), Footwear +0.151 (t = 26.3),
Accessories +0.148 (t = 26.8), Dresses +0.141 (t = 19.7), Outerwear +0.136 (t = 19.9).

The two channels barely overlap: **67 of the 67 store rows with returns below 10% are store rows**, and
**all 17 rows above 30% are online**. A regression on channel + category reaches **R² = 0.931** with an
Online effect of **+0.147 (t = 49.6)**; adding channel × category interactions adds only **+0.002 R²**,
meaning the penalty is uniform — not price- or category-specific.

**Not related to returns:**

| Factor | Correlation | p |
|---|---:|---|
| Season (`month_num`) | +0.002 | 0.980 |
| Temperature | −0.004 | 0.949 |
| Discount depth | −0.102 | 0.113 |
| Sale price | +0.054 | 0.401 |
| Unit volume | +0.078 | 0.227 |
| Ad spend | +0.228 | <0.001 (channel confound, see §11) |

Monthly mean return rates sit in a tight 0.162–0.181 band all year.

**The one real internal relationship is with ratings:** corr(return_rate, avg_rating) = **−0.353**
(p < 0.001), and it holds inside both channels (Online −0.271, Store −0.292). The fitted coefficient is
**−0.044 return-rate points per rating point (t = −9.5)** — half a star is worth about +2.2pp of
returns. Direction cannot be established from this data.

Business reading: roughly **1 in 4 online units comes back**, and none of it is explained by season,
price or discounting. It is a property of the channel itself.

## 13. What is related to ratings?

Ratings are tightly clustered and roughly normal around 4.2 (only 0.1 steps, range 3.4–5.0, sd 0.282).

| Comparison | Result |
|---|---|
| Store vs Online | **4.288 vs 4.149**, Welch t = −3.9, p < 0.001 |
| corr(rating, return_rate) | **−0.353** (p < 0.001) |
| corr(rating, units_sold) | +0.132 (p = 0.039) — weak |
| corr(rating, sale_price) | −0.014 (p = 0.828) |
| corr(rating, discount_pct) | −0.013 (p = 0.845) |
| corr(rating, ad_spend) | −0.085 (p = 0.188) |
| corr(rating, avg_temp_F) | +0.004 (p = 0.954) |

Category means are all within 0.07 of each other (Accessories 4.252 … Denim 4.183) against a standard
deviation of 0.282 — not a meaningful difference. The channel gap is not significant within any single
category (p = 0.059–0.183) and only emerges pooled.

Critically, the channel gap **collapses once you match on return rate**:

| Return-rate band | Online n / rating | Store n / rating |
|---|---|---|
| 0.00–0.10 | — | 67 / 4.33 |
| 0.10–0.15 | — | 29 / 4.26 |
| 0.15–0.20 | 18 / 4.37 | 23 / 4.23 |
| 0.20–0.25 | 51 / 4.11 | 1 / 3.90 |
| 0.25–0.30 | 33 / 4.16 | — |
| 0.30–0.36 | 18 / 4.03 | — |

Matched at 15–25% returns: Online **4.18** vs Store **4.21** — a 0.04 gap versus the raw 0.14. So
Online's lower rating is largely a *reflection of its higher return rate*, not a separate quality
deficit. Within Online, ratings also fall from 4.37 to 4.03 as returns rise from 15–20% to 30%+.

## 14. Explanatory power at a glance

| Question | Factor | Strength of evidence |
|---|---|---|
| Units sold | Discount depth | r = 0.513; +13.3/pp (t = 9.2); +10.6/pp with month dummies (t = 6.5); full model R² = 0.485 |
| Units sold | Temperature — Outerwear | r = −0.771, R² = 0.595; slope −16.3/°F (interaction t = −10.0) |
| Units sold | Temperature — Dresses | r = +0.621, R² = 0.386; slope +18.6/°F (t = 7.5) |
| Units sold | Channel | +106 to +115 units/cell online once price tier is held constant (t ≈ 3); raw gap 74 units not significant (t = 1.6, p = 0.101) |
| Revenue | Calendar month | η² = 0.084 pooled; 0.888 Outerwear, 0.783 Dresses |
| Revenue | Ad spend | r = 0.208, R² = 0.043; elasticity 0.175; no diminishing returns |
| Revenue | Year (2024 vs 2025) | η² = 0.004; +6.4%, of which 114% from volume |
| Returns | Channel | 0.247 vs 0.100 (t = 25.0); channel + category R² = 0.931; uniform across categories |
| Returns | Rating | r = −0.353; −0.044 per rating point (t = −9.5) |
| Returns | Season, discount, price, volume | No relationship (all p > 0.10) |
| Ratings | Return rate | r = −0.353; model R² = 0.332 |
| Ratings | Channel | Raw −0.139, falls to −0.04 when matched on returns |

## 15. Caveats — which parts look generated

Several columns behave like synthetic data rather than measurements. Treat conclusions about them as
descriptions of this file:

1. **`ad_spend`** — values are all multiples of 10, range similar in every category and channel,
   explains 4% of revenue, shows constant elasticity rather than diminishing returns, and produces
   ROAS from 1.4× to 38×.
2. **`return_rate`** — near-uniform *within* each channel (observed sd 0.045/0.046 versus 0.055/0.058
   for a uniform band), with the channel split as the only structure. It looks like random noise inside
   a channel-specific interval.
3. **`sale_price`** — carries a ±10% uniform random term, which produces 21 above-list prices, makes
   "0% discount" rows deviate from list, and weakens the link between discount and realised price
   across categories (corr −0.27).
4. **`revenue`** is fully derived from `sale_price × units_sold`, so it adds no information.
5. **`avg_temp_F` (24 values) and `list_price` (5 values)** are monthly/category attributes joined onto
   rows, not per-row measurements.
6. `avg_rating` is drawn in 0.1 steps and clustered in 3.4–5.0; category and channel differences are
   tiny relative to its spread.

The **discount → units** relationship, the **opposing seasonal patterns**, the **channel return gap**
and the **volume-led 2025 growth** are internally coherent and consistent across subgroups, so they
look like intended signal. The rating ↔ return link is consistent too, but it is a correlation between
two joint outcomes; direction is undetermined.

## 16. Method and reproduction

- All figures computed directly from `fashion_sales.csv`; correlations are Pearson, significance from
  t-tests (Welch for group comparisons), and models are OLS with standard errors.
- Significance reference: with n = 240, |r| > 0.13 is p < 0.05; within one category (n = 48),
  |r| > 0.28.
- Charts in `charts/` (10 PNGs, 1400×850) cover monthly revenue by category, Online vs Store revenue,
  units vs discount, units vs temperature by category, revenue vs ad spend, return rate by category and
  channel, revenue and units by category, total monthly revenue, and rating by category and channel.
