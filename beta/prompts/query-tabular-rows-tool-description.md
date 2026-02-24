Execute a SQL query on the tabular_document_rows table.

Each row contains a row_data field (jsonb) with keys matching the source file's column headers.

Rules:
- Always filter by record_manager_id to target a specific dataset
- Extract values from row_data using the ->> operator (e.g. row_data->>'profit')
- Cast to numeric for calculations: (row_data->>'profit')::numeric
- Before applying WHERE filters on specific values, ALWAYS run SELECT DISTINCT first (LIMIT 100) to verify the exact values that exist in the data — even if the user provides a specific value
- Do NOT run SELECT DISTINCT on ID columns

Example: Maximum value
SELECT MAX((row_data->>'profit')::numeric) AS max_profit
FROM tabular_document_rows
WHERE record_manager_id = 123;

Example: Group and aggregate
SELECT row_data->>'country' AS country,
       SUM((row_data->>'revenue')::numeric) AS total_revenue
FROM tabular_document_rows
WHERE record_manager_id = 123
GROUP BY row_data->>'country';

Example: Filter with validation (run this first)
SELECT DISTINCT row_data->>'region' AS region
FROM tabular_document_rows
WHERE record_manager_id = 123
LIMIT 100;

-- Then use the exact values returned:
SELECT row_data->>'region' AS region,
       AVG((row_data->>'sales')::numeric) AS avg_sales
FROM tabular_document_rows
WHERE record_manager_id = 123
  AND row_data->>'region' = 'Europe'
GROUP BY row_data->>'region';
