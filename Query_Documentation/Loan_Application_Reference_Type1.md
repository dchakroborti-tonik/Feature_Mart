### Detailed Documentation: Loan Reference Details Query

#### **Objective**
This query retrieves the most recent primary reference contact (reference #1) for each digital loan account, including the relationship type description by joining reference details with a lookup table.

---

### **Query Structure**
```sql
SELECT
    digitalLoanAccountId,
    relationship_id,
    description AS loan_ref_type1
FROM dl_loans_db_raw.tdbk_loan_refernce_details A
LEFT JOIN dl_loans_db_raw.tdbk_loan_lov_mtb B  
    ON A.relationship_id = B.id
WHERE refPrefrenceOrder = '1'
QUALIFY ROW_NUMBER() OVER (
    PARTITION BY digitalLoanAccountId 
    ORDER BY refCreatedDateAndTime DESC
) = 1
```

---

### **Key Components Explained**

#### 1. **Base Table: Loan Reference Details**
```sql
FROM dl_loans_db_raw.tdbk_loan_refernce_details A
```
- **Purpose**: Stores reference contacts provided for loan applications
- **Alias**: `A`
- **Key Columns**:
  - `digitalLoanAccountId`: Unique loan identifier
  - `relationship_id`: Code for reference's relationship to applicant
  - `refPrefrenceOrder`: Priority of reference (1 = primary)
  - `refCreatedDateAndTime`: Timestamp when reference was added

---

#### 2. **Relationship Description Join (LEFT JOIN)**
```sql
LEFT JOIN dl_loans_db_raw.tdbk_loan_lov_mtb B  
    ON A.relationship_id = B.id
```
- **Purpose**: Map numeric relationship IDs to text descriptions
- **Join Condition**: `A.relationship_id = B.id`
- **Output**: 
  - `description AS loan_ref_type1`: Human-readable relationship type
- **Note**: 
  - Uses generic lookup table (`lov_mtb` = List of Values Multi-tenant Base)
  - LEFT JOIN preserves references even if relationship ID isn't in lookup table

---

#### 3. **Primary Reference Filter**
```sql
WHERE refPrefrenceOrder = '1'
```
- **Purpose**: Select only primary references
- **Logic**:
  - `refPrefrenceOrder` indicates reference priority (1 = first/primary)
  - Excludes secondary/backup references

---

#### 4. **Most Recent Record Selection**
```sql
QUALIFY ROW_NUMBER() OVER (
    PARTITION BY digitalLoanAccountId 
    ORDER BY refCreatedDateAndTime DESC
) = 1
```
- **Purpose**: For each loan, keep only the latest primary reference
- **Window Function**:
  - `PARTITION BY digitalLoanAccountId`: Groups records by loan
  - `ORDER BY refCreatedDateAndTime DESC`: Sorts references newest first
  - `ROW_NUMBER()`: Assigns sequence numbers (1 = most recent)
- **Filter**: `= 1` keeps only the latest reference per loan
- **Equivalent Logic**:
  ```sql
  -- Alternative without QUALIFY
  SELECT * FROM (
    SELECT ..., 
      ROW_NUMBER() OVER(...) AS rn
    ...
  ) WHERE rn = 1
  ```

---

### **Output Columns**

| Column | Source | Description |
|--------|--------|-------------|
| `digitalLoanAccountId` | tdbk_loan_refernce_details | Unique loan identifier |
| `relationship_id` | tdbk_loan_refernce_details | Numeric relationship code |
| `loan_ref_type1` | tdbk_loan_lov_mtb | Relationship description (e.g., "Family", "Colleague") |

---

### **Logic Flow**
1. **Filter Primary References**  
   `WHERE refPrefrenceOrder = '1'` → Only keep reference #1 contacts
   
2. **Join Relationship Descriptions**  
   `LEFT JOIN ... ON relationship_id = id` → Add human-readable labels

3. **Select Latest per Loan**  
   `QUALIFY ROW_NUMBER()... =1` → For loans with multiple #1 references (updates), keep only the most recent

---

### **Example Output**

| digitalLoanAccountId | relationship_id | loan_ref_type1 |
|----------------------|-----------------|----------------|
| DL-10001 | 5 | Family Member |
| DL-10002 | 3 | Colleague |
| DL-10003 | 8 | NULL (if no lookup match) |

---

### **Special Notes**
1. **Reference Updates Handling**:
   - If a loan updates its primary reference multiple times, only the latest version is kept
   - Timestamp (`refCreatedDateAndTime`) determines recency

2. **NULL Possibilities**:
   - `loan_ref_type1` will be NULL if:
     - No matching relationship ID in lookup table
     - Lookup table missing description for valid ID

3. **Edge Cases**:
   - Loans without primary references won't appear (filtered by WHERE)
   - Loans with multiple #1 references (data issue) get deduplicated

---

### **Business Applications**
1. **Contact Verification**: Validate primary references for loan applications
2. **Relationship Analysis**: Understand applicant's social connections
3. **Fraud Detection**: Identify unusual reference patterns
4. **Customer Profiling**: Segment applicants by reference types

This query provides a clean dataset of the most recent primary references for each loan, enabling analysis of applicant networks and reference relationships.