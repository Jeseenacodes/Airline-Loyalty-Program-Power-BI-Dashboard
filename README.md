# ✈️ Airline Loyalty Program — Power BI Dashboard

A complete end-to-end Power BI analytics solution for an airline loyalty program, built with a clean semantic model, 38 DAX measures organized across 5 display folders, and a 5-page interactive dashboard designed for both executive and analyst audiences.

---

## Dashboard Preview

| Page | Purpose |
|---|---|
| **Overview** | Executive KPI strip, member growth trend, tier distribution, province breakdown |
| **Membership Trends** | Enrollment vs cancellation trends, seasonality, running total |
| **Segmentation** | Tier, education, marital status, and geographic analysis |
| **Flight & Points** | Flight activity YOY, points accumulation vs redemption gap |
| **Performance Trends** | Province × tier breakdown, annual performance, CLV × education matrix |

---

## Key Findings

> **1.54% points redemption rate** - 796.5M points accumulated, only 12.3M ever redeemed. Members are earning but not engaging. This is a financial liability and a leading indicator of churn.

| Metric | Value |
|---|---|
| Total active members | 14,670 |
| Total enrolled (all time) | 16,737 |
| Retention rate | 87.7% |
| Cancellation rate | 12.3% |
| Total flights booked | 508,808 |
| Total distance flown | 762.9M km |
| Total points accumulated | 796.5M |
| Total points redeemed | 12.3M |
| Points redemption rate | 1.54% ⚠️ |
| Avg customer lifetime value | $7,989 |
| Total CLV portfolio | $133.7M |
| Avg flights per member | 34.7 |

---

## 🗂️ Semantic Model

### Tables

```
📁 Customer Loyalty History    [Fact + Quasi-Dimension]
📁 Customer Flight Activity    [Fact]
📁 Calendar                    [Date Dimension]
📁 DAX Measures                [Calculation Island]
📁 Measures                    [Legacy Calculation Island]
```

### Relationships

```
Customer Flight Activity[Loyalty Number]
    ──► Customer Loyalty History[Loyalty Number]          ✅ Active

Customer Loyalty History[Enrollment Start of Month]
    ──► Calendar[Date]                                    ✅ Active

Customer Loyalty History[Cancellation Start of Month]
    ──► Calendar[Date]                                    ❌ Inactive (USERELATIONSHIP)

Customer Flight Activity[Flight Booked Start of Month]
    ──► Calendar[Date]                                    ❌ Inactive (USERELATIONSHIP)
```

> The two inactive relationships use the **role-playing dimension pattern** — activated on demand via `USERELATIONSHIP()` inside individual DAX measures. This avoids duplicating the Calendar table.

### Key Columns

| Table | Column | Role |
|---|---|---|
| Customer Loyalty History | `Loyalty Number` | Member identity key — joins to Flight Activity |
| Customer Loyalty History | `Enrollment Start of Month` | Active date relationship to Calendar |
| Customer Loyalty History | `Cancellation Start of Month` | Inactive date relationship (USERELATIONSHIP) |
| Customer Loyalty History | `Loyalty Card` | Tier dimension (Star / Nova / Aurora) |
| Customer Loyalty History | `Has cancelled` | Boolean flag for active member filter |
| Customer Loyalty History | `CLV` | Customer Lifetime Value metric |
| Customer Loyalty History | `Salary` | Member income — demographic segmentation |
| Customer Flight Activity | `Total Flights` | Flight count per member per month |
| Customer Flight Activity | `Distance` | Route distance |
| Customer Flight Activity | `Points Accumulated` | Loyalty points earned |
| Customer Flight Activity | `Points Redeemed` | Loyalty points spent |
| Customer Flight Activity | `Dollar Cost Points Redeemed` | Monetary value of redemptions |
| Customer Flight Activity | `Flight Booked Start of Month` | Inactive date relationship (USERELATIONSHIP) |
| Calendar | `Date` | Primary date key |
| Calendar | `Start of Month` | Month-level grouping axis |
| Calendar | `Start of Year` | Year-level grouping axis |

---

## Notable DAX Debugging

### The Cancellation Rate Bug

**Symptom:** `Cancellation Rate` returned 100%, `Retention Rate` returned 0%.

**Root cause:** The original measure used `USERELATIONSHIP` to activate the cancellation date relationship, but without a date filter in context and no blank guard, it joined every member row — including nulls — producing a count equal to total enrollments.

**Fix:**

```dax
-- ❌ Before (broken)
CALCULATE(
    [DAX Loyalty Member Enrollments],
    USERELATIONSHIP(
        'Customer Loyalty History'[Cancellation Start of Month],
        'Calendar'[Date]
    )
)

-- ✅ After (correct)
CALCULATE(
    DISTINCTCOUNT('Customer Loyalty History'[Loyalty Number]),
    USERELATIONSHIP(
        'Customer Loyalty History'[Cancellation Start of Month],
        'Calendar'[Date]
    ),
    NOT ISBLANK('Customer Loyalty History'[Cancellation Start of Month])
)
```

