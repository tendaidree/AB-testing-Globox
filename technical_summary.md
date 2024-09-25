# A/B Test Report: Food and Drink Banner


## Purpose

GloBox is primarily known amongst its customer base for boutique fashion items and high-end decor products. 
However, their food and drink offerings have grown tremendously in the last few months, and the company wants to bring awareness to this product 
category to increase revenue. The purpose of this project is to run an A/B test that highlights key products in the food and drink category as a 
banner at the top of the website. The control group does not see the banner, and the test group sees it.

## Hypotheses
Null Hypothesis: There is no change in (conversion rate or average spend) between the groups.

Alternative Hypothesis: There is a change in (conversion rate or average spend) between the groups

## Methodology
### Test Design

The setup of the A/B test is as follows:

1.The experiment is only being run on the mobile website

2. A user visits the GloBox main page and is randomly assigned to either the control or test group. This is the join date for the user.

3. The page loads the banner if the user is assigned to the test group, and does not load the banner if the user is assigned to the control group.

4. The user subsequently may or may not purchase products from the website. It could be on the same day they join the experiment, or days later and if they do make one or more purchases, this is considered a “conversion”.


- **Population:** 
Sample size of Control group (A) 24343
Sample size of Test groups (B) 24600
Total Sample size 48943

- **Duration:** 
Duration of Test was 13 Days
Test start date : 2023-01-25 
Test end date :2023-02-06

- **Success Metrics:** 
The metrics that were used to measure success are :
Average spent
Conversion rate (the increase in conversion rate comparing the two groups)

## Results
### Data Analysis

- **Pre-Processing Steps:**

1.The initial data analysis, cleaning, and transformation were conducted using PostgreSQL.

2. After cleaning and transforming the data, a new dataset was created and exported to Tableau Desktop for exploratory data visualization.

3.The same dataset was also analysed in Google Sheets for hypothesis testing and determining confidence intervals. These confidence interval values were then added to the existing dataset and exported to Tableau for visualization.

4.Another dataset was extracted in sql then exported to Tableau Desktop to check for novelty effects. 
  

