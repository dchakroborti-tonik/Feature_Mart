### Documentation of SQL Query Logic

This query retrieves outstanding principal amounts for loan accounts at specific default milestones (10/30 days after due dates) where the account has default flags (`defFPD10`, `defFPD30`, `defSPD30`, or `defTPD30`) set to `1`. The logic is broken down into multiple CTEs for clarity.

---

### **1. `all_default_account` (Base CTE)**
**Purpose**: Identify all loan accounts with at least one active default flag.  
**Source Table**:  
- `risk_credit_mis.loan_master_table`  
**Columns Selected**:  
- `digitalLoanAccountId`, `loanAccountNumber`, `firstDueDate`, `secondDueDate`, `thirdDueDate`  
- `defFPD30`, `defSPD30`, `defTPD30` (default flags for 30-day milestones)  
**Filter Condition**:  
```sql 
WHERE defFPD10=1 OR defFPD30=1 OR defSPD30=1 OR defTPD30=1
```
**Output**: All loans with at least one default flag active.

---

### **2. Default Principal Outstanding CTEs**
**Purpose**: For each default type, capture the **PRINCIPALARREARS** from `core_raw.loan_accounts` at specific dates relative to due dates. Uses a 1-day buffer to handle non-business days.  

#### **a. `defpd10_pricipal_outstanding` (First Payment Default @ 10 Days)**
- **Applicable Loans**: `defFPD10=1`  
- **Lookup Date**: `firstDueDate + 10/11 days`  
- **Logic**:  
  ```sql
  JOIN core_raw.loan_accounts core 
    ON core.ACCOUNTNUMBER = loan.loanAccountNumber 
    AND (core._Partitiondate = DATE_ADD(firstDueDate, INTERVAL 10 DAY)
         OR core._Partitiondate = DATE_ADD(firstDueDate, INTERVAL 11 DAY))
  WHERE defFPD10=1
  QUALIFY ROW_NUMBER() OVER (
    PARTITION BY loanAccountNumber 
    ORDER BY PRINCIPALARREARS DESC
  ) = 1
  ```
- **Result**: Highest `PRINCIPALARREARS` for each loan on day 10/11 after the first due date.

#### **b. `defpd30_pricipal_outstanding` (First Payment Default @ 30 Days)**
- **Applicable Loans**: `defFPD30=1`  
- **Lookup Date**: `firstDueDate + 30/31 days`  
- **Same logic as above**, but for 30-day default.

#### **c. `defspd30_pricipal_outstanding` (Second Payment Default @ 30 Days)**
- **Applicable Loans**: `defSPD30=1`  
- **Lookup Date**: `secondDueDate + 30/31 days`  
- **Uses `secondDueDate`** instead of `firstDueDate`.

#### **d. `deftpd30_pricipal_outstanding` (Third Payment Default @ 30 Days)**
- **Applicable Loans**: `defTPD30=1`  
- **Lookup Date**: `thirdDueDate + 30/31 days`  
- **Uses `thirdDueDate`**.

---

### **3. `default_outstanding_principal` (Combined Results)**
**Purpose**: Consolidate results from all default CTEs into a single row per loan account.  
**Logic**:  
```sql
SELECT 
  allaccount.digitalLoanAccountId,
  allaccount.loanAccountNumber,
  day10dpd.defpd10_outstanding_principal,  -- From defpd10 CTE
  firstdpd.defpd30_outstanding_principal,   -- From defpd30 CTE
  seconddpd.defspd30_outstanding_principal, -- From defspd30 CTE
  thirddpd.deftpd30_outstanding_principal,  -- From deftpd30 CTE
  allaccount.firstDueDate,
  allaccount.secondDueDate,
  allaccount.thirdDueDate,
  allaccount.defFPD30 AS loan_defFPD30,     -- Original flags
  allaccount.defSPD30 AS loan_defSPD30,
  allaccount.defTPD30 AS loan_defTPD30
FROM all_default_account allaccount
LEFT JOIN defpd10_pricipal_outstanding day10dpd 
  ON allaccount.loanAccountNumber = day10dpd.loanAccountNumber
LEFT JOIN defpd30_pricipal_outstanding firstdpd 
  ON allaccount.loanAccountNumber = firstdpd.loanAccountNumber
LEFT JOIN defspd30_pricipal_outstanding seconddpd 
  ON allaccount.loanAccountNumber = seconddpd.loanAccountNumber
LEFT JOIN deftpd30_pricipal_outstanding thirddpd 
  ON allaccount.loanAccountNumber = thirddpd.loanAccountNumber
```
- **Left Joins**: Ensures all loans from `all_default_account` are included, even if they don’t have data in a specific default CTE.  
- **Output**: One row per loan with columns for each default type’s principal arrears (null if not applicable).

---

### **4. Final `SELECT`**
```sql
SELECT * FROM default_outstanding_principal;
```
**Output**: All columns from the `default_outstanding_principal` CTE.

---

### **Key Notes**
1. **Date Handling**:  
   - Uses `DATE_ADD` and a 1-day buffer (e.g., `+10/11 days`) to handle weekends/holidays where data might not be captured.
   
2. **Deduplication**:  
   - `QUALIFY ROW_NUMBER() ... =1` ensures only one record per loan is kept, prioritizing the **highest principal arrears** if multiple dates match.

3. **Default Flags**:  
   - The base CTE (`all_default_account`) filters loans with **any** default flag active.
   - Individual CTEs further filter loans relevant to their specific default type.

4. **Null Values**:  
   - If a loan has `defFPD10=1` but no matching `core_raw.loan_accounts` record on day 10/11, `defpd10_outstanding_principal` will be `null`.

---

### **Summary Flow**
1. Identify defaulted loans →  
2. For each default type, fetch principal arrears at defined days post-due date →  
3. Combine results into a unified structure →  
4. Output all data.