**Result:** Cancellation Rate corrected from 100% → **12.3%**. Retention Rate corrected from 0% → **87.7%**.

---

### The Total Flights Booked Naming Bug

**Symptom:** `Total Flights Booked` was summing `Points Redeemed` instead of `Total Flights`.

**Fix:**
```dax
-- ❌ Before
SUM('Customer Flight Activity'[Points Redeemed])

-- ✅ After
SUM('Customer Flight Activity'[Total Flights])
```

---

## Spike Annotation Pattern

To highlight the 2018 promotion spike on line charts without a separate visual, two measures return a value only for the target year and `BLANK()` for all others. This places a precise dot on just that data point.

```dax
-- Enrollment Spike 2018
IF(
    SELECTEDVALUE('Calendar'[Start of Year]) = DATE(2018, 1, 1),
    [DAX Loyalty Member Enrollments],
    BLANK()
)

-- Enrollment Avg Baseline (flat reference line — 2013–2017 average)
CALCULATE(
    AVERAGEX(
        VALUES('Calendar'[Start of Year]),
        [DAX Loyalty Member Enrollments]
    ),
    FILTER(
        ALL('Calendar'[Start of Year]),
        'Calendar'[Start of Year] >= DATE(2013, 1, 1)
            && 'Calendar'[Start of Year] < DATE(2018, 1, 1)
    )
)
```

In Power BI: set the spike series to **no line + large marker (size 12)** and data labels **On** for that series only. The `BLANK()` return means only 2018 gets a dot and label — all other years are invisible.

---

## Theme & Color Guide

| Color | Hex | Usage |
|---|---|---|
| Blue | `#2E9BFF` | Primary — enrollments, Star tier, flight lines |
| Green | `#27D98F` | Positive — active members, retention, growth |
| Gold | `#F5A623` | Warning — Nova tier, financial metrics, spike annotation |
| Red | `#FF4D6A` | Alert — cancellations, redemption rate, churn |
| Purple | `#9B6BFF` | Secondary — Aurora tier, points, CLV |
| Dark bg | `#0B0F1A` | Report canvas background |
| Surface | `#131929` | Card and panel background |
| Border | `#1F2E47` | Gridlines and card borders |

```
Star   → Blue   #2E9BFF
Nova   → Gold   #F5A623
Aurora → Purple #9B6BFF
```

---

## Slicer Architecture

### Synced across all pages
| Slicer | Field | Type |
|---|---|---|
| Year | `Calendar[Start of Year]` | Dropdown |
| Loyalty Tier | `Customer Loyalty History[Loyalty Card]` | Tile (button) |
| Province | `Customer Loyalty History[Province]` | Dropdown |

### Detail Tables page only
| Slicer | Field | Type |
|---|---|---|
| Province search | `Customer Loyalty History[Province]` | Text search |
| Year | `Calendar[Start of Year]` | Dropdown (also synced) |
| Loyalty Tier | `Customer Loyalty History[Loyalty Card]` | Tile (also synced) |

> To sync slicers: **View tab → Sync slicers → check both Sync ✅ and Visible ✅ for each page**. For Detail Tables, check Sync ✅ but optionally uncheck Visible ❌ to inherit filters silently.

---

## Segmentation Insights

### By Loyalty Tier

| Tier | Enrolled | Active | Avg CLV | Churn |
|---|---|---|---|---|
| ⭐ Star | 7,637 | 6,736 | $6,742 | 11.8% |
| 🌟 Nova | 5,671 | 4,954 | $8,046 | 12.6% |
| 🔮 Aurora | 3,429 | 2,980 | $10,673 | 13.1% |

> Aurora members generate **58% more CLV** than Star members — but have the highest churn. Protect and retain Aurora tier as a priority.

### By Province (Top 6)

| Province | Active Members | Avg CLV |
|---|---|---|
| Ontario | 4,730 | $7,914 |
| British Columbia | 3,887 | $7,994 |
| Quebec | 2,887 | $8,161 |
| Alberta | 847 | $7,753 |
| New Brunswick | 569 | $8,154 |
| Manitoba | 558 | $8,067 |

> Ontario, BC, and Quebec = **68% of all active members** — geographic concentration risk.

### By Marital Status

| Status | Flights | Avg CLV | Avg Salary |
|---|---|---|---|
| Married | 295,540 | $8,058 | $78,792 |
| Single | 137,408 | $7,719 | $76,399 |
| Divorced | 75,860 | $8,201 | $83,739 |

> Divorced members have the highest CLV and salary despite flying less — a high-yield segment for targeted campaigns.

### By Education

| Education | Enrolled | Avg CLV |
|---|---|---|
| Bachelor | 10,475 (62.6%) | $8,207 |
| College | 4,238 (25.3%) | $7,595 |
| High school or below | 782 (4.7%) | $7,707 |
| Doctor | 734 (4.4%) | $7,833 |
| Master | 508 (3.0%) | $7,441 |

