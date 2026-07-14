# IT Helpdesk Ticket Analytics — Capstone Part 1: Data Acquisition, Cleaning & EDA


## Dataset description

**Source:** FP20 Analytics Challenge 8 — IT Help Desk Analysis (Tickets and Agents, joined on `Agent ID`). The source data as-is is actually 0% null and has zero duplicates, so for this exercise I intentionally introduced realistic missing values and duplicate rows to give Tasks 2 and 3 something genuine to find and fix. What exactly got changed and why is written up in `NOTE_on_joined_with_issues.md`.

**Why this dataset:** It's built around a real operational question — how do you speed up ticket resolution and keep customers happy — which is exactly the kind of internal analytics tool a company would actually build for its own support team. The relationships in here aren't randomly generated either: `Request Category` has a big, consistent effect on how long a ticket takes, and `Priority_level` correlates with resolution speed in a way that makes intuitive sense.

**Loaded shape:** 98,960 rows × 15 columns (97,498 original tickets plus 1,462 rows I duplicated on purpose).

**Columns:**
| Column | Type | Description |
|---|---|---|
| `ID Ticket` | text | Unique ticket identifier |
| `Ticket Date` | date | Date the ticket was raised |
| `Employee ID` | numeric | ID of the employee who filed the ticket |
| `Agent ID` | numeric | ID of the assigned support agent (null if unassigned) |
| `Request Category` | categorical | System / Login Access / Software / Hardware |
| `Issue Type` | categorical | IT Request vs IT Error |
| `Severity` | categorical (ordinal in text) | e.g. "2 - Normal", "4 - Urgent" |
| `Priority` | categorical (ordinal in text) | e.g. "3 - High" (null if not yet triaged) |
| `Resolution Time (Days)` | numeric | Days to resolve the ticket |
| `Satisfaction Rate` | numeric | Customer satisfaction, 1–5 (null if not yet rated) |
| `Full Name`, `Email` | text | Assigned agent's identity (null if unassigned) |
| `Year/Month/Day of Birth` | numeric | Agent's date of birth (null if unassigned) |

---

## Task 1 — Load

Loaded straight from the pre-joined `data/joined_with_issues.csv`, with `parse_dates=["Fecha"]`, then renamed `Fecha` to `Ticket Date`. Shape came out to 98,960 rows × 15 columns, as expected.

---

## Task 2 — Null value analysis

| Column | Null % |
|---|---|
| `Agent ID`, `Full Name`, `Email`, `Year/Month/Day of Birth` | 25.12% each |
| `Satisfaction Rate` | 13.02% |
| `Priority` | 7.01% |
| all others | 0.00% |

**Columns over 20% null:** `Agent ID`, `Full Name`, `Email`, `Year of Birth`, `Month of Birth`, `Day of Birth` — all sitting around 25.1%. They move together because they all trace back to the same root cause: roughly a quarter of tickets haven't been assigned to an agent yet, so every field that depends on the agent is missing for those same rows. I left these alone rather than filling them — making up an agent identity for a ticket that genuinely doesn't have one yet wouldn't be cleaning the data, it'd be fabricating it.

**Numeric column under 20% null:** `Satisfaction Rate` at 13.02%, filled with the median (5.0).

Why median instead of mean here? The rating scale only runs 1 to 5, and it's heavily stacked toward the top — most people rate 4 or 5, with only a small minority giving low scores. That skews things negatively (confirmed later in Task 5), which drags the mean down below what a "typical" rating actually looks like. The median (5) reflects what most customers are actually saying far better than the mean (4.10) does, so it's the safer choice for filling in the roughly 13% of tickets that haven't been rated yet.

`Priority` at 7.01% null is categorical, not numeric, so the median-fill rule doesn't apply — I just left it as a genuine "hasn't been triaged" null.

---

## Task 3 — Duplicates

Found and removed 1,462 exact duplicate rows, which brought the dataset down from 98,960 to 97,498 rows — which, conveniently, lines up exactly with the true underlying ticket count, since these duplicates were rows that had simply been inserted a second time.

