### SQL Query Documentation: Delinquency Flag Calculation

#### **Objective**  
This query retrieves loan account details and computes delinquency flags (e.g., first/second/third payment default) based on observed installment default behavior. The results help identify loans at different stages of delinquency (10-day and 30-day buckets).

---

#### **Tables Used**
1. **`prj-prod-dataplatform.risk_credit_mis.loan_deliquency_data`** (Alias: `ldd`)  
   - Contains granular delinquency data for loans.  
   - **Key Column**: `loanAccountNumber` (primary key).  
   - **Critical Fields**:  
     - `obs_min_inst_def10`: Observed occurrences of 10-day installment defaults.  
     - `min_inst_def10`: Minimum installment default count (10-day bucket).  
     - `obs_min_inst_def30`/`min_inst_def30`: 30-day equivalents.  

2. **`risk_credit_mis.loan_master_table`** (Alias: `lmt`)  
   - Stores master loan records.  
   - **Key Column**: `loanAccountNumber` (joined to `ldd`).  
   - **Selected Fields**:  
     - `customerId`, `digitalLoanAccountId`, `loanPaidStatus`, `currentDelinquency`.  

---

#### **Join Logic**  
- **`LEFT JOIN`** between `ldd` and `lmt`:  
  ```sql
  LEFT JOIN `risk_credit_mis.loan_master_table` lmt 
    ON lmt.loanAccountNumber = ldd.loanAccountNumber
  ```  
  - All records from `loan_deliquency_data` (`ldd`) are retained.  
  - Matching records from `loan_master_table` (`lmt`) are appended. Non-matches return `NULL` for `lmt` columns.  

---

#### **Computed Flags**  
Flags indicate specific delinquency events using `CASE` statements. All flags output `1` (true) or `0` (false).  

| **Flag Name**          | **Logic**                                                                 | **Business Meaning**                                                                 |
|------------------------|---------------------------------------------------------------------------|--------------------------------------------------------------------------------------|
| `deffpd10`             | `obs_min_inst_def10 >=1 AND min_inst_def10 = 1`                           | First payment default (10+ days).                                                    |
| `deffpd30`             | `obs_min_inst_def30 >=1 AND min_inst_def30 = 1`                           | First payment default (30+ days).                                                    |
| `deffspd30`            | `obs_min_inst_def30 >=2 AND min_inst_def30 IN (1, 2)`                     | Second payment default (30+ days). Requires ≥2 observed defaults.                    |
| `deffstpd30`           | `obs_min_inst_def30 >=3 AND min_inst_def30 IN (1, 2, 3)`                  | Third payment default (30+ days). Requires ≥3 observed defaults.                     |
| `flg_mature_fpd10`     | `obs_min_inst_def10 >=1`                                                  | Loan "matured" into a first payment default (10+ days).                              |
| `flg_mature_fpd30`     | `obs_min_inst_def30 >=1`                                                  | Loan "matured" into a first payment default (30+ days).                              |
| `flg_mature_fspd_30`   | `obs_min_inst_def30 >=2`                                                  | Loan "matured" into a second payment default (30+ days).                             |
| `flg_mature_fstpd_30`  | `obs_min_inst_def30 >=3`                                                  | Loan "matured" into a third payment default (30+ days).                              |

> **Key Notes on Flags**:  
> - **`deff*` flags**: Require both a threshold of observed defaults (`obs_min_inst_def*`) AND a specific minimum default count (`min_inst_def*`).  
> - **`flg_mature_*` flags**: Depend solely on the count of observed defaults, regardless of the minimum default value.  

---

#### **Field Glossary**
| **Field**                   | **Source Table** | **Description**                                              |
|-----------------------------|------------------|--------------------------------------------------------------|
| `loanAccountNumber`         | `ldd`            | Unique identifier for the loan account.                      |
| `customerId`                | `lmt`            | Identifier for the customer holding the loan.                |
| `digitalLoanAccountId`      | `lmt`            | Digital ID of the loan account (if applicable).              |
| `loanPaidStatus`            | `lmt`            | Current repayment status (e.g., paid/unpaid).                |
| `currentDelinquency`        | `lmt`            | Latest delinquency status (e.g., days past due).             |
| `obs_min_inst_def{10/30}`   | `ldd`            | Number of times installments were defaulted (10/30-day).     |
| `min_inst_def{10/30}`       | `ldd`            | Minimum installment default count (10/30-day).               |

---

#### **Key Observations**
1. **Delinquency Focus**:  
   - Flags use **10-day** and **30-day** delinquency buckets.  
   - `deffpd10`/`deffpd30` identify initial defaults, while `deffspd30`/`deffstpd30` escalate severity.  

2. **Data Relationship**:  
   - `ldd` is the primary table; `lmt` enriches with loan/customer metadata.  

3. **Null Handling**:  
   - Non-matching `lmt` records return `NULL` for its columns (e.g., `customerId`).  

---

#### **Example Use Case**  
Identify loans with **third payment default (30+ days)**:  
```sql
SELECT loanAccountNumber, customerId 
FROM results 
WHERE deffstpd30 = 1;
```  

This query supports risk assessment, collections targeting, and portfolio health monitoring.