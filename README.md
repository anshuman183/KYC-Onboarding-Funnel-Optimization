

# **KYC Onboarding Funnel Optimization – README**

## **Overview**

This project analyzes the **KYC onboarding funnel** for a fintech/crypto-style product and evaluates an **A/B experiment** comparing two ID verification flows. The objective is to identify funnel drop-offs, assess experiment performance, and validate statistical significance using a two-proportion Z-test.

---

# **1. Dataset Overview**

## **a. `user_attributes`**

Metadata for each user.

| Column             | Description                           |
| ------------------ | ------------------------------------- |
| `user_id`          | Unique user identifier                |
| `country`          | User’s country                        |
| `device`           | Device used (`android`, `ios`, `web`) |
| `signup_date`      | Date of registration                  |
| `experiment_group` | Variant `A`, `B`, or `NULL`           |

---

## **b. `user_onboarding_logs`**

Detailed timestamped logs of user onboarding events.

| Column             | Description                |
| ------------------ | -------------------------- |
| `user_id`          | Links to `user_attributes` |
| `step_name`        | Name of onboarding step    |
| `step_status`      | `completed` / `dropped`    |
| `timestamp`        | Event time                 |
| `experiment_group` | `A` / `B` / `NULL`         |

---

## **Funnel Steps**

1. Signup
2. Email Verification
3. Profile Setup
4. ID Verification
5. Onboarding Complete

**Dataset Summary**

* **50,000 total users**
* **10,000 experiment users** (5,000 in Group A + 5,000 in Group B)
* **~250,000 timestamped events**

---

# **2. Methodology**

## **Step 1: Data Preparation**

* Loaded CSVs into SQL and Pandas
* Cleaned missing experiment flags
* Filtered only `step_status = 'completed'` events for funnel calculation

---

## **Step 2: Funnel Construction (SQL)**

```sql
SELECT step_name, COUNT(DISTINCT user_id) AS users_completed
FROM user_onboarding_logs
WHERE step_status = 'completed'
GROUP BY step_name
ORDER BY FIELD(step_name, 
    'signup',
    'email_verification',
    'profile_setup',
    'id_verification',
    'onboarding_complete');
```

This query captures user drop-offs across all funnel stages.

---

## **Funnel Completion Metrics**

| Funnel Stage            | Users Completed | Stage Conversion % | Overall Conversion % |
| ----------------------- | --------------- | ------------------ | -------------------- |
| **signup**              | 50,000          | —                  | 100%                 |
| **email_verification**  | 23,917          | **47.83%**         | 47.83%               |
| **profile_setup**       | 11,746          | **49.12%**         | 23.49%               |
| **id_verification**     | 5,750           | **48.95%**         | 11.50%               |
| **onboarding_complete** | 2,885           | **50.17%**         | 5.77%                |

---

## **Key Insights**

### **1. Largest Drop-Off: Signup → Email Verification**

* 52% users drop off before verifying email
* Indicates possible:

  * Email deliverability issues
  * Friction in initial verification
  * Low user motivation

### **2. Mid-Funnel Consistency**

* From Email → ID Verification, each stage retains ~49–50%
* Typical for KYC-heavy flows

### **3. Overall Completion**

* Only **5.77%** reach “Onboarding Complete”
* Typical benchmark for regulated KYC flows (5–10%)

---

# **3. Experiment Comparison (A/B Test)**

SQL used:

```sql
SELECT step_name,
       COUNT(DISTINCT CASE WHEN experiment_group = 'A' THEN user_id END) AS users_A,
       COUNT(DISTINCT CASE WHEN experiment_group = 'B' THEN user_id END) AS users_B
FROM user_onboarding_logs
WHERE experiment_group IS NOT NULL
  AND step_status = 'completed'
GROUP BY step_name
ORDER BY FIELD(step_name, 
    'signup',
    'email_verification',
    'profile_setup',
    'id_verification',
    'onboarding_complete');
```

---

## **Experiment Results (A vs B)**

| Step Name               | Users A | Users B |
| ----------------------- | ------- | ------- |
| **signup**              | 5,000   | 5,000   |
| **email_verification**  | 2,250   | 2,817   |
| **profile_setup**       | 1,030   | 1,604   |
| **id_verification**     | 434     | 919     |
| **onboarding_complete** | 179     | 502     |

---

## **Key Insights**

### **1. Variant B outperforms A at every step**

* A completions: **179**
* B completions: **502**
* **B converts ~2.8× more users**

### **2. Strongest improvement at ID Verification**

* A → 434
* B → 919
* **+112% improvement**

### **3. Better overall funnel performance**

* A: **3.58% completion**
* B: **10.04% completion**
* **+6.5 percentage points improvement**

---

# **4. Statistical Validation — Two-Proportion Z-Test**

To check if Group B’s improvement is statistically significant.

### **Hypotheses**

Let:

* $p_1$ = true completion rate for Group A

* $p_2$ = true completion rate for Group B

* **Null Hypothesis ($H_0$):**
  $p_1 = p_2$

  > No difference between A and B.

* **Alternative Hypothesis ($H_1$):**
  $p_2 > p_1$

  > Group B improves completion rate.

* **Significance level:**
  $\alpha = 0.05$

---

## **Calculations**

### **Sample Proportions**

$$
\hat{p}_1 = \frac{179}{5000} = 0.0358
$$

$$
\hat{p}_2 = \frac{502}{5000} = 0.1004
$$


### **Pooled Proportion**

$$
\hat{p} = \frac{179 + 502}{5000 + 5000} = \frac{681}{10000} = 0.0681
$$

### **Standard Error**

The standard error (SE) is:

$$
SE = \sqrt{\hat{p}(1 - \hat{p})\left(\frac{1}{n_1} + \frac{1}{n_2}\right)}
$$

Using:

* $\hat{p} = 0.0681$
* $n_1 = n_2 = 5000$

We get:

$$
SE \approx 0.0050
$$

### **Z-Statistic**

The z-statistic is:

$$
z = \frac{\hat{p}_1 - \hat{p}_2}{SE}
$$

Substituting values:

$$
z = \frac{0.0358 - 0.1004}{0.0050} \approx -12.82
$$

### **P-Value**

For a one-sided test with $H_1: p_2 > p_1$:

$$
p\text{-value} = P(Z \leq -12.82) \approx 0
$$

---

# **Decision**

* Since **p-value ≪ 0.05**, **reject $H_0$**.
* The probability that this lift occurred by chance is effectively **zero**.

---

# **Conclusion**

* **Group B significantly outperforms Group A.**
* **The new ID verification flow should be rolled out.**
* **Statistical evidence strongly supports adopting Variant B.**


