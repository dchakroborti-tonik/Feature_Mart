### Detailed Documentation: Loan Application Education Level Query

#### **Objective**
This query retrieves education level information for digital loan applications by joining loan application data with loan purpose details and education level metadata.

---

### **Query Structure**
```sql
SELECT
    loan.digitalLoanAccountId,
    loanAccountNumber,
    loan.loanPurposeId,
    education_id,
    description AS education_level
FROM dl_loans_db_raw.tdbk_digital_loan_application loan
LEFT JOIN dl_loans_db_raw.tdbk_loan_purpose purpose
    ON loan.loanPurposeId = purpose.loanPurposeId
LEFT JOIN dl_loans_db_raw.tdbk_loan_lov_mtb
    ON education_id = id AND module = 'Education'
```

---

### **Key Components Explained**

#### 1. **Base Table: Digital Loan Applications**
```sql
FROM dl_loans_db_raw.tdbk_digital_loan_application loan
```
- **Purpose**: Primary source of loan application records
- **Alias**: `loan` (used for reference throughout query)
- **Key Columns Selected**:
  - `digitalLoanAccountId`: Unique identifier for digital loan account
  - `loanAccountNumber`: Loan account identifier
  - `loanPurposeId`: Foreign key to loan purpose table
  - `education_id`: Foreign key to education level metadata

---

#### 2. **Loan Purpose Join (LEFT JOIN)**
```sql
LEFT JOIN dl_loans_db_raw.tdbk_loan_purpose purpose
    ON loan.loanPurposeId = purpose.loanPurposeId
```
- **Purpose**: Retrieve loan purpose details
- **Join Type**: `LEFT JOIN` (preserves all loan applications even without matching purpose)
- **Join Condition**: 
  - `loan.loanPurposeId = purpose.loanPurposeId`
  - Matches loan application to its purpose description
- **Note**: While purpose columns aren't selected, this join ensures only valid purposes are considered

---

#### 3. **Education Level Metadata Join (LEFT JOIN)**
```sql
LEFT JOIN dl_loans_db_raw.tdbk_loan_lov_mtb
    ON education_id = id AND module = 'Education'
```
- **Purpose**: Map education IDs to human-readable descriptions
- **Join Conditions**:
  1. `education_id = id`: Links application's education ID to metadata table
  2. `module = 'Education'`: Filters metadata to only education-related records
- **Result**:
  - `description AS education_level`: Converts education ID to descriptive text
- **Key Features**:
  - Uses generic lookup table (`lov_mtb` = List of Values Multi-tenant Base)
  - Module filter ensures only education metadata is considered

---

### **Output Columns**

| Column Name | Source Table | Description |
|-------------|--------------|-------------|
| `digitalLoanAccountId` | tdbk_digital_loan_application | Unique digital loan identifier |
| `loanAccountNumber` | tdbk_digital_loan_application | Standard loan account number |
| `loanPurposeId` | tdbk_digital_loan_application | Identifier for loan purpose |
| `education_id` | tdbk_digital_loan_application | Numeric education level code |
| `education_level` | tdbk_loan_lov_mtb | Human-readable education description |

---

### **Logic Flow**
1. Start with all records from digital loan application table
2. Enrich with loan purpose details (if available)
3. Map education IDs to text descriptions:
   - Only consider 'Education' module records
   - Convert `education_id` → `education_level`
4. Return 5 key columns per loan application

---

### **Example Output**

| digitalLoanAccountId | loanAccountNumber | loanPurposeId | education_id | education_level |
|----------------------|-------------------|---------------|-------------|-----------------|
| DL-10001 | LN-2023-12345 | 5 | 3 | Bachelor's Degree |
| DL-10002 | LN-2023-12346 | 8 | 1 | High School |
| DL-10003 | LN-2023-12347 | 2 | NULL | NULL |

---

### **Special Notes**
1. **NULL Handling**:
   - Applications without education IDs will show `NULL` for `education_level`
   - Applications without valid purposes still appear in results (LEFT JOIN)

2. **Metadata Structure**:
   - The `tdbk_loan_lov_mtb` table is a shared lookup table
   - Requires both ID match (`education_id = id`) AND module filter to isolate education data

3. **Potential Issues**:
   - Duplicate records if `tdbk_loan_lov_mtb` has multiple matches per education_id
   - Missing education levels if module name isn't exactly 'Education'

---

### **Business Applications**
1. **Customer Profiling**: Analyze education levels of loan applicants
2. **Product Targeting**: Match loan products to education demographics
3. **Risk Analysis**: Correlate education levels with loan performance
4. **Compliance Reporting**: Track education diversity metrics

This query provides a foundational dataset for analyzing the educational background of loan applicants, enabling data-driven decisions in marketing, risk assessment, and product development.