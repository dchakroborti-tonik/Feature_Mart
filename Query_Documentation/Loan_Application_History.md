### Detailed Documentation: Loan Application History Analysis Query

#### **Objective**
This query analyzes a customer's loan application history prior to a specific reference date ("obdate") for each loan application. It calculates aggregated metrics across previous applications to understand customer behavior patterns.

---

### **Query Structure**
The query uses two CTEs:
1. **`a1`**: Enriches loan data with status information
2. **`b1`**: Computes historical metrics based on previous applications

---

### **CTE a1: Loan Status Enrichment**
```sql
a1 as (
  select 
    lmt.customerId,
    lmt.digitalLoanAccountId,
    ... -- loan attributes
    case 
      when lmt.disbursementDateTime is not null and e1.loanStatus is null then '4. Active Loan' 
      when lmt.disbursementDateTime is not null and lower(e1.loanStatus) like '%sold%' then '1. Sold'
      when lmt.disbursementDateTime is not null and lower(e1.loanStatus) like '%settl%' then '2. Settled'
      when lmt.disbursementDateTime is not null and lower(e1.loanStatus) like '%compl%' then '3. Completed'
      else '5. No Disbursed loan' 
    end loanStatus,
    e1.eventdate
  from loan_master_table lmt
  left join (
    select loanAccountNumber, loanStatus, min(bucketDate) eventdate 
    from loan_bucket_flow_report_core 
    where loanStatus in ('Settled', 'Completed', 'Sold')
    group by 1, 2
  ) e1 on e1.loanAccountNumber = lmt.loanAccountNumber
)
```

#### **Key Logic**
1. **Status Classification**:
   - `Active Loan`: Disbursed but no status in flow report
   - `Sold`: Disbursed and status contains "sold"
   - `Settled`: Disbursed and status contains "settl"
   - `Completed`: Disbursed and status contains "compl"
   - `No Disbursed loan`: Not disbursed

2. **Subquery `e1`**:
   - Finds earliest event date for final loan statuses
   - Aggregates by loan account number and status

---

### **CTE b1: Historical Metrics Calculation**
```sql
b1 as (
  select 
    lmt.customerId,
    lmt.digitalLoanAccountId,
    case  -- Define obdate
      when lmt.new_loan_type ='Flex-up' and lmt.disbursementDateTime is not null then lmt.disbursementDateTime
      when lmt.new_loan_type ='Flex-up' and lmt.disbursementDateTime is null then coalesce(lmt.startApplyDateTime, lmt.startInitiateDateTime)
      when lmt.new_loan_type not like 'Flex-up' then coalesce(lmt.termsAndConditionsSubmitDateTime, lmt.startApplyDateTime, lmt.startInitiateDateTime)
    end obdate,
    ... -- 40+ aggregated metrics
  from loan_master_table lmt 
  left join a1 
    on a1.customerId = lmt.customerId 
    and [a1_derived_date] < [lmt_derived_date] -- Date comparison logic
  group by 1,2,3
)
```

#### **Core Components**
1. **`obdate` (Reference Date)**:
   - `Flex-up` loans: Disbursement date > Apply date > Initiate date
   - Other loans: T&C submission > Apply date > Initiate date

2. **Historical Metrics**:
   - **Counts**:
     - Total loans tried before `obdate`
     - Approved/rejected/disbursed counts
     - Loan type breakdowns (Quick, SIL, Other)
     - Status-based counts (Sold/Settled/Completed/Active)
   - **Financial Metrics**:
     - Min/Max requested/approved/disbursed amounts
     - Min/Max requested/approved/disbursed tenures

3. **Date Comparison Logic**:
   - Compares derived dates using same rules as `obdate`:
     ```sql
     case when a1.new_loan_type ='Flex-up' then coalesce(a1.disbursementDateTime, a1.startApplyDateTime, a1.startInitiateDateTime)
          else coalesce(a1.termsAndConditionsSubmitDateTime, a1.startApplyDateTime, a1.startInitiateDateTime)
     end < obdate
     ```

---

### **Key Metrics Summary**
| Category | Metrics | Description |
|----------|---------|-------------|
| **Volume Metrics** | `cnt_loans_tried_bf_obdate` | Total previous applications |
|  | `cnt_loans_rejected_bf_obdate` | Previously rejected loans |
|  | `cnt_quick_loan_tried_bf_obdate` | Quick loan applications |
| **Financial Metrics** | `min_loan_amt_requested_bf_obdate` | Smallest requested amount |
|  | `max_loan_amt_approved_bf_obdate` | Largest approved amount |
|  | `min_loan_tenure_disb_bf_obdate` | Shortest disbursed tenure |
| **Status Metrics** | `no_of_loans_sold_bf_obdate` | Loans sold to third parties |
|  | `no_of_loans_active_bf_obdate` | Currently active loans |

---

### **Technical Notes**
1. **Null Handling**:
   - `coalesce()` equivalent logic in `case` statements
   - Zero-default in financial aggregates (`else 0`)

2. **Performance**:
   - Self-join on `customerId` may be expensive
   - Repeated date derivation logic could be optimized

3. **Edge Cases**:
   - "Other loans" defined as neither Quick nor SIL
   - Flex-up loans prioritize disbursement date

---

### **Example Output**
| customerId | digitalLoanAccountId | obdate | cnt_loans_tried_bf_obdate | max_loan_amt_approved... |
|------------|----------------------|--------|---------------------------|--------------------------|
| 12345 | DL-2023-001 | 2023-01-15 | 2 | 5000 |
| 12345 | DL-2023-002 | 2023-03-22 | 3 | 7500 |

---

### **Usage Scenarios**
1. Customer risk profiling
2. Loan approval decision support
3. Customer lifetime value analysis
4. Product preference analysis (Quick vs SIL vs Other)

This query provides a comprehensive view of a customer's loan application history prior to each new application, enabling data-driven decision making in lending operations.