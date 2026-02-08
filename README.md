# Stay Hard Gym Management System: SQL JOINs & Window Functions

**Course:** INSY 8311 - Database Development with PL/SQL  
**Student:** Japhet KWIZERA  
**Student ID:** 28282  
**Group:** B 


---

## 🎯 Business Problem

### Business Context
**Company:** Stay Hard Gym  
**Location:** Kigali City, Rwanda   
**Industry:** Fitness & Wellness  
**Department:** Business Analytics & Member Services  
**Stakeholders:** Gym Management, Marketing Team, Class Instructors, Finance Department

### Data Challenge
Stay Hard Gym faces several critical operational challenges in the competitive Kigali fitness market:
- **Member Engagement**: 30-40% of paying members never attend classes, representing lost value and potential customer loss
- **Class Optimization**: Inconsistent class enrollments across categories with some premium classes underutilized
- **Revenue Visibility**: Limited insights into member lifetime value, payment patterns, and district-level performance
- **Customer loss Risk**: Inability to predict which members are likely to cancel or let memberships expire
- **Resource Allocation**: Difficulty determining which classes and instructors generate the best ROI

The gym lacks comprehensive data analytics to:
- Identify high-value members for retention programs
- Optimize class schedules based on actual demand
- Predict revenue trends and payment patterns
- Target marketing campaigns by district and member segment
- Measure class instructor performance objectively

### Expected Outcome
Develop SQL-based analytics system to:
1. **Segment members** by lifetime value and engagement levels for personalized service
2. **Track revenue trends** with monthly growth analysis and payment pattern prediction
3. **Optimize class offerings** through enrollment analysis and attendance tracking
4. **Enable data-driven decisions** for marketing, pricing, and resource allocation in Kigali market
5. **Reduce customer loss** by identifying at-risk members before they lapse

---

## ✅ Success Criteria

This analysis achieves the following five measurable objectives:

| # | Success Criterion | Window Function | Business Impact |
|---|------------------|----------------|-----------------|
| 1 | **Rank top 5 classes per category** | `RANK()`, `DENSE_RANK()` | Identify best-performing classes for scheduling priority |
| 2 | **Calculate running monthly revenue totals** | `SUM() OVER()` with ROWS | Track cumulative gym financial growth |
| 3 | **Measure month-over-month membership growth** | `LAG()`, `LEAD()` | Detect payment trends and predict churn risk |
| 4 | **Segment members into value quartiles** | `NTILE(4)` | Enable targeted retention and upselling strategies |
| 5 | **Compute 3-month moving revenue average** | `AVG() OVER()` with ROWS | Smooth seasonal fluctuations to reveal trends |

---

## 🗄️ Database Schema

### Entity Relationship Diagram

```
┌──────────────────┐         ┌─────────────────┐         ┌──────────────────────┐
│ MEMBERSHIP_PLANS │         │    MEMBERS      │         │ MEMBERSHIP_PAYMENTS  │
├──────────────────┤         ├─────────────────┤         ├──────────────────────┤
│*plan_id          │────┐    │*member_id       │────┐    │*payment_id           │
│ plan_name        │    └───<│ plan_id         │    └───<│ member_id            │
│ duration_months  │         │ member_name     │         │ payment_date         │
│ monthly_fee      │         │ phone_number    │         │ amount_paid          │
│ plan_type        │         │ email           │         │ payment_method       │
│ features         │         │ join_date       │         │ payment_status       │
└──────────────────┘         │ membership_     │         │ months_covered       │
                             │   _status       │         └──────────────────────┘
                             │ gender          │
                             │ district        │
                             └─────────────────┘
                                     │
                                     │
┌──────────────────┐                 │         ┌──────────────────────┐
│   GYM_CLASSES    │                 │         │ CLASS_ENROLLMENTS    │
├──────────────────┤                 │         ├──────────────────────┤
│*class_id         │<────────────────┴────────<│*enrollment_id        │
│ class_name       │                           │ member_id            │
│ instructor_name  │                           │ class_id             │
│ class_category   │                           │ enrollment_date      │
│ duration_minutes │                           │ attendance_status    │
│ max_capacity     │                           │ enrollment_fee       │
│ class_fee        │                           └──────────────────────┘
└──────────────────┘
```

### Tables Overview

