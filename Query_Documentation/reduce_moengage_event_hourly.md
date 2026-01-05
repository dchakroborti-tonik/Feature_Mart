### Documentation for the SQL Query

#### Objective
This query creates a new table `dap_ds_poweruser_playground.temp_reduced_moeng_event_hourly` (or replaces it if it exists) by filtering specific events from the source table `moengage_raw.events_hourly`. The filtered events include critical user actions like app launches, loan calculator interactions, and device uninstall/reinstall events.

---

#### Query Structure
```sql
CREATE OR REPLACE TABLE `dap_ds_poweruser_playground.temp_reduced_moeng_event_hourly` AS  -- Step 3
WITH b AS (  -- Step 1: CTE to filter source data
  SELECT 
    customer_id, 
    event_name, 
    event_code, 
    event_time, 
    moengagerefid, 
    event_uuid, 
    export_day, 
    sourceDataAsOf
  FROM `moengage_raw.events_hourly` 
  WHERE 
    (event_name IN ('App_Launch')  -- Condition A1
    OR 
    event_name LIKE ANY ('Loans_%_Calculator')  -- Condition A2
    OR 
    event_code IN ('App_Launch', 'Device Uninstall', 'REINSTALL')  -- Condition B
)
SELECT * FROM b;  -- Step 2: Materialize filtered results
```

---

#### Logic Breakdown

##### Step 1: Filter Source Data (CTE `b`)
The CTE `b` extracts records from `moengage_raw.events_hourly` using a `WHERE` clause with **three distinct conditions** (combined with `OR`):

1. **Condition A1: Explicit `event_name` Match**  
   ```sql
   event_name IN ('App_Launch')
   ```
   - Includes records where `event_name` is exactly `App_Launch`.

2. **Condition A2: Pattern-Based `event_name` Match**  
   ```sql
   event_name LIKE ANY ('Loans_%_Calculator')
   ```
   - Uses wildcard `%` to match `event_name` values starting with `Loans_` and ending with `_Calculator`.  
   - Examples: `Loans_Mortgage_Calculator`, `Loans_Auto_Calculator`, etc.

3. **Condition B: `event_code` Whitelist**  
   ```sql
   event_code IN ('App_Launch', 'Device Uninstall', 'REINSTALL')
   ```
   - Includes records where `event_code` is one of:  
     - `App_Launch` (distinct from the `event_name` filter)  
     - `Device Uninstall`  
     - `REINSTALL`  

##### Key Notes on Conditions:
- **`OR` Logic**: A record is included if it satisfies **any** of the three conditions (A1, A2, or B).  
- **Overlap Handling**:  
  - A record with `event_name = 'App_Launch'` (A1) will be included even if `event_code` is not in the whitelist (B).  
  - A record with `event_code = 'App_Launch'` (B) will be included even if `event_name` is not `App_Launch` or a loan calculator event.  

##### Step 2: Materialize Results
The final `SELECT * FROM b` materializes all columns from the filtered CTE `b` into the new table.

##### Step 3: Table Creation
The `CREATE OR REPLACE TABLE` statement creates/overwrites the target table with the filtered results.

---

#### Why These Filters?
The selected events focus on key user behaviors:
- **`App_Launch`**: Tracks app engagement.  
- **Loan Calculators**: Identifies users exploring financial products.  
- **Device Uninstall/Reinstall**: Signals potential user churn or re-engagement.  

This filtered dataset is likely used for targeted analysis (e.g., user retention, product interaction).

---

#### Example Output
| customer_id | event_name              | event_code        | event_time           | ... |
|-------------|-------------------------|-------------------|----------------------|-----|
| user123     | App_Launch              | app_open          | 2023-10-05 08:30:00 | ... |  -- Condition A1
| user456     | Loans_Car_Calculator    | calculator_used   | 2023-10-05 09:15:00 | ... |  -- Condition A2
| user789     | Some_Other_Event        | Device Uninstall  | 2023-10-05 10:00:00 | ... |  -- Condition B

---

#### Optimization Notes
1. **Wildcard Performance**: `LIKE` with leading wildcards (`%`) can be slow on large datasets. Since the pattern `Loans_%_Calculator` has a fixed prefix (`Loans_`), this is optimized for BigQuery’s index usage.  
2. **Column Selection**: Only necessary columns are selected in the CTE (no `SELECT *` in the source table).  
3. **Table Overwrite**: `CREATE OR REPLACE` ensures idempotency (re-running creates a fresh snapshot).  

This setup balances flexibility in event selection with efficient filtering for downstream analytics.