---

## Year-over-Year Enrollment Trend

| Year | Enrollments | Cancellations | Net |
|---|---|---|---|
| 2012 | 1,686 | — | 1,686 |
| 2013 | 2,397 | 43 | 2,354 |
| 2014 | 2,370 | 181 | 2,189 |
| 2015 | 2,331 | 265 | 2,066 |
| 2016 | 2,456 | 427 | 2,029 |
| 2017 | 2,487 | 506 | 1,981 |
| **2018** | **3,010** ⬆️ | **645** | **2,365** |

> 2018 promotion drove **+25% enrollments** above the 2013–2017 average of 2,408. Cancellations also peaked at 645 — **+27% above 2017**.

---

## Recommended Next Steps

| Enhancement | Description |
|---|---|
| **Predictive churn model** | Python/R integration to score each member's cancellation probability based on flight recency, points age, and tier |
| **What-if parameter** | Simulate revenue impact of improving redemption rates by 5%, 10%, 15% |
| **Row-level security** | Province-level RLS so regional managers see only their data |
| **Redemption campaign tracker** | Measure impact of targeted campaigns on redemption rate over time |
| **CLV forecasting** | Project CLV trajectory by tier using historical flight and points data |

---

## Tools & Technologies

![Power BI](https://img.shields.io/badge/Power%20BI-F2C811?style=flat&logo=powerbi&logoColor=black)
![DAX](https://img.shields.io/badge/DAX-0078D4?style=flat&logo=microsoft&logoColor=white)
![Data](https://img.shields.io/badge/Power%20BI%20Modeling-6B4FBB?style=flat)

- **Power BI Desktop** — report authoring and semantic model
- **DAX** — 38 custom measures across 5 display folders
- **Tabular model** — compatibility level 1600, star schema design

---
## Screenshots

<img width="1241" height="752" alt="image" src="https://github.com/user-attachments/assets/ddc76d92-dae0-4752-b6a7-cbf2311c84e6" />

## Repository Structure

```
airline-loyalty-dashboard/
│
├── README.md                          ← This file
├── AirlineLoyaltyDashboard.pbix       ← Power BI report file
│
├── dax/
│   ├── flight-activity.md             ← Flight Activity folder measures
│   ├── points.md                      ← Points folder measures
│   ├── loyalty-members.md             ← Loyalty Members folder measures
│   ├── time-intelligence.md           ← Time Intelligence folder measures
│   └── demographics.md               ← Customer Demographics folder measures
│
├── docs/
│   ├── model-diagram.png              ← Semantic model relationship diagram
│   ├── dashboard-overview.png         ← Overview page screenshot
│   ├── dashboard-trends.png           ← Membership Trends page screenshot
│   ├── dashboard-segments.png         ← Segmentation page screenshot
│   └── color-theme.json              ← Power BI JSON theme file
│
└── theme/
    └── AirlineLoyaltyDark.json        ← Dark canvas Power BI theme
```

---

## Power BI Theme File

Save as `AirlineLoyaltyDark.json` and import via **View → Themes → Browse for themes**:

```json
{
  "name": "Airline Loyalty Dark",
  "dataColors": [
    "#2E9BFF",
    "#F5A623",
    "#27D98F",
    "#FF4D6A",
    "#9B6BFF",
    "#3D4F6E"
  ],
  "background": "#0B0F1A",
  "foreground": "#131929",
  "tableAccent": "#2E9BFF",
  "visualStyles": {
    "*": {
      "*": {
        "background": [{"color": {"solid": {"color": "#131929"}}}],
        "title": [{"fontColor": {"solid": {"color": "#E8EDF5"}}}],
        "subTitle": [{"fontColor": {"solid": {"color": "#6B7FA3"}}}]
      }
    }
  }
}
```

---

*Built with Power BI Desktop · DAX · Power BI · 16,737 members · 508,808 flights · $133.7M CLV*

## Old Dashboard 
Linkedin post: https://www.linkedin.com/posts/activity-7455288564864577536-cIeF?utm_source=social_share_send&utm_medium=member_desktop_web&rcm=ACoAAF0HMosBCwWtZDjWY982g72-fhT3ov8dzKc 
<img width="1252" height="705" alt="image" src="https://github.com/user-attachments/assets/2344320a-efa9-46c3-90f9-0201004be1d6" />

<img width="1263" height="652" alt="image" src="https://github.com/user-attachments/assets/3c425459-8630-4514-9a3b-6bdc675f660f" />

<img width="879" height="597" alt="image" src="https://github.com/user-attachments/assets/adf165d4-13aa-4bcd-9810-1cfeae966fda" />

<img width="1182" height="672" alt="image" src="https://github.com/user-attachments/assets/7baa52e3-f6be-4a68-9f00-a5d708d3fc9f" />










