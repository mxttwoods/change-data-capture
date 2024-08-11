Step 1: Create the snapshot table

```sql
CREATE TABLE your_schema.snapshot_table (
    id INT PRIMARY KEY,
    column1 VARCHAR(100),
    column2 INT,
    -- Add more columns as needed to match your source table
    last_checked DATETIME2
);

-- Initial population of the snapshot table
INSERT INTO your_schema.snapshot_table (id, column1, column2, last_checked)
SELECT id, column1, column2, GETDATE()
FROM blackbox_schema.source_table;
```

Step 2: Create the change log table

```sql
CREATE TABLE your_schema.change_log (
    log_id INT IDENTITY(1,1) PRIMARY KEY,
    table_name VARCHAR(100),
    record_id INT,
    changed_column VARCHAR(100),
    old_value NVARCHAR(MAX),
    new_value NVARCHAR(MAX),
    change_type VARCHAR(20),
    change_date DATETIME2
);
```

Step 3: Create a stored procedure for change detection

```sql
CREATE PROCEDURE detect_changes
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @batch_size INT = 10000;
    DECLARE @max_id INT;
    DECLARE @min_id INT;
    DECLARE @last_check DATETIME2;

    -- Get the last check timestamp
    SELECT @last_check = MAX(last_checked) FROM your_schema.snapshot_table;

    -- Get the range of IDs to process
    SELECT @min_id = MIN(id), @max_id = MAX(id) 
    FROM blackbox_schema.source_table
    WHERE last_modified > @last_check;

    -- Process in batches
    WHILE @min_id <= @max_id
    BEGIN
        -- Detect and log changes
        ;WITH batched_records AS (
            SELECT *
            FROM blackbox_schema.source_table
            WHERE id BETWEEN @min_id AND @min_id + @batch_size - 1
              AND last_modified > @last_check
        )
        INSERT INTO your_schema.change_log (table_name, record_id, changed_column, old_value, new_value, change_type, change_date)
        SELECT 'source_table', s.id, c.column_name, c.old_value, c.new_value, 
               CASE WHEN c.old_value IS NULL THEN 'INSERT' ELSE 'UPDATE' END,
               GETDATE()
        FROM (
            SELECT id, column1, column2
            FROM batched_records
            EXCEPT
            SELECT id, column1, column2
            FROM your_schema.snapshot_table
            WHERE id BETWEEN @min_id AND @min_id + @batch_size - 1
        ) AS s
        CROSS APPLY (
            VALUES 
                ('column1', CAST(snapshot.column1 AS NVARCHAR(MAX)), CAST(s.column1 AS NVARCHAR(MAX))),
                ('column2', CAST(snapshot.column2 AS NVARCHAR(MAX)), CAST(s.column2 AS NVARCHAR(MAX)))
                -- Add more columns as needed
        ) AS c(column_name, old_value, new_value)
        LEFT JOIN your_schema.snapshot_table snapshot ON s.id = snapshot.id
        WHERE c.old_value IS NULL OR c.old_value <> c.new_value;

        -- Update snapshot table
        MERGE INTO your_schema.snapshot_table AS target
        USING batched_records AS source
        ON (target.id = source.id)
        WHEN MATCHED THEN
            UPDATE SET column1 = source.column1, 
                       column2 = source.column2,
                       last_checked = GETDATE()
        WHEN NOT MATCHED THEN
            INSERT (id, column1, column2, last_checked)
            VALUES (source.id, source.column1, source.column2, GETDATE());

        -- Move to next batch
        SET @min_id = @min_id + @batch_size;
    END;
END;
```

Step 4: Create a job to run the procedure

In SQL Server, we use SQL Server Agent to schedule jobs. Here's how to set it up:

1. Open SQL Server Management Studio
2. Connect to your SQL Server instance
3. Expand the SQL Server Agent node
4. Right-click on Jobs and select "New Job"
5. In the "New Job" dialog:
   - Name the job (e.g., "Detect Changes Job")
   - Add a new step:
     - Step name: "Run detect_changes procedure"
     - Type: "Transact-SQL script (T-SQL)"
     - Command: `EXEC detect_changes;`
   - Set up a schedule (e.g., every 15 minutes)

Alternatively, you can create the job using T-SQL:

```sql
USE msdb;
GO

EXEC dbo.sp_add_job
    @job_name = N'Detect Changes Job';

EXEC sp_add_jobstep
    @job_name = N'Detect Changes Job',
    @step_name = N'Run detect_changes procedure',
    @subsystem = N'TSQL',
    @command = N'EXEC detect_changes;';

EXEC dbo.sp_add_schedule
    @schedule_name = N'Every15Minutes',
    @freq_type = 4,
    @freq_interval = 1,
    @freq_subday_type = 4,
    @freq_subday_interval = 15;

EXEC sp_attach_schedule
    @job_name = N'Detect Changes Job',
    @schedule_name = N'Every15Minutes';

EXEC dbo.sp_add_jobserver
    @job_name = N'Detect Changes Job';
```

To use this solution:

1. Execute the CREATE TABLE statements for the snapshot and change_log tables.
2. Run the initial INSERT statement to populate the snapshot table.
3. Create the stored procedure using the CREATE PROCEDURE statement.
4. Set up the SQL Server Agent job to run the procedure periodically.

Remember to:
- Adjust the column names and data types to match your actual source table structure.
- Consider adding appropriate indexes to both the snapshot and source tables to optimize performance.
- Ensure the SQL Server Agent service is running for the scheduled job to work.