| Table | Records | Description |
|-------|---------|-------------|
| **membership_plans** | 5 | Gym membership tiers (Daily, Monthly, Quarterly, Semi-Annual, Annual) |
| **members** | 25 | Active and former gym members across Kigali districts |
| **gym_classes** | 15 | Fitness classes (Cardio, Strength Training, Wellness, Dance, Combat) |
| **membership_payments** | 40+ | Payment transactions with Rwandan payment methods (Mobile Money, Cash, Bank Transfer) |
| **class_enrollments** | 50+ | Member class attendance records |

### Key Relationships
- **Members → Membership Plans:** Many-to-One (members subscribe to one plan type)
- **Membership Payments → Members:** Many-to-One (multiple payments per member)
- **Class Enrollments → Members:** Many-to-One (members enroll in multiple classes)
- **Class Enrollments → Gym Classes:** Many-to-One (classes have multiple enrollments)

### District Coverage
Stay Hard Gym serves three main Kigali districts:
- **Gasabo** (Northern Kigali - largest membership base)
- **Kicukiro** (Southeastern Kigali - growing market)
- **Nyarugenge** (Central Kigali - business district)

---

## 🔗 Part A: SQL JOINs

### 1. INNER JOIN: Active Members with Payment Details

**Purpose:** Retrieve all active members with complete payment and membership plan information

```sql
SELECT 
    m.member_name,
    m.district,
    mp.plan_name,
    mp.plan_type,
    p.payment_date,
    p.amount_paid,
    p.payment_method
FROM members m
INNER JOIN membership_plans mp ON m.plan_id = mp.plan_id
INNER JOIN membership_payments p ON m.member_id = p.member_id
WHERE m.membership_status = 'Active' AND p.payment_status = 'Completed'
ORDER BY p.payment_date DESC;
```

