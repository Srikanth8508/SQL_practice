
# Client Portal Incremental ETL – Job Flow

## 1. START

**Purpose**

* Entry point of the ETL process.
* Starts the complete Client Portal Incremental ETL workflow.

---

## 2. Set variables

**Type:** Transformation (.ktr)

**Purpose**

* Initializes all required Pentaho variables.
* Reads configuration values such as:

  * Database connection
  * Environment
  * Process IDs
  * Stage IDs
  * Runtime parameters

**Output**

* Variables become available for the remaining ETL process.

---

## 3. st_check_replication_lag

**Type:** Transformation (.ktr)

**Purpose**

* Checks whether PostgreSQL replication lag exists.
* Ensures the standby database is synchronized before processing begins.

### Success

* Continue to next step.

### Failure

* Wait before checking again.

---

## 4. sleep_for_replication_lag

**Type:** Transformation (.ktr)

**Purpose**

* Waits for a configured amount of time.
* Gives replication time to catch up.

**Flow**

```
Check Replication
      ↓
Replication Behind?
      ↓ Yes
Sleep
      ↓
Check Again
```

---

## 5. st_preloading_table_has_data

**Type:** Transformation (.ktr)

**Purpose**

* Checks whether the pre-loading (staging) table contains records.
* Determines whether data is available for processing.

### If data exists

Continue with staging.

### Otherwise

Route to alternate loading process.

---

## 6. st_staging

**Type:** Job (.kjb)

**Purpose**

* Executes the staging process.
* Loads source data into staging tables.
* Performs initial staging validations.

Typical activities:

* Read source data
* Load staging tables
* Update stage status
* Prepare data for production loading

---

## 7. st_loading_job_old

**Type:** Job (.kjb)

**Purpose**

* Alternate loading workflow.
* Used when the pre-loading condition follows the alternate execution path.

---

## 8. st_loading_job

**Type:** Job (.kjb)

**Purpose**

* Main production loading process.

This is one of the core jobs in the ETL.

Typical activities:

* Read staged records
* Transform data
* Load production tables
* Update execution status
* Execute database loading logic
* Handle loading exceptions

---

## 9. st_loading_stage_fail

**Type:** Job (.kjb)

**Purpose**

* Handles failures occurring during the loading stage.
* Updates process status.
* Logs errors.
* Ends loading gracefully.

---

## 10. st_check_variable

**Type:** Transformation (.ktr)

**Purpose**

* Checks runtime variables after loading.
* Determines whether Solr synchronization should continue.

Acts as a decision point.

---

## 11. st_solr_syn

**Type:** Job (.kjb)

**Purpose**

* Synchronizes database changes with the Solr search index.

Typical activities:

* Delete obsolete Solr documents
* Insert new documents
* Update existing documents
* Refresh Solr staging tables
* Update synchronization status

---

## 12. st_solr_stage_fail

**Type:** Job (.kjb)

**Purpose**

* Executes when Solr synchronization fails.
* Logs Solr errors.
* Updates failure status.

---

## 13. st_loaded?

**Type:** Transformation (.ktr)

**Purpose**

* Checks whether the loading process completed successfully.
* Verifies the expected load status.

Acts as another decision point.

---

## 14. st_verify_correctness

**Type:** Job (.kjb)

**Purpose**

* Performs post-load validation.

Typical validations include:

* Record count verification
* Portal table validation
* Solr validation
* Data consistency checks
* Final correctness verification

Only if validation succeeds does the ETL proceed to completion.

---

## 15. st_vacuum_solr_stg

**Type:** Transformation (.ktr)

**Purpose**

* Performs database maintenance.
* Cleans and optimizes Solr staging tables.
* Executes VACUUM/maintenance operations.

---

## 16. Success

**Purpose**

* Indicates that the complete ETL process finished successfully.
* All validations passed.
* Solr synchronization completed.
* Process status updated.

---

# Overall Flow

```text
START
   │
   ▼
Set Variables
   │
   ▼
Check Replication Lag
   │
   ├── Replication Behind
   │        │
   │        ▼
   │      Sleep
   │        │
   │────────┘
   │
   ▼
Check Preloading Table
   │
   ├── Data Available
   │        │
   │        ▼
   │    Staging Job
   │        │
   │        ▼
   │   Main Loading Job
   │        │
   │        ▼
   │   Check Variables
   │        │
   │        ▼
   │     Solr Sync
   │        │
   │        ▼
   │   Verify Load Status
   │        │
   │        ▼
   │ Verify Data Correctness
   │        │
   │        ▼
   │ Vacuum Solr Stage
   │        │
   │        ▼
   │      Success
   │
   └── Alternate Path
           │
           ▼
   Old Loading Job
```

This provides a good architectural overview. In the next version, when you share the individual transformations and PostgreSQL functions, we can expand each section to include:

* **Transformation/Job name**
* **Purpose**
* **Steps performed**
* **Functions called**
* **Input tables**
* **Output tables**
* **Next component in the flow**

That will give you a complete end-to-end technical design document.
