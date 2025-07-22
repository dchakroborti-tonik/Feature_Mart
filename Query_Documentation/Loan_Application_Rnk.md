### Documentation: Loan Application Ranking Logic  

#### Overview  
This query creates a table `lat_Loan_application_rnk` that categorizes customers as **New Applicants** (first loan application) or **Repeat Applicants** (subsequent applications). It uses the `loan_master_table` to determine the chronological order of applications per customer.  

---

### Step-by-Step Logic  

#### **1. Common Table Expression (CTE) `a1`**  
```sql
Select 
  customerId,  
  digitalLoanAccountId,
  row_number() over (partition by customerId order by startApplyDateTime) rnk
from `risk_credit_mis.loan_master_table`
```  
- **Purpose**: Assigns a rank to each loan application per customer based on application time.  
- **Key Operations**:  
  - `PARTITION BY customerId`: Groups rows by `customerId` (each customer’s applications are ranked separately).  
  - `ORDER BY startApplyDateTime`: Ranks applications chronologically (earliest = `rnk=1`).  
  - `row_number()`: Assigns a unique sequential rank (e.g., 1, 2, 3...) within each customer partition.  
- **Output Columns**:  
  - `customerId`, `digitalLoanAccountId`, `rnk` (application rank).  

---

#### **2. CTE `a2`**  
```sql
select 
  customerId,
  digitalLoanAccountId,
  rnk,
  case 
    when rnk = 1 then 'New Applicant'
    when rnk > 1 then 'Repeat Applicant' 
  end Loan_application_rnk
from a1
```  
- **Purpose**: Converts the numeric rank (`rnk`) into a categorical label.  
- **Logic**:  
  - `rnk = 1` → Labeled **"New Applicant"** (customer’s first application).  
  - `rnk > 1` → Labeled **"Repeat Applicant"** (subsequent applications).  
- **Output Columns**:  
  - `customerId`, `digitalLoanAccountId`, `rnk`, `Loan_application_rnk`.  

---

#### **3. Final Table Creation**  
```sql
select 
  lmt.customerId,
  lmt.digitalLoanAccountId,
  a2.rnk,
  a2.Loan_application_rnk
from `risk_credit_mis.loan_master_table` lmt 
left join a2 
  on a2.customerId = lmt.customerId 
  and lmt.digitalLoanAccountId = a2.digitalLoanAccountId
where lmt.customerId is not null
```  
- **Purpose**: Joins the original table with ranked/labeled data from `a2`.  
- **Operations**:  
  - **Left Join**: Ensures all records from `loan_master_table` (aliased as `lmt`) are retained, even if no match exists in `a2` (though unlikely due to CTE logic).  
  - **Join Conditions**:  
    - Match on `customerId` and `digitalLoanAccountId` (ensures correct rank/label per loan application).  
  - **Filter**: `WHERE lmt.customerId IS NOT NULL` excludes records without a `customerId`.  
- **Output Columns**:  
  - `customerId`, `digitalLoanAccountId`, `rnk` (rank), `Loan_application_rnk` (category).  

---

### Key Business Rules  
1. **Rank Assignment**:  
   - The **earliest** `startApplyDateTime` for a customer = `rnk=1` ("New Applicant").  
   - All later applications = `rnk>1` ("Repeat Applicant").  
2. **Uniqueness**:  
   - Each `digitalLoanAccountId` (loan application) is ranked individually per customer.  
3. **Data Integrity**:  
   - Excludes records with `NULL customerId` to ensure accurate customer-level analysis.  

---

### Example Output  
| customerId | digitalLoanAccountId | rnk | Loan_application_rnk |  
|------------|----------------------|-----|-----------------------|  
| C-001      | L-123                | 1   | New Applicant        |  
| C-001      | L-456                | 2   | Repeat Applicant     |  
| C-002      | L-789                | 1   | New Applicant        |  

---

### Notes  
- **Input Table**: `risk_credit_mis.loan_master_table` (assumes one row per `digitalLoanAccountId`).  
- **Output Table**: `dap_ds_poweruser_playground.lat_Loan_application_rnk`.  
- **Use Case**: Identify first-time vs. repeat loan applicants for risk analysis or customer behavior studies.  

This logic ensures accurate chronological ranking and categorization of loan applications while preserving all valid records from the source table.