Did removing them change the null percentages? Yes, slightly — `Agent ID` and the fields tied to it moved from 25.12% to 23.999% null, and `Priority` moved from 7.01% to 6.999%. That's because the duplicated rows weren't a perfectly even cross-section of the dataset; they happened to contain a slightly different mix of assigned/unassigned tickets than the dataset as a whole, so removing them nudges the percentages a little. Columns that were already at 0% null obviously didn't move at all.

---

## Task 4 — Data type correction

**The problem:** `Priority` and `Severity` each cram a numeric ordinal code together with a text label into one object column — things like `"3 - High"` or `"2 - Normal"`. That leading number is genuinely useful numeric information for ordering, correlating, or doing math with, but it's stuck in a form you can't actually use as-is.

**How I fixed it:**
```python
df["Priority_level"] = df["Priority"].str.extract(r"^(\d+)")
df["Priority_level"] = pd.to_numeric(df["Priority_level"], errors="coerce")

df["Severity_level"] = df["Severity"].str.extract(r"^(\d+)")
df["Severity_level"] = pd.to_numeric(df["Severity_level"], errors="coerce")
```
Since `Priority` already has real nulls (about 7%), `Priority_level` naturally inherits that same missingness — `str.extract` on a `NaN` just returns `NaN` — which comes out to 6,824 nulls, 7.00%. Not a new problem, just an existing one passed through the transformation.

I also engineered `agent_age`, computed as the ticket's year minus the agent's year of birth. Since around 24% of tickets don't have an agent assigned yet (and therefore no birth date to pull from), `agent_age` picks up that same 24% missingness — 23,399 nulls. Again, that's a correctly propagated null, not something I patched over with a made-up value.

For the category conversion, `Request Category`, `Issue Type`, `Priority`, and `Severity` all got switched from `object` to `category` dtype.

**Memory usage:**
- Before: 48.15 MB
- After: 28.34 MB
- Saved 19.81 MB, about a 41.1% reduction

---

## Task 5 — Descriptive statistics & skewness

| Column | Skew |
|---|---|
| `Severity_level` | 1.70 |
| `Satisfaction Rate` | -1.67 |
| `Resolution Time (Days)` | 0.85 |
| `Priority_level` | -0.11 |
| `Agent_age` | -0.03 |

The most skewed column is `Severity_level`, at roughly 1.70.

What's driving that: about 91% of all tickets get tagged as `"2 - Normal"` severity, with everything else thinly spread across Minor, Major, Urgent, and Unclassified. That kind of extreme pile-up around one value, with a long thin tail stretching toward the higher severity codes, is exactly what produces a strong positive skew. Practically speaking, this means severity as it's currently recorded doesn't discriminate much between tickets — most of them look identical on this field — which is something to keep in mind for feature selection in Part 2.

`Satisfaction Rate` isn't far behind at -1.67 (actually a bit more skewed than before, since the median fill nudged a few more values toward 5) — most customers give the top score, with a thinner tail trailing down toward 1. `Resolution Time (Days)` is moderately right-skewed at 0.85.

---

## Task 6 — Outlier detection (IQR)

| Column | Q1 | Q3 | IQR | Lower bound | Upper bound | Outliers |
|---|---|---|---|---|---|---|
| `Resolution Time (Days)` | 0.0 | 7.0 | 7.0 | -10.50 | 17.50 | 258 rows (0.26%) |
| `Satisfaction Rate` | 4.0 | 5.0 | 1.0 | 2.50 | 6.50 | 10,379 rows (10.65%) |

`Resolution Time (Days)` barely has any statistical outliers — the IQR method only flags tickets taking longer than 17.5 days, which is a small enough group that it's worth actually looking into individually as possible process failures. `Satisfaction Rate`'s 10.65% outlier rate looks more dramatic than it really is — it's mostly an artifact of running IQR on a narrow, bounded 1-5 scale. Since most ratings cluster tightly at 4-5, the IQR itself only spans 1 point, so any rating of 1 or 2 automatically gets flagged even though it's a completely valid, if unhappy, response — not a data error.

Going forward into Part 2, I'll cap `Resolution Time (Days)` outliers at the IQR upper bound before regression modeling, so a handful of extremely long-running tickets don't skew the fit. `Satisfaction Rate`'s "outliers" I'm keeping as-is — they're genuine low ratings, and that's exactly the kind of signal a satisfaction model needs to actually learn from, not noise to filter out.

