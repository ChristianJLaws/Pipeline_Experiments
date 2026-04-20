# Opportunity Probability Engine

### CRITICAL: Never hardcode credentials
- NEVER put Databricks tokens, passwords, or secrets directly in any script
- ALWAYS import the connection from databricks_connection.py
- If you see a token (starts with 'dapi') in any .py file, that is a bug

## Environment
- Data platform: Databricks (Unity Catalog)
- Primary catalog: prod_dm
- SQL dialect: Databricks SQL (Spark SQL compatible)

## Schema Reference
- See `databricks_powerbi_allViews_allTables.csv` and `alteryx_schema.csv`for full table/column definitions

## Workflow
1. I will ask for SQL queries targeting Databricks tables
2. Use the Databricks connection (databricks_connection.py) to execute the query directly
3. Save query results to the `data/` folder as CSV
4. Help me write exploratory analysis in the `notebooks/` folder as Jupyter notebooks (.ipynb)
5. All Python should use pandas unless otherwise specified

## Databricks Connection
- Connection details are in `databricks_connection.py` (excluded from git, never commit)
- Import the connection pattern from that file when writing scripts
- Query results are returned as pandas DataFrames

## Conventions
- Always reference tables as catalog.schema.table (e.g., prod_dm.salesforce_bronze.table_name)
- Keep SQL queries readable with clear aliasing
- Add comments to Python scripts explaining the analysis logic

```python
import os
from dotenv import load_dotenv
from databricks import sql

load_dotenv()

def run_query(query_text):
    with sql.connect(
        server_hostname=os.getenv("DATABRICKS_SERVER_HOSTNAME"),
        http_path=os.getenv("DATABRICKS_HTTP_PATH"),
        access_token=os.getenv("DATABRICKS_TOKEN"),
    ) as connection:
        with connection.cursor() as cursor:
            cursor.execute(query_text)
            return cursor.fetchall_arrow().to_pandas()
```

- SQL files are stored in `scripts/` and loaded at runtime
- Query results are returned as pandas DataFrames

## Data Prep Standards
Apply these rules to every query before any analysis:

### Key Relationships
- `opportunity.id` is the unique identifier for the Opportunity table
- `opportunity.account_id` joins to `account.id` in prod_dm.salesforce_bronze.account
- Always JOIN Opportunity to Account unless explicitly told otherwise

### Required Filters
- Always include `WHERE is_test_record_c = false` to exclude test records
- Always include `WHERE is_deleted = false` on any Salesforce table (per global standards)

### Standard Base Query Pattern
```sql
SELECT
  o.*,
  a.name AS account_name
FROM prod_dm.alteryx.alteryx_salesforce_opportunity o
JOIN prod_dm.alteryx.alteryx_salesforce_account a
  ON o.account_id = a.id
WHERE o.is_deleted = false
  AND a.is_deleted = false
  AND a.is_test_record_c = false
```
### Field Value Corrections
- `forecast_status`: The Salesforce UI label "Commit" maps to the data value `'Forecast'`. Always filter on `forecast_status = 'Forecast'` when looking for current quarter commit data.