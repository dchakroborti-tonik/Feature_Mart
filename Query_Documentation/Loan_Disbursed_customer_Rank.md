### Detailed Documentation: SQL Query Logic  

#### Objective  
Create a table `lat_Loan_disbursed_customer_rnk` that classifies customers as **New** or **Repeat Disbursement Customers** based on their loan disbursement history.  

---

### Step-by-Step Logic  
#### 1. **Common Table Expression (CTE) `a1`**  
```sql
with a1 as (  
  Select  
    customerId,  
    digitalLoanAccountId,  
    startApplyDateTime,  
    disbursementDateTime,  
    row_number() over (  
      partition by customerId  
      order by disbursementDateTime  
    ) rnk  
  from `risk_credit_mis.loan_master_table`  
  where customerId is not null  
    and disbursementDateTime is not null  
)  
```  

- **Purpose**: Prepare clean, partitioned loan data for ranking.  
- **Key Operations**:  
  - **Filtering**: Exclude records where `customerId` or `disbursementDateTime` is `NULL` (ensures valid customer/disbursement data).  
  - **Partitioning**: Group data by `customerId` (each group represents one customer’s loan history).  
  - **Ranking**: Assign a row number (`rnk`) to each loan within a customer’s group, ordered chronologically by `disbursementDateTime`.  
    - `rnk = 1`: First disbursed loan for the customer.  
    - `rnk > 1`: Subsequent disbursed loans.  

---

#### 2. **Final Selection & Classification**  
```sql
select  
  customerId,  
  digitalLoanAccountId,  
  startApplyDateTime,  
  disbursementDateTime,  
  case  
    when disbursementDateTime is not null and rnk = 1 then 'New Disbursement Customer'  
    when disbursementDateTime is not null and rnk > 1 then 'Repeat Disbursement Customer'  
  end Loan_disbursement_rnk  
from a1  
order by customerId, startApplyDateTime, disbursementDateTime;  
```  

- **Purpose**: Classify customers and output results.  
- **Key Operations**:  
  - **Classification Logic**:  
    - **New Disbursement Customer**: First disbursed loan (`rnk = 1`).  
    - **Repeat Disbursement Customer**: Subsequent disbursed loans (`rnk > 1`).  
  - **Sorting**: Results ordered by:  
    1. `customerId` (group loans by customer).  
    2. `startApplyDateTime` (order loans by application time).  
    3. `disbursementDateTime` (secondary chronological order).  

---

### Key Concepts Explained  
#### 1. **Window Function: `row_number()`**  
- **Functionality**: Assigns a unique sequential integer to rows within each partition.  
- **Partition**: `customerId` (process each customer’s loans separately).  
- **Order**: `disbursementDateTime` (ranks loans from earliest to latest disbursement).  
- **Example Output**:  
  | customerId | disbursementDateTime | rnk |  
  |------------|----------------------|-----|  
  | 123        | 2023-01-01           | 1   |  
  | 123        | 2023-02-15           | 2   |  

#### 2. **Classification Logic**  
- **New Customer**: The earliest disbursed loan per customer (`rnk = 1`).  
- **Repeat Customer**: Any loan after the first (`rnk > 1`).  
- **Why `disbursementDateTime`?**: Determines the chronological order of loans.  

#### 3. **Data Quality Filters**  
- `customerId is not null`: Ensures loans are tied to identifiable customers.  
- `disbursementDateTime is not null`: Guarantees loans have been disbursed (excludes pending/rejected applications).  

---

### Example Output  
| customerId | digitalLoanAccountId | startApplyDateTime | disbursementDateTime | Loan_disbursement_rnk       |  
|------------|----------------------|--------------------|----------------------|-----------------------------|  
| 123        | DL-001              | 2023-01-01 10:00   | 2023-01-05 12:00     | New Disbursement Customer   |  
| 123        | DL-002              | 2023-02-10 11:00   | 2023-02-15 14:00     | Repeat Disbursement Customer|  
| 456        | DL-003              | 2023-03-20 09:00   | 2023-03-25 15:00     | New Disbursement Customer   |  

---

### Notes  
1. **Handling Ties**:  
   - If two loans have the same `disbursementDateTime`, `row_number()` arbitrarily assigns ranks. To ensure deterministic results, add a tiebreaker (e.g., `order by disbursementDateTime, digitalLoanAccountId`).  

2. **Table Structure**:  
   - Output table contains:  
     - **customerId**: Unique customer identifier.  
     - **digitalLoanAccountId**: Loan account ID.  
     - **startApplyDateTime**: When the loan application started.  
     - **disbursementDateTime**: When the loan was disbursed.  
     - **Loan_disbursement_rnk**: Classification of the customer for that loan.  

3. **Performance**:  
   - Partitioning by `customerId` leverages BigQuery’s parallel processing for large datasets.  
   - Filters (`is not null`) reduce data scanned.  

---

### Use Cases  
- **Customer Behavior Analysis**: Track new vs. repeat loan customers.  
- **Risk Modeling**: Identify repeat borrowers for credit risk assessment.  
- **Marketing**: Target repeat customers for loyalty programs.  

This query transforms raw loan data into actionable insights about customer disbursement behavior.