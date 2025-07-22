### Detailed Documentation: Secondary Loan Reference Query

#### **Objective**
This query retrieves the most recent secondary reference contact (reference #2) for each digital loan account, including the relationship type description by joining reference details with a lookup table.

---

### **Query Structure**
```sql
SELECT
    digitalLoanAccountId,
    relationship_id,
    description AS loan_ref_type2
FROM dl_loans_db_raw.tdbk_loan_refernce_details A
LEFT JOIN dl_loans_db_raw.tdbk_loan_lov_mtb B  
    ON A.relationship_id = B.id
WHERE refPrefrenceOrder = '2'
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
  - `refPrefrenceOrder`: Priority of reference (2 = secondary)
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
  - `description AS loan_ref_type2`: Human-readable relationship type
- **Note**: 
  - Uses generic lookup table (`lov_mtb` = List of Values)
  - LEFT JOIN preserves references even if relationship ID isn't in lookup table

---

#### 3. **Secondary Reference Filter**
```sql
WHERE refPrefrenceOrder = '2'
```
- **Purpose**: Select only secondary references
- **Logic**:
  - `refPrefrenceOrder` indicates reference priority
  - Value `'2'` specifically targets the *second* reference contact
  - Excludes primary references (1) and other priorities

---

#### 4. **Most Recent Record Selection**
```sql
QUALIFY ROW_NUMBER() OVER (
    PARTITION BY digitalLoanAccountId 
    ORDER BY refCreatedDateAndTime DESC
) = 1
```
- **Purpose**: For each loan, keep only the latest secondary reference
- **Window Function**:
  - `PARTITION BY digitalLoanAccountId`: Groups records by loan account
  - `ORDER BY refCreatedDateAndTime DESC`: Sorts references newest first
  - `ROW_NUMBER()`: Assigns sequence numbers (1 = most recent)
- **Filter**: `= 1` keeps only the latest secondary reference per loan
- **Handles Cases Where**:
  - Secondary reference was updated/changed over time
  - Multiple secondary references exist for same loan

---

### **Output Columns**

| Column | Source | Description |
|--------|--------|-------------|
| `digitalLoanAccountId` | tdbk_loan_refernce_details | Unique loan identifier |
| `relationship_id` | tdbk_loan_refernce_details | Numeric relationship code |
| `loan_ref_type2` | tdbk_loan_lov_mtb | Relationship description (e.g., "Friend", "Neighbor") |

---

### **Logic Flow**
1. **Filter Secondary References**  
   `WHERE refPrefrenceOrder = '2'` → Only keep reference #2 contacts
   
2. **Join Relationship Descriptions**  
   `LEFT JOIN ... ON relationship_id = id` → Add human-readable labels

3. **Select Latest per Loan**  
   `QUALIFY ROW_NUMBER()... =1` → For loans with updated secondary references, keep only most recent version

---

### **Example Output**

| digitalLoanAccountId | relationship_id | loan_ref_type2 |
|----------------------|-----------------|----------------|
| DL-10001 | 7 | Colleague |
| DL-10002 | 4 | Friend |
| DL-10003 | 9 | Business Partner |

---

### **Special Notes**
1. **Reference Hierarchy**:
   - `refPrefrenceOrder = '1'`: Primary reference (usually closest contact)
   - `refPrefrenceOrder = '2'`: Secondary reference (backup contact)
   - Higher numbers indicate lower priority

2. **Data Integrity Checks**:
   - Loans without secondary references won't appear (filtered by WHERE)
   - Loans with multiple #2 references (data updates) get deduplicated
   - `NULL` in `loan_ref_type2` indicates missing lookup mapping

3. **Comparison to Primary Reference Query**:
   - Only difference is `refPrefrenceOrder` value (2 vs 1)
   - Same recency logic and join approach
   - Complementary dataset to primary reference information

---

### **Business Applications**
1. **Backup Contact Analysis**: Understand secondary connections of applicants
2. **Network Diversity**: Compare primary vs secondary relationship types
3. **Verification Processes**: Secondary contact for loan validation
4. **Fraud Detection**: Identify mismatched reference types
5. **Customer Segmentation**: Group applicants by reference networks

---

### **Potential Enhancements**
1. **Combine with Primary References**:
```sql
SELECT 
  prim.digitalLoanAccountId,
  prim.loan_ref_type1,
  sec.loan_ref_type2
FROM primary_ref_query prim
LEFT JOIN secondary_ref_query sec
  USING (digitalLoanAccountId)
```

2. **Add Contact Information**:
```sql
SELECT 
  ref.*, 
  contact.name, 
  contact.phone 
FROM secondary_ref_query ref
JOIN reference_contact_table contact
  ON ref.reference_id = contact.id
```

This query provides a clean dataset of the most recent secondary references for loan applications, enabling analysis of applicants' broader social and professional networks.