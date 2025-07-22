### Documentation for SQL Query: Customer Loan Application Ranking Logic

This query creates a table `lat_Loan_application_customer_rnk` that classifies loan applicants as **New**, **Repeat**, or **Not Applied** based on their loan application history relative to their first disbursed loan. Below is a detailed breakdown of the logic:

---

#### **Key Components**
1. **CTE `a1`: Identify First Disbursed Loan per Customer**
   ```sql
   Select 
     customerId,  
     digitalLoanAccountId,
     startApplyDateTime,
     disbursementDateTime,
     row_number() over (partition by customerId order by disbursementDateTime) rnk
   from `risk_credit_mis.loan_master_table`
   where customerId is not null
     and disbursementDateTime is not null
   ```
   - **Purpose**: For each customer, rank their **disbursed loans** by `disbursementDateTime` (earliest first).
   - `rnk = 1`: The earliest disbursed loan for a customer.
   - Filters out records with missing `customerId` or `disbursementDateTime`.

2. **CTE `a2`: Classify Loan Applications**
   ```sql
   select 
     lmt.customerId,
     lmt.digitalLoanAccountId,
     lmt.startApplyDateTime,
     a1.startApplyDateTime obdate,  -- Application date of the first disbursed loan
     a1.rnk,                       -- Rank of the disbursed loan (1 = earliest)
     row_number() over (partition by lmt.customerId, lmt.digitalLoanAccountId 
                        order by a1.rnk) rnkk,
     case 
       when lmt.startApplyDateTime is null then 'Not Applied for Loan yet'
       when a1.startApplyDateTime is null and lmt.startApplyDateTime is not null then 'New Applicant'
       when lmt.startApplyDateTime <= a1.startApplyDateTime and a1.rnk = 1 then 'New Applicant'
       when lmt.startApplyDateTime < a1.startApplyDateTime and a1.rnk > 1 then 'Repeat Applicant'
       when lmt.startApplyDateTime >= a1.startApplyDateTime and a1.rnk = 1 then 'Repeat Applicant'
       when lmt.startApplyDateTime >= a1.startApplyDateTime and a1.rnk > 1 then 'Repeat Applicant'
     end loan_application_approved_cust_rank
   from `risk_credit_mis.loan_master_table` lmt
   left join a1 on a1.customerId = lmt.customerId 
   where lmt.customerId is not null
   ```
   - **Left Join**: All loans from `loan_master_table` (including non-disbursed) are joined with `a1` (disbursed loans) on `customerId`.
   - **`rnkk`**: Orders joined results by `a1.rnk` (lowest first) and keeps only the top result (`rnkk = 1` later).  
     *(Ensures we compare against the customer's earliest disbursed loan)*.
   - **Classification Logic** (`loan_application_approved_cust_rank`):
     | **Condition** | **Classification** | **Explanation** |
     |---------------|---------------------|----------------|
     | `lmt.startApplyDateTime IS NULL` | `Not Applied for Loan yet` | Loan application has no start date. |
     | `a1.startApplyDateTime IS NULL AND lmt.startApplyDateTime NOT NULL` | `New Applicant` | No disbursed loans exist for the customer. |
     | `lmt.startApplyDateTime <= a1.startApplyDateTime AND a1.rnk = 1` | `New Applicant` | Loan applied **before or at the same time** as the first disbursed loan. |
     | All other cases | `Repeat Applicant` | Loan applied **after** the first disbursed loan or compared to a non-first disbursed loan. |

3. **Final Output**
   ```sql
   select distinct * 
   from a2
   where rnkk = 1
   order by customerId, startApplyDateTime, obdate
   ```
   - **`rnkk = 1`**: Keeps only the earliest disbursed loan record per `digitalLoanAccountId`.
   - **Distinct**: Ensures unique rows (redundant due to `rnkk=1` but safe).
   - **Order**: Sorts by `customerId`, loan application date (`startApplyDateTime`), and first disbursed loan's application date (`obdate`).

---

#### **Workflow Summary**
1. **Step 1 (CTE `a1`)**  
   Identify each customer's first disbursed loan (by `disbursementDateTime`).

2. **Step 2 (CTE `a2`)**  
   - For every loan application:
     - Join with the customer's disbursed loans from `a1`.
     - Use `rnkk` to prioritize the **earliest disbursed loan** (`a1.rnk = 1`) for comparison.
     - Classify the application based on timing relative to the first disbursed loan.

3. **Step 3 (Final Output)**  
   - Filter to keep only the most relevant disbursed loan (`rnkk = 1`).
   - Remove duplicates and order results.

---

#### **Example Scenarios**
| **Scenario** | `lmt.startApplyDateTime` | `a1.startApplyDateTime` | `a1.rnk` | **Classification** |
|--------------|--------------------------|-------------------------|----------|--------------------|
| New applicant | 2023-01-01 | `NULL` | `NULL` | `New Applicant` |
| First loan application | 2023-01-01 | 2023-01-01 | 1 | `New Applicant` |
| Applied before first disbursed loan | 2023-01-01 | 2023-01-05 | 1 | `New Applicant` |
| Applied after first disbursed loan | 2023-01-10 | 2023-01-05 | 1 | `Repeat Applicant` |
| Compared to non-first disbursed loan | 2023-01-03 | 2023-01-02 | 2 | `Repeat Applicant` *(Ignored due to `rnkk=1`)* |

---

#### **Key Notes**
- **Purpose**: Track whether a loan application is from a **new** or **repeat** customer based on their **first disbursed loan**.
- **Critical Fields**:
  - `startApplyDateTime`: Date the loan application was started.
  - `disbursementDateTime`: Date the loan was disbursed (must exist for ranking).
  - `obdate`: Application date of the customer’s first disbursed loan.
- **Edge Handling**: 
  - Applications without `startApplyDateTime` are marked as `Not Applied for Loan yet`.
  - Customers with no disbursed loans are always `New Applicant`.

This logic efficiently categorizes loan applicants by comparing application dates against the earliest disbursed loan, providing clear insights into customer behavior.