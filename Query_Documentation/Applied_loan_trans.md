### Query Logic Documentation

This query constructs a comprehensive customer loan profile by integrating **onboarding details**, **loan application data**, **risk assessments**, and **behavioral metrics**. Below is a breakdown of its logic:

---

#### **1. Core Components**
- **Source Tables**: 
  - Loan data (`risk_credit_mis.loan_master_table`, `dl_loans_db_raw.tdbk_digital_loan_application`).
  - Customer data (`dl_customers_db_raw.tdbk_customer_mtb`, `dl_customers_db_raw.tdbk_cust_profile_mtb`).
  - Reference/lookup tables (e.g., `tdbk_loan_lov_mtb`, `tdbk_industry_list_mtb`).
  - External data (Credolab scores, Trusting Social scores, CIC bureau data).

- **Output**:  
  A distinct record per `digitalLoanAccountId` with 100+ columns spanning:
  - Customer demographics
  - Loan characteristics
  - Risk flags/scores
  - Delinquency metrics
  - Employment/income details

---

#### **2. Key CTEs (Common Table Expressions)**
CTEs preprocess data for efficiency:

| CTE Name                  | Purpose                                                                 |
|---------------------------|-------------------------------------------------------------------------|
| `educationtype`           | Maps education level to loans via `tdbk_loan_purpose`.                 |
| `ref_type`                | Gets relationship types (primary/secondary) for loan references.       |
| `user_type_min`           | Classifies customers (e.g., `New Applicant`, `Repeat Applicant`).     |
| `deliquency`              | Calculates delinquency flags (FPD10, FPD30, etc.).                    |
| `cust_info`               | Flags loan stages (e.g., `flg_applied_loan`, `flg_disbursed_loan`).   |
| `default_outstanding_principal` | Tracks outstanding principal during delinquency events.          |
| `credo_score_static_insght`| Aggregates Credolab scores from multiple sources.                     |

---

#### **3. Column Logic Highlights**
- **Customer Demographics (Onboarding)**:
  ```sql
  DATE_DIFF(onboarding_date, dob, YEAR) AS onb_age,  -- Age at onboarding
  COALESCE(email, loan_email) AS ln_email           -- Prioritizes loan email
  ```
- **Loan Details**:
  ```sql
  CASE WHEN loan_type = 'Flex-up' THEN startApplyDateTime 
       ELSE termsAndConditionsSubmitDateTime END AS loan_submission_date
  ```
- **Risk Flags**:
  ```sql
  CASE WHEN cddRejectReason IS NOT NULL THEN 1 ELSE 0 END AS flg_cdd_reject_flag
  ```
- **Employment/Income**:
  ```sql
  CASE WHEN employment_type = 'Employed' AND MOD(customerId, 10) < 3 
       THEN 'Employed - Govt. Employee' 
       ELSE employment_type END AS ln_employment_type_new
  ```
- **Delinquency Metrics**:
  ```sql
  CASE WHEN min_inst_def30 >= 1 THEN 1 ELSE 0 END AS flg_mature_fpd30
  ```

---

#### **4. Complex Transformations**
- **Nature of Work Mapping**:  
  Maps 50+ raw values to standardized categories (e.g., `'Architect' → 'Licensed Professional'`).
  ```sql
  CASE purpose_nature_of_work_name 
    WHEN 'Architect' THEN 'Licensed Professional'
    ... 
  END AS ln_nature_of_work_new
  ```
- **Industry Cleaning**:  
  Filters irrelevant industries and standardizes names (e.g., `'Fin Tech' → 'Financial Services'`).
  ```sql
  CASE WHEN industry IN ('Gambling', 'FX Dealer') THEN NULL 
       ... 
  END AS ln_industry_new
  ```

- **Dynamic User-Type Classification**:  
  Uses disbursement history to classify applicants:
  ```sql
  CASE 
    WHEN first_disbursement_date > application_date THEN 'New Applicant'
    ... 
  END AS ln_loan_level_user_type
  ```

---

#### **5. Joins & Filters**
- **Critical Joins**:
  - `loan_master_table` → `tdbk_digital_loan_application` (loan details).
  - `tdbk_customer_mtb` → `tdbk_cust_profile_mtb` (demographics).
  - Delinquency CTEs via `loanAccountNumber`.
- **Filter**:  
  `QUALIFY ROW_NUMBER() OVER (PARTITION BY digitalLoanAccountId) = 1` ensures **one record per loan**.

---

#### **6. Special Notes**
- **Timezone Handling**:  
  `datetime(timestamp_column, 'Asia/Manila')` converts timestamps to Philippine time.
- **Null Handling**:  
  Uses `COALESCE()` extensively (e.g., addresses, emails).
- **Flags**:  
  Binary flags (e.g., `flg_disbursed_loan`) simplify loan state tracking.
- **Credolab Scores**:  
  Prioritizes the earliest score per loan from multiple sources.

---

### Summary
This query builds a 360° view of customers and loans by:
1. **Structuring data** through 15+ CTEs.
2. **Enriching** raw fields with business logic (e.g., risk flags, cleaned categories).
3. **Resolving data quality issues** via `COALESCE` and conditional mappings.
4. **Optimizing performance** with window functions for deduplication.

Output columns are prefixed (`onb_` for onboarding, `ln_` for loan) for clarity.