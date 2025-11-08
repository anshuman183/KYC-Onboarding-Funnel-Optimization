

# KYC Onboarding Funnel Optimization

## Objective
The goal of this analysis is to understand user drop-offs during the KYC onboarding funnel and evaluate an A/B experiment testing two ID verification flows to improve overall completion rates.

---

## 1. Dataset Overview

### a. `user_attributes`
Contains metadata about each user.

| Column | Description |
|---------|-------------|
| user_id | Unique user identifier |
| country | User’s country |
| device | Device used (`android`, `ios`, `web`) |
| signup_date | Date of registration |
| experiment_group | Experiment variant (`A`, `B`, or `NULL`) |

### b. `user_onboarding_logs`
Contains detailed onboarding logs.

| Column | Description |
|---------|-------------|
| user_id | Links to `user_attributes` |
| step_name | Onboarding step name |
| step_status | Step completion status (`completed`, `dropped`) |
| timestamp | Event time |
| experiment_group | Experiment variant |

### Funnel Steps
1. Signup  
2. Email Verification  
3. Profile Setup  
4. ID Verification  
5. Onboarding Complete  

**Dataset summary:**
- 50,000 total users  
- 10,000 experiment users (5,000 in Group A, 5,000 in Group B)  
- ~250,000+ timestamped events  

---

## 2. Methodology

### Step 1: Data Preparation
- Loaded both CSVs into SQL and Python (Pandas)  
- Cleaned missing experiment flags  
- Filtered `step_status = 'completed'` to track funnel progression  

---

### Step 2: Funnel Construction (SQL)
The first step was to identify the number of users who completed each stage of the onboarding process.

```sql
SELECT step_name, COUNT(DISTINCT user_id) AS users_completed
FROM user_onboarding_logs
WHERE step_status = 'completed'
GROUP BY step_name
ORDER BY FIELD(step_name, 'signup','email_verification','profile_setup','id_verification','onboarding_complete');
````

This query helped visualize user drop-offs across funnel stages such as Signup → Email Verification → ID Verification → Onboarding Complete.

---

### Step 3: Experiment Comparison

A comparison between the two experiment groups (A and B) was conducted to evaluate the impact of the new ID verification flow.
Distinct users who completed each step were counted per variant.

```sql
SELECT step_name,
       COUNT(DISTINCT CASE WHEN experiment_group = 'A' THEN user_id END) AS users_A,
       COUNT(DISTINCT CASE WHEN experiment_group = 'B' THEN user_id END) AS users_B
FROM user_onboarding_logs
WHERE experiment_group IS NOT NULL
  AND step_status = 'completed'
GROUP BY step_name
ORDER BY FIELD(step_name, 'signup','email_verification','profile_setup','id_verification','onboarding_complete');
```

This query was used to compare funnel conversion rates between Group A (control) and Group B (variant).

---

### Step 4: Statistical Validation (Z-Test for Proportions)

After computing the funnel completion rates for both experiment variants, a **two-proportion z-test** was performed to determine if the difference between Group A and Group B was statistically significant.

The z-statistic is calculated as:

z = (p1 - p2) / sqrt( p * (1 - p) * (1/n1 + 1/n2) )


Where:  
- `p1`, `p2` → conversion rates for Group A and Group B  
- `n1`, `n2` → total users in each group  
- `p` = (x1 + x2) / (n1 + n2) → pooled conversion rate  
- `x1`, `x2` → number of completions in Group A and Group B  

The resulting **z-score** and **p-value** determine whether the lift observed in Group B’s completion rate is statistically significant.  
If `p < 0.05`, the improvement in Group B’s funnel completion is considered **significant** at a 95% confidence level.

---

## 3. Tools and Libraries Used

* **SQL (MySQL/PostgreSQL)** – for funnel aggregation and experiment comparison
* **Python (Pandas, NumPy, SciPy)** – for data cleaning, preparation, and statistical testing
* **Matplotlib / Seaborn** – for funnel and conversion rate visualization

---

## 4. Key Insights (Example)

* Major user drop-off observed during **ID Verification** (~30% drop)
* Experiment **Group B** showed a **+7% improvement** in ID Verification completion rate
* Z-test results confirmed the improvement was **statistically significant (p < 0.05)**

---

## 5. Next Steps

* Further segment analysis by **device** and **country**
* Evaluate long-term retention of users who completed onboarding
* A/B test refinements for additional friction points in the funnel

---

## Author

Anshuman Gupta

Data Analyst | SQL | Experimentation | Product Insights

```