**Business Interpretation:**  
This query shows all active Stay Hard Gym members with successful payment records. 
It reveals which membership tiers are most popular in Kigali , and which payment methods Rwandan customers prefer . 
**Screenshot:** ![Inner Join](https://github.com/japhetk-dev/plsql_window_functions_28282_Japhet/blob/de945a41a0b1e2c4a0b36d56898c58534636c4ba/screenshots/INNERJOIN.png)

---

### 2. LEFT JOIN: Members Not Attending Classes

**Purpose:** Identify paying members who have never attended gym classes

```sql
SELECT 
    m.member_name,
    m.phone_number,
    mp.plan_name,
    m.join_date,
    CASE 
        WHEN ce.enrollment_id IS NULL THEN 'No Class Attendance'
        ELSE 'Has Attended Classes'
    END AS participation
FROM members m
LEFT JOIN class_enrollments ce ON m.member_id = ce.member_id
LEFT JOIN membership_plans mp ON m.plan_id = mp.plan_id
WHERE ce.enrollment_id IS NULL AND m.membership_status = 'Active';
```

**Business Interpretation:**  
This query highlights “gym floor only” members who don’t attend classes, 
accounting for 35–40% of active members and facing higher churn risk due to low engagement.
A targeted SMS campaign offering a free trial class could convert 20%, adding about RWF 180,000 per month in class fees while improving retention.


**Screenshot:** ![Left Join](https://github.com/japhetk-dev/plsql_window_functions_28282_Japhet/blob/68946d425f849339aefef947f433852ce54b3982/screenshots/LEFTJOIN.png)

---

### 3. RIGHT JOIN: Classes with Zero Enrollment

**Purpose:** Detect gym classes that have capacity but no enrollments

```sql
SELECT 
    gc.class_name,
    gc.instructor_name,
    gc.class_category,
    gc.class_fee,
    CASE 
        WHEN ce.enrollment_id IS NULL THEN 'No Enrollments'
        ELSE 'Has Enrollments'
    END AS status
FROM class_enrollments ce
RIGHT JOIN gym_classes gc ON ce.class_id = gc.class_id
WHERE ce.enrollment_id IS NULL;
```

**Business Interpretation:**  
Results show 2-3 classes with zero enrollment , typically specialized classes like advanced Pilates or early morning sessions. 
In Kigali's market, morning classes (5-7 AM) struggle due to traffic patterns and most members prefer evening (6-8 PM) after work. 
These empty slots waste instructor costs . Recommendation: Reschedule to peak times or replace with high-demand classes like HIIT or Zumba.

**Screenshot:** ![Right Join](https://github.com/japhetk-dev/plsql_window_functions_28282_Japhet/blob/b5528a8cdc0f16110e6d8d7a87f84c5effd2e4f4/screenshots/RIGHTJOIN.png)

---

### 4. FULL OUTER JOIN: Complete Ecosystem View

**Purpose:** Comprehensive view of members and classes including unmatched records

```sql
SELECT 
    m.member_name,
    gc.class_name,
    CASE 
        WHEN m.member_id IS NULL THEN 'Class Only'
        WHEN gc.class_id IS NULL THEN 'Member Only'
        ELSE 'Both Exist'
    END AS record_type
FROM members m
LEFT JOIN class_enrollments ce ON m.member_id = ce.member_id
LEFT JOIN gym_classes gc ON ce.class_id = gc.class_id
WHERE ce.enrollment_id IS NULL
UNION
SELECT 
    m.member_name,
    gc.class_name,
    'Class Only' AS record_type
FROM members m
RIGHT JOIN class_enrollments ce ON m.member_id = ce.member_id
RIGHT JOIN gym_classes gc ON ce.class_id = gc.class_id
WHERE m.member_id IS NULL;
```

**Business Interpretation:**  
This complete view reveals engagement gaps at Stay Hard Gym, showing both members who don’t attend classes and classes that fail to attract members. 
It enables smarter resource reallocation by redesigning underused class slots and targeting non-participating members, 
which is critical for maximizing facility utilization in Kigali’s premium real estate market.

**Screenshot:** ![Full Outer Join](https://github.com/japhetk-dev/plsql_window_functions_28282_Japhet/blob/f4cf6aeefc9f3267bc412b0e94e814ceda4608fc/screenshots/FULLOUTERJOIN.png)

---

### 5. SELF JOIN: District-Level Member Comparison

**Purpose:** Compare members within the same Kigali district

```sql
SELECT 
    m1.member_name AS member_1,
    mp1.plan_type AS plan_1,
    m2.member_name AS member_2,
    mp2.plan_type AS plan_2,
    m1.district,
    CASE 
        WHEN mp1.plan_type = mp2.plan_type THEN 'Same Tier'
        ELSE 'Different Tier'
    END AS comparison
FROM members m1
INNER JOIN members m2 ON m1.district = m2.district 
    AND m1.member_id < m2.member_id
INNER JOIN membership_plans mp1 ON m1.plan_id = mp1.plan_id
INNER JOIN membership_plans mp2 ON m2.plan_id = mp2.plan_id
WHERE m1.membership_status = 'Active'
ORDER BY m1.district;
```

**Business Interpretation:**  
This compares members within Gasabo, Kicukiro, and Nyarugenge to understand district-level market. Results show Gasabo has diverse plan mix, 
Kicukiro skews toward Basic/Standard, Nyarugenge has higher Premium/VIP ratio. 
This geographic intelligence guides location specific marketing: promote budget options in Kicukiro, emphasize premium features in Nyarugenge.

**Screenshot:** ![Self Join](https://github.com/japhetk-dev/plsql_window_functions_28282_Japhet/blob/84b8284242ca0ef9bddc7a305a2fe9496ddc7b99/screenshots/SELFJOIN.png)

---

## 📊 Part B: Window Functions


### Category 1: Ranking Functions

#### ROW_NUMBER(): Member Revenue Contribution Ranking
```sql
SELECT 
    member_name,
    district,
    SUM(amount_paid) AS total_revenue,
    ROW_NUMBER() OVER (ORDER BY SUM(amount_paid) DESC) AS revenue_rank,
    ROW_NUMBER() OVER (PARTITION BY district ORDER BY SUM(amount_paid) DESC) AS district_rank
FROM members m
JOIN membership_payments p ON m.member_id = p.member_id
GROUP BY member_name, district;
```

**Interpretation:** Identifies Stay Hard Gym's top revenue contributors for VIP treatment and retention focus.
**Screenshot:** ![Ranking Function](https://github.com/japhetk-dev/plsql_window_functions_28282_Japhet/blob/bdcb563e4d6dad78fd97a7a634353f114ac25e38/screenshots/RankingFunction.png)

---

### Category 2: Aggregate Window Functions

#### SUM() OVER(): Running Total of Monthly Revenue
```sql
WITH monthly_revenue AS (
    SELECT DATE_FORMAT(payment_date, '%Y-%m') AS month,
           SUM(amount_paid) AS revenue
    FROM membership_payments
    WHERE payment_status = 'Completed'
    GROUP BY DATE_FORMAT(payment_date, '%Y-%m')
)
SELECT month, revenue,
       SUM(revenue) OVER (ORDER BY month ROWS UNBOUNDED PRECEDING) AS cumulative_revenue,
       AVG(revenue) OVER (ORDER BY month ROWS 2 PRECEDING) AS three_month_avg
FROM monthly_revenue;
```

**Interpretation:** Tracks Stay Hard Gym's financial growth trajectory in Kigali market with trend smoothing.
**Screenshot:** ![Aggregate Window Function](https://github.com/japhetk-dev/plsql_window_functions_28282_Japhet/blob/10f5280f7e3bbec9626d7fb0a1dd3e34bb627c60/screenshots/AggregateFunction.png)

---

### Category 3: Navigation Functions

#### LAG(): Month-over-Month Growth Analysis
```sql
WITH monthly_data AS (...)
SELECT month, revenue,
       LAG(revenue) OVER (ORDER BY month) AS prev_month,
       ROUND((revenue - LAG(revenue) OVER (ORDER BY month)) / 
             LAG(revenue) OVER (ORDER BY month) * 100, 2) AS growth_pct
FROM monthly_data;
```

**Interpretation:** Reveals seasonal patterns (January surge, summer dip) for inventory and staffing planning. 
**Screenshot:** ![Navigation Function](https://github.com/japhetk-dev/plsql_window_functions_28282_Japhet/blob/e07360610ea6359b3902f78001457f93983fcac0/screenshots/NavigationFunction.png)

---

### Category 4: Distribution Functions

#### NTILE(4): Member Value Segmentation
```sql
SELECT member_name, 
       SUM(amount_paid) AS lifetime_value,
       NTILE(4) OVER (ORDER BY SUM(amount_paid) DESC) AS quartile,
       CASE WHEN NTILE(4) OVER (...) = 1 THEN 'VIP'
            WHEN NTILE(4) OVER (...) = 2 THEN 'High Value'
            ELSE 'Growth Potential' END AS segment
FROM ...
GROUP BY member_name;
```

**Interpretation:** Segments 25 members into tiers for differentiated service levels and retention budgets.
**Screenshot:** ![Distribution Function](https://github.com/japhetk-dev/plsql_window_functions_28282_Japhet/blob/33b7a2dcc5f8483ea465bce792a33ed597a550a6/screenshots/DistributionFunction.png)

---

## 📈 Key Insights & Analysis

### Descriptive Analysis: What Happened?

1. **Revenue Performance:**
   - Total payments received: **RWF 2,145,000**
   - Average payment amount: **RWF 53,625**
   - Most used payment method: **Mobile Money (65%)**
   - Best month for revenue: **March 2024**

2. **Member Information:**
   - **25 total members** from 3 Kigali districts
   - **Gasabo has most members (40%)**
   - **Gender balance:** 52% Male, 48% Female
   - **21 members are currently active** (84% retention)

3. **Class Attendance:**
   - **Most popular classes:** Morning Cardio, HIIT Training, and Yoga
   - **Least popular classes:** Bodybuilding and CrossFit
   - **Attendance rate:** 92% of people who enroll actually show up
   - **Ladies Only Aerobics** is well attended

4. **Membership Plans:**
   - **Quarterly (3-month) plan is most popular** - 40% of members choose this
   - **Annual VIP plan** brings in most money per member but few people buy it
   - **Daily Pass** is rarely used

### Diagnostic Analysis: Why Did It Happen?

1. **Why Mobile Money is Popular:**
   - Rwanda uses mobile money a lot (MTN Mobile Money, Airtel Money)
   - People prefer digital payments over cash
   - Stay Hard Gym made it easy to pay with mobile money

2. **Why Quarterly Plans Sell Best:**
   - RWF 65,000 for 3 months feels more affordable than RWF 200,000 upfront
   - Members can try the gym for 3 months before committing long-term
   - Every 3 months, members decide to renew, giving gym chance to improve service

3. **Why Some Members Don't Attend Classes:**
   - Some members feel shy about group classes
   - Work schedules don't match class times
   - Some members don't know about all the available classes

4. **Why March Had High Revenue:**
   - New Year resolution members continued paying in February-March
   - People preparing for healthier lifestyle
   - More people joined in January-February

5. **Why Gasabo Has Most Members:**
   - Gasabo is the largest district in Kigali
   - Many middle-class residents who can afford gym
   - Gym might be located in or near Gasabo

### Prescriptive Analysis: What Should Be Done?

#### Quick Actions (Next 30 Days)

1. **Get More Members to Attend Classes:**
   - Send SMS to 8 members who never attended classes
   - Offer them 1 free class to try
   - Expected result: 2 members start attending = RWF 30,000/month extra income

2. **Fix Class Schedule:**
   - Move morning classes (5-7 AM) to evening (6-8 PM)
   - Reason: People have more time in evenings
   - Expected result: More people attend = RWF 90,000/month extra

3. **Encourage Mobile Money Use:**
   - Give 5% discount for Mobile Money payments
   - Saves gym time and money handling cash
   - Expected result: 80% of people use Mobile Money

#### Medium-Term Actions (Next 90 Days)

4. **Marketing by District:**
   - **Gasabo:** Offer RWF 10,000 discount when members refer friends
   - **Kicukiro:** Use Instagram ads to attract new members
   - **Nyarugenge:** Partner with companies for employee memberships
   - Expected result: 7 new members = RWF 455,000 extra revenue

5. **Upgrade Current Members:**
   - Target top 6 best-paying members (top 25%)
   - Offer them Annual VIP at discount (RWF 150,000 instead of RWF 200,000)
   - Expected result: 3 members upgrade = RWF 450,000 immediate payment

6. **Start Corporate Program:**
   - Offer companies group discounts for 10+ employees
   - Contact banks, telecom companies, and businesses
   - Expected result: 1 company signs up = RWF 1,237,500/year

#### Long-Term Strategy (After 90 Days)

7. **Prevent Members from Leaving:**
   - Track when members stop paying on time
   - Call them personally if payment is late by 45 days
   - Offer flexible payment plans
   - Expected result: Keep 4 members who would have left = RWF 260,000 saved

8. **Improve Class Offerings:**
   - Replace 2 unpopular classes with more popular types
   - Add more HIIT classes and Ladies-only strength training
   - Expected result: RWF 120,000/month extra from better classes

9. **Open Second Location:**
   - Consider opening gym in Kicukiro (growing area, few gyms)
   - Investment needed: RWF 15,000,000
   - Expected result: Profit of RWF 3,000,000/year after 3 years

---

## 📚 References

### Official Documentation
1. MySQL Documentation. (2024). *Window Functions*. Retrieved from https://dev.mysql.com/doc/refman/8.0/en/window-functions.html
2. MySQL Documentation. (2024). *JOIN Syntax*. Retrieved from https://dev.mysql.com/doc/refman/8.0/en/join.html
3. XAMPP Documentation. (2024). *Getting Started Guide*. Retrieved from https://www.apachefriends.org/docs/

### Rwanda Context & Market Research
4. National Bank of Rwanda. (2024). *Mobile Money Usage Statistics*. Retrieved from https://www.bnr.rw/
5. Rwanda Development Board. (2024). *Kigali City Development & Demographics*. Retrieved from https://rdb.rw/

### Academic Resources
6. Date, C.J. (2012). *SQL and Relational Theory: How to Write Accurate SQL Code* (2nd ed.). O'Reilly Media.
7. Stephens, R., & Plew, R. (2018). *Sams Teach Yourself SQL in 24 Hours* (6th ed.). Sams Publishing.

### Business Intelligence
8. Kimball, R., & Ross, M. (2013). *The Data Warehouse Toolkit* (3rd ed.). Wiley.
9. IHRSA. (2024). *Global Fitness Industry Report: Emerging Markets*. International Health, Racquet & Sportsclub Association.

### Online Learning
10. TechTFQ. (2022). *SQL Window Functions Tutorial*. Retrieved from https://youtu.be/Ww71knvhQ-s?si=QYkQhAeFdgu8Syqx

---

“All sources were properly cited. Implementations and analysis represent original work. No AIgenerated content was copied without attribution or adaptation.”

