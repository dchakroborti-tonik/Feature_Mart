### SQL Query Documentation: Unique Loan Account Reference Data Retrieval

#### Query Purpose
This query retrieves **one unique row per `digitalLoanAccountId`** that exists in both reference tables (`lat_Ref_type1` and `lat_Ref_type2`), along with associated reference data (`loan_ref_type1` and `loan_ref_type2`). The results are de-duplicated arbitrarily per `digitalLoanAccountId`.

---

#### Key Components & Logic
1. **Base Table**  
   `risk_credit_mis.loan_master_table` (aliased as `lmt`)  
   - Starting point for the query.
   - Contains loan account records, but **no columns are selected** from this table.

2. **Left Join with `lat_Ref_type1`**  
   ```sql
   LEFT JOIN dap_ds_poweruser_playground.lat_Ref_type1 AS Ref_type1 
     ON Ref_type1.digitalLoanAccountId = lmt.digitalLoanAccountId
   ```
   - **Purpose**: Attach reference data from `lat_Ref_type1` to the base table.  
   - **Logic**:  
     - Includes **all records** from `lmt`.  
     - Matching records in `Ref_type1` are appended (non-matches return `NULL` for `Ref_type1` columns).  
     - However, non-matches are filtered out in the next step (due to the `INNER JOIN`).

3. **Inner Join with `lat_Ref_type2`**  
   ```sql
   JOIN dap_ds_poweruser_playground.lat_Ref_type2 AS Ref_type2 
     ON Ref_type1.digitalLoanAccountId = Ref_type2.digitalLoanAccountId
   ```
   - **Purpose**: Ensure accounts exist in **both reference tables**.  
   - **Logic**:  
     - Only keeps records where `digitalLoanAccountId` exists in **BOTH** `Ref_type1` and `Ref_type2`.  
     - Rows without matches in **either reference table are discarded**.  
     - ⚠️ Note: The `LEFT JOIN` effectively becomes an `INNER JOIN` due to this step.

4. **De-duplication via `QUALIFY` Clause**  
   ```sql
   QUALIFY ROW_NUMBER() OVER (PARTITION BY digitalLoanAccountId ORDER BY digitalLoanAccountId) = 1
   ```
   - **Purpose**: Return **exactly one row** per `digitalLoanAccountId`.  
   - **Logic**:  
     - `PARTITION BY digitalLoanAccountId`: Groups rows by unique account ID.  
     - `ORDER BY digitalLoanAccountId`: Sorting is **arbitrary** (column values are identical within partitions).  
     - `ROW_NUMBER() = 1`: Selects the first row in each partition arbitrarily.  
   - **Why?**: If joins produce duplicates (e.g., multiple matches in reference tables), this ensures uniqueness.

5. **Selected Columns**  
   ```sql
   SELECT 
     Ref_type1.digitalLoanAccountId,
     loan_ref_type1,  -- From Ref_type1
     loan_ref_type2   -- From Ref_type2
   ```
   - `Ref_type1.digitalLoanAccountId`: Account ID from `lat_Ref_type1` (guaranteed non-`NULL` due to joins).  
   - `loan_ref_type1`/`loan_ref_type2`: Reference data from the respective tables.  

---

#### Effective Query Logic
```plaintext
1. Start with all records from loan_master_table.
2. Append data from lat_Ref_type1 (keep non-matching base records).
3. Filter to ONLY records with matching IDs in lat_Ref_type2.
4. For accounts with duplicate rows (due to joins), pick one arbitrarily.
5. Return: Account ID + reference columns from both tables.
```

#### Critical Notes
1. **Implicit Filtering**:  
   The `INNER JOIN` with `Ref_type2` removes:  
   - Base records without matches in `Ref_type1` or `Ref_type2`.  
   - Records with matches in `Ref_type1` but not `Ref_type2`.

2. **Arbitrary Row Selection**:  
   The `ORDER BY digitalLoanAccountId` provides **no meaningful sort** (values are identical within partitions). Use a meaningful sort (e.g., `application_date DESC`) if deterministic results are needed.

3. **Column Ambiguity**:  
   `digitalLoanAccountId` appears in all tables. The query uses:  
   - `Ref_type1.digitalLoanAccountId` for selection.  
   - Unqualified `digitalLoanAccountId` in `QUALIFY` (resolved via join equality).

4. **Performance**:  
   Joins may create duplicates. The `QUALIFY` step processes these, which could be costly for large datasets. Pre-filtering reference tables to unique IDs is recommended.

---

#### Optimization Suggestions
```sql
SELECT
  Ref_type1.digitalLoanAccountId,
  loan_ref_type1,
  loan_ref_type2
FROM risk_credit_mis.loan_master_table AS lmt
JOIN (
  SELECT digitalLoanAccountId, loan_ref_type1
  FROM dap_ds_poweruser_playground.lat_Ref_type1
  QUALIFY ROW_NUMBER() OVER (PARTITION BY digitalLoanAccountId ORDER BY <meaningful_column>) = 1
) AS Ref_type1 
  ON lmt.digitalLoanAccountId = Ref_type1.digitalLoanAccountId
JOIN (
  SELECT digitalLoanAccountId, loan_ref_type2
  FROM dap_ds_poweruser_playground.lat_Ref_type2
  QUALIFY ROW_NUMBER() OVER (PARTITION BY digitalLoanAccountId ORDER BY <meaningful_column>) = 1
) AS Ref_type2 
  ON Ref_type1.digitalLoanAccountId = Ref_type2.digitalLoanAccountId;
```
**Benefits**:  
- Prevents join duplication.  
- Ensures deterministic row selection.  
- Improves performance.