```sql

/*  Checking if a user can show up in the activity table more than once  */

SELECT 
      uid,
      COUNT(uid) AS num_purchases
FROM activity
GROUP BY uid
HAVING COUNT(uid) > 1
ORDER BY 1 DESC
;

--- a single user can make purchases on multiple days and this entails that the user will appear more than once in the activity table


/* All users will not make purchases and for the purpose of the analysis we will need to include all users regardless of purchase or not
       hence will filled all Null purchases with 0 */

SELECT 
      u.id,
      u.country,
      COALESCE(a.spent,0)
FROM users u 
LEFT JOIN activity a
ON u.id = a.uid
ORDER BY 3 DESC
;

/* What is the start and end date of the experiment*/

SELECT
      MIN(join_dt) AS start_date,
      MAX(join_dt) AS end_date
FROM groups
;

/* Total number of users in the experiment and users were in the control and treatment groups */

--total users

SELECT
      COUNT(DISTINCT id) AS total_users
FROM users
;

--- control and treatment group

SELECT
     CASE
        WHEN gp.group = 'A' THEN 'Control'
        ELSE 'Treatment'
     END AS num_per_group,
     COUNT(uid)
FROM groups gp
GROUP BY gp.group
;

/* Calculating the conversion rate of all users?*/

SELECT 
      (COUNT(DISTINCT a.uid)*1.0 / COUNT(DISTINCT u.id)) * 100 AS allusers_conv_rate
FROM users u
LEFT JOIN activity a
ON a.uid=u.id
;

/*  The user conversion rate for the control and treatment groups*/

SELECT 
      CASE
        WHEN gp.group = 'A' THEN 'Control'
        ELSE 'Treatment'
     END AS group_name,
     (COUNT(DISTINCT a.uid)*1.0 / COUNT(DISTINCT u.id)) * 100 AS group_conv_rate
FROM users u
LEFT JOIN activity a
ON a.uid=u.id
LEFT JOIN groups gp
ON gp.uid = u.id
GROUP BY gp.group
;


/* The average amount spent per user for the control and treatment groups, including users who did not convert?*/


WITH CTE AS (SELECT
      gp.uid,
      CASE
          WHEN gp.group ='A' THEN 'Control'
          ELSE 'Treatment'
       END AS "group_type",
      ROUND(SUM(COALESCE(a.spent,0)),2) AS conversion_rate
FROM groups gp
LEFT JOIN activity a
ON gp.uid = a.uid
GROUP BY 1,2)
SELECT 
      group_type,
      ROUND(AVG(conversion_rate),2)
FROM CTE
GROUP BY group_type
;



/* The next step was to extract a dataset with all the relevant fields for further analysis in Tableau

Steps for the extraction of the dataset

1. Combine all relevant information in one table (user id, group, device, gender, total spend)
2. Create a new column to indicate whether a website guest had converted into a buying customer (converted_not_Converted)
3. Clean duplicate values by summing the spend amount per unique user  
  
  */


WITH cte AS (
SELECT
      gp.uid AS user_id,
      SUM(COALESCE(a.spent,0)) AS amount_spent
FROM groups gp
LEFT JOIN activity a
ON gp.uid = a.uid
GROUP BY 1)

SELECT DISTINCT
      user_id,
      u.country,
      u.gender,
      gp.device,
      gp.group,
      amount_spent,
      CASE 
            WHEN amount_spent > 0 THEN 1
            ELSE '0'
       END AS "converted_not_Converted"
FROM cte
INNER JOIN users u
ON user_id = u.id
INNER JOIN groups gp
ON u.id = gp.uid
LEFT JOIN activity a
ON a.uid = u.id

;


/* Extracting a dataset for further analysis in Tableau for a potential novelty effect by examining user engagement  */


SELECT   
    g.join_dt AS join_date, g.group, 
    COUNT(DISTINCT g.uid) AS total_users,  --Counting the total number of distinct users in each group.
    COUNT(DISTINCT a.uid) AS paying_users,  --Counting the total number of distinct users who paid.
    SUM(a.spent) AS total_spent  --Total amount spent by users in all paying activities.
FROM groups g
LEFT JOIN activity a ON g.uid = a.uid  --Left join groups and activity tables.
GROUP BY g.group, g.join_dt  -- Grouping the results by group and join date.
ORDER BY join_date;


SELECT n.group, COUNT(n.user_id), n.date_difference
FROM (
    
      SELECT 
        a.uid AS user_id,  --Selecting user ID from activity table.
        g.group,  --Selecting user group from groups table.
        g.join_dt AS date_registered,        ---Selecting registration date from groups table as 'date_registered'.
        a.dt AS date_converted,      ---Selecting conversion date from activity table as 'date_converted'.
        SUM(COALESCE(a.spent, 0)) AS total_spent,     ---Summing the spent amount, handling NULLS with COALESCE.
        a.dt - g.join_dt AS date_difference     ---Calculating the date difference between conversion and registration.
    FROM groups g
    JOIN activity a ON g.uid = a.uid       ---Joining groups and activity tables.
    GROUP BY 1, 2, 3, 4       ---Grouping by user ID, user group, registration date, and conversion date.
) AS n  --Subquery (Aliased as n):
GROUP BY 1, 3      ---Grouping the results by user group and date difference.

;

```

- **Statistical Tests Used:** 

Two-sample z-test for difference in proportions: Hypothesis test conducted to see whether there is a difference in the conversion rate between treatment and control group.

Two-sample t-test for a difference in means : Hypothesis test conducted to see whether there is a difference in the average amount spent per user between treatment and control group


- **Results Overview:** High-level summary.

### Findings

Conversion rate: p-value = 0.000111 (p-value<0.05), Reject null hypothesis. There is a significant difference between the groups

Average spent: p-value = 0.944194 (p-value>0.05) Fail to reject null hypothesis. There is no significant difference between the groups

## Interpretation

- **Outcome of the Test(s):** 

Conversion rate: There is a significant difference between the groups p-value = 0.000111 (p-value<0.05), Reject null hypothesis. 

Average spent: There is no significant difference between the groups p-value = 0.944194 (p-value>0.05) Fail to reject null hypothesis. 

- **Confidence Level:**

We can be confident that 95% of values for conversion rate will fall between 3.68-4.17% for the control group and 4.37-4.89% for the test
group (those seeing the banner).

We can also be confident that 95% of values for average spend per user will fall between $3.05-$3.70 for our control group, and for those shown
the banner, values will fall between $3.07 - $3.71. This continues to prove that we are not seeing any significant revenue growth as most
values overlap.


## Conclusions

- **Key Takeaways:** 

There was a positive response in conversions rate for the test group but however our revenue increase is significantly low.

No novelty effects were detected


- **Limitations/Considerations:** Any potential biases or anomalies.

1. Increase the sample size to at least 77K users split equally for sufficient power.

2. Consider a longer duration for the test.

3. Include the type of purchase in data collection for further analysis.


## Recommendations

- **Next Steps:** 

I recommend launching because generally Group B presented significantly better results compared to group A

Considering that the new feature is at low cost there is a positive cost benefit

- **Further Analysis:**

Predictive analysis or market basket analysis if we include the type of purchase in data collection