---

## Task 7 — Visualizations

### 7a. Line plot — monthly ticket volume over time
![Line plot](plots/1_line_ticket_volume.png)
Ticket volume climbs steadily from 2016 through 2020, which fits with a growing employee base driving more IT support demand over time — this isn't a flat or randomly bouncing series.

### 7b. Bar chart — average resolution time by request category
![Bar chart](plots/2_bar_resolution_by_category.png)
This is the strongest pattern in the whole dataset: Hardware tickets average 7.6 days to resolve, System tickets 6.6 days, Software 5.2 days, while Login Access tickets wrap up in just 0.3 days. Makes sense — login issues are probably handled through fast, mostly automated password/access resets, while hardware problems need physical intervention, parts, or coordinating with a vendor.

### 7c. Histogram — most skewed column (`Severity_level`)
![Histogram](plots/3_histogram_most_skewed.png)
Strongly concentrated at a single value (2 = Normal), with just thin bars showing up at 0, 1, 3, and 4. It's a sharp spike rather than a smooth tail, which matches the skew number from Task 5.

### 7d. Scatter plot — priority level vs resolution time
![Scatter plot](plots/4_scatter_priority_resolution.png)
There's a negative relationship here — higher-priority tickets tend to resolve faster — and it's real, if moderate (Pearson r is about -0.20, see Task 8). It's a pretty noisy relationship though, so priority level on its own is a real but incomplete predictor of how fast something gets resolved.

### 7e. Box plot — resolution time by request category
![Box plot](plots/5_box_resolution_by_category.png)
This backs up the bar chart but adds the spread detail: Login Access tickets aren't just fast on average, nearly all of them resolve within a day or two — a tight box sitting right near zero. Hardware and System tickets have both a higher median and a much wider spread, meaning they're slower and a lot less predictable.

---

## Task 8 — Correlation heat map

![Heatmap](plots/6_correlation_heatmap.png)

The strongest correlation is between `Resolution Time (Days)` and `Priority_level`, at about -0.200.

Is this causal, or is something else going on? This one's probably closer to actual causation than most correlations turn out to be — higher-priority tickets are, by design, supposed to get escalated and routed faster, so priority directly affecting resolution time is a pretty reasonable mechanism, not just coincidence. That said, `Request Category` is worth naming as a possible confounder — if Login Access tickets (already the fastest category by a mile) also tend to get tagged with higher priority on average, then category could be doing more of the actual work here than priority itself gets credit for. Worth teasing apart with feature importance once we're in Part 2.

One relationship that's notably missing: `Satisfaction Rate` shows basically zero correlation with anything else in the numeric columns, including resolution time. That's a bit surprising — it suggests customers aren't rating their satisfaction mainly based on how quickly their ticket got solved, which pushes back on an assumption a lot of support teams probably make by default.

---

## Task 9a — Imputation strategy comparison

The two highest-|skew| numeric columns are `Severity_level` (skew ≈ 1.70) and `Satisfaction Rate` (skew ≈ -1.67).

| Column | Mean | Median |
|---|---|---|
| `Severity_level` | 2.048 | 2.000 |
| `Satisfaction Rate` | 4.099 | 5.000 |

Both of these numbers are the true pre-imputation statistics. `Satisfaction Rate` had already gotten median-filled back in Task 2 since it's a numeric column under 20% null, so to actually honor this task's "before any imputation" requirement, I pulled these values from a snapshot I took right after Task 1, before any cleaning had touched the column.

I went with the median for both columns, which fits the general rule that skewed data is better represented by the median than the mean. For `Severity_level`, the median (2) captures the overwhelming "Normal" tag much better than the mean (2.05, which gets nudged up slightly by the minority of Urgent/Major tickets). Same story for `Satisfaction Rate` — the median (5) reflects that most customers give the top score, while the mean (4.10) gets dragged down by the smaller group of low ratings.

Checked with `isnull().sum()` afterward and got 0 nulls in both columns — which was already true for `Satisfaction Rate` since Task 2, and `Severity_level` never had any nulls to begin with since `Severity` itself is fully populated.

---

## Task 9b — Spearman rank correlation

Top 3 pairs by |Spearman − Pearson| difference, out of the 10 possible pairs across the 5 numeric columns:

| Pair | Pearson | Spearman | \|Difference\| |
|---|---|---|---|
| `Resolution Time` – `Priority_level` | -0.200 | -0.143 | 0.057 |
| `Resolution Time` – `Severity_level` | -0.041 | -0.017 | 0.024 |
| `Satisfaction Rate` – `Severity_level` | 0.003 | 0.014 | 0.011 |

For `Resolution Time`–`Priority_level`, Pearson (0.200) actually comes out bigger than Spearman (0.143), which tells me this relationship leans closer to linear/proportional than curved — resolution time drops off fairly steadily as priority goes up, rather than in big jumps concentrated at one end of the scale. So Pearson is the more appropriate lens for this particular pair.

For the other two pairs — `Resolution Time`–`Severity_level` and `Satisfaction Rate`–`Severity_level` — both differences are small and both correlations are weak to begin with, so there's no meaningful relationship, linear or otherwise, between severity coding and either outcome. That backs up what Task 5 already suggested: severity, as it's currently recorded, doesn't carry much standalone predictive signal.

For Part 2 feature selection, I'll lean on Pearson specifically for `Priority_level`, since its relationship with resolution time is if anything slightly more linear than purely monotonic — Spearman isn't uncovering extra hidden structure the way it would if the relationship were genuinely curved. I'll still compute both routinely as a sanity check before including any feature in the actual models though.

---

## Task 9c — Grouped aggregation

Grouped `Resolution Time (Days)` by `Request Category`:

| Category | Mean | Std | Count |
|---|---|---|---|
| Hardware | 7.63 | 4.35 | 9,733 |
| System | 6.62 | 4.02 | 39,002 |
| Software | 5.24 | 3.49 | 19,570 |
| Login Access | 0.31 | 0.71 | 29,193 |

Highest mean: Hardware, at 7.63 days. Highest std: also Hardware, at 4.35 days — so it's both the slowest category and the least predictable one. The ratio between the highest and lowest group means comes out to 7.63 / 0.31, roughly 24.3x.

**(a)** Hardware topping both metrics makes sense given that hardware problems tend to hinge on things outside anyone's direct control — parts availability, vendor response times, physical repair scheduling — in a way that software or login issues just don't.

**(b) Is that high within-group std a problem for modeling?** Yes, especially for Hardware and System — their standard deviations (4.35 and 4.02 days) are large relative to their own means, which means just knowing "this is a Hardware ticket" only gets you partway to a good resolution-time estimate. You'd need more features layered on top — priority, which agent it's assigned to, maybe eventually a complexity signal pulled from ticket text in Part 4 — to actually narrow things down.

**(c) Does that 24.3x ratio suggest real predictive signal?** Definitely — a 24x gap between the fastest and slowest categories is a massive, unambiguous effect that's nowhere near what you'd expect from noise alone. `Request Category` is clearly one of the strongest, most useful features in this dataset for predicting resolution time going into Part 2.

---

## Files in this repository

- `data_prep_eda.ipynb` — notebook with all Part 1 tasks
- `data/joined_with_issues.csv` — raw data
- `cleaned_data.csv` — cleaned, feature-engineered output, used in Parts 2 and 3
- `plots/` — all five required visualizations plus the correlation heat map
- `README.md` — this file

## How to Reproduce

### 1. Clone the repository

```bash
git clone https://github.com/Jayaprada6/Part4.git
cd Part4
```

### 2. Install the required packages

```bash
pip install pandas numpy matplotlib seaborn scikit-learn joblib
```

If your notebook uses additional libraries such as `python-dotenv`, `requests`, or `openai`, install those too:

```bash
pip install python-dotenv requests
```

### 3. Launch Jupyter Notebook

```bash
jupyter notebook
```

or

```bash
jupyter lab
```

### 4. Open the notebook

Open `data_prep_eda.ipynb` (or swap in your notebook's actual filename).

### 5. Run all cells

Run top to bottom to reproduce the full analysis and all outputs. This regenerates `cleaned_data.csv` and every plot in `plots/` from `data/joined_with_issues.csv`.
