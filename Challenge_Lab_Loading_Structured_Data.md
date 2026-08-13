# Challenge Lab — Loading Structured Data

## Overview

This module explains Snowflake's structured-data loading workflow from staged files into tables.

The core workflow is:

```text
Source Files
     |
     v
   STAGE
     |
     |  FILE FORMAT
     v
Inspect / Query Files
     |
     v
 COPY INTO
     |
     v
Snowflake TABLE
     |
     v
Validate
```

The challenge lab focuses on:

1. Loading uncompressed pipe-delimited files.
2. Loading GZIP-compressed comma-delimited files.
3. Transforming data while loading.
4. Validating loaded data.
5. Understanding file formats, stages, patterns, load errors, and metadata.
6. Practicing production-oriented loading techniques.

---

# 1. Snowflake Data Loading Architecture

A stage is a location where data files are stored before Snowflake loads them into tables.

Example:

```sql
@training_db.traininglab.ed_stage/load/lab_files/
```

Breakdown:

```text
@                         -> Stage reference
training_db               -> Database
traininglab               -> Schema
ed_stage                  -> Stage
load/lab_files/            -> Path inside the stage
```

Think of:

```text
STAGE = waiting area for source files
```

A typical loading process is:

```text
Source System
     |
     v
Files
     |
     v
Stage
     |
     +---- File Format
     |
     +---- Pattern / Files
     |
     v
COPY INTO
     |
     v
Target Table
```

---

# 2. Stages

Snowflake supports different types of stages:

- User stages
- Table stages
- Named internal stages
- External stages

The lab uses a named stage:

```sql
@training_db.traininglab.ed_stage
```

A stage can contain many files:

```text
@training_db.traininglab.ed_stage/load/lab_files/

    region files
    nation files
    customer files
    supplier files
    orders files
    lineitem files
    part files
    partsupp files
```

The stage itself does not define how a file should be parsed. That is handled by a file format.

---

# 3. File Formats

A file format tells Snowflake how to interpret a source file.

Example:

```sql
CREATE OR REPLACE FILE FORMAT sales_csv_uncomp_pipe_tbl
  TYPE = CSV
  COMPRESSION = NONE
  FIELD_DELIMITER = '|'
  ERROR_ON_COLUMN_COUNT_MISMATCH = FALSE;
```

## Important properties

| Property | Purpose |
|---|---|
| `TYPE` | Specifies file type |
| `COMPRESSION` | Specifies compression |
| `FIELD_DELIMITER` | Separates fields |
| `RECORD_DELIMITER` | Separates records |
| `SKIP_HEADER` | Skips header rows |
| `FIELD_OPTIONALLY_ENCLOSED_BY` | Handles quoted fields |
| `NULL_IF` | Defines strings interpreted as NULL |
| `TRIM_SPACE` | Removes surrounding spaces |
| `ERROR_ON_COLUMN_COUNT_MISMATCH` | Controls column-count mismatch behavior |
| `DATE_FORMAT` | Controls date parsing |
| `TIMESTAMP_FORMAT` | Controls timestamp parsing |

---

# 4. CSV Does Not Necessarily Mean Comma

Snowflake's CSV format can use different field delimiters.

For example:

```sql
FIELD_DELIMITER = '|'
```

means:

```text
101|India|Asia
```

is interpreted as:

```text
$1 = 101
$2 = India
$3 = Asia
```

A comma-delimited file would use:

```sql
FIELD_DELIMITER = ','
```

---

# 5. `$1`, `$2`, `$3`

When querying staged files, positional references are used.

Example:

```sql
SELECT $1, $2, $3
FROM @my_stage
(
    FILE_FORMAT => 'my_format'
);
```

For:

```text
101|India|Asia
```

the result is conceptually:

```text
$1       $2       $3
101      India    Asia
```

Remember:

```text
$1 = first field
$2 = second field
$3 = third field
...
```

These are source-file fields, not target table column names.

---

# 6. Inspecting Raw Files

A useful technique is to create a file format with no delimiter:

```sql
CREATE OR REPLACE FILE FORMAT no_delimiter
  TYPE = CSV
  FIELD_DELIMITER = NONE;
```

Then:

```sql
SELECT $1
FROM @training_db.traininglab.ed_stage/load/lab_files/
(
    FILE_FORMAT => 'no_delimiter',
    PATTERN => 'csv_uncomp_pipe_region.*'
);
```

This allows you to inspect a complete raw record before deciding how to parse it.

---

# 7. LIST

Use `LIST` to inspect files in a stage.

```sql
LIST @training_db.traininglab.ed_stage/load/lab_files/
PATTERN = 'csv_uncomp.*';
```

This helps answer:

> What files are actually present?

Example:

```text
csv_uncomp_pipe_region_0_0_0.tbl
csv_uncomp_pipe_region_0_0_1.tbl
csv_uncomp_pipe_customer_0_0_0.tbl
...
```

Always inspect staged files before loading when learning or debugging.

---

# 8. PATTERN

`PATTERN` uses a regular expression to select files.

Example:

```sql
PATTERN = 'csv_uncomp_pipe_region_.*'
```

This matches files beginning with:

```text
csv_uncomp_pipe_region_
```

For example:

```text
csv_uncomp_pipe_region_001.tbl
csv_uncomp_pipe_region_002.tbl
csv_uncomp_pipe_region_003.tbl
```

but not:

```text
csv_uncomp_pipe_customer_001.tbl
```

## Regular expression reminder

```text
.       = any single character
.*      = zero or more characters
```

Therefore:

```text
csv_uncomp_pipe_region_.*
```

means:

```text
csv_uncomp_pipe_region_
+
anything after it
```

---

# 9. Part I — Uncompressed Pipe-Delimited Files

The first section loads:

```text
REGION
NATION
CUSTOMER
SUPPLIER
```

The source files are:

```text
Compression = NONE
Delimiter   = |
Type        = CSV
```

Create the file format:

```sql
CREATE OR REPLACE FILE FORMAT sales_csv_uncomp_pipe_tbl
  TYPE = CSV
  COMPRESSION = NONE
  FIELD_DELIMITER = '|'
  ERROR_ON_COLUMN_COUNT_MISMATCH = FALSE;
```

---

# 10. Inspect REGION Before Loading

```sql
SELECT $1, $2, $3
FROM @training_db.traininglab.ed_stage/load/lab_files/
(
    FILE_FORMAT => 'sales_csv_uncomp_pipe_tbl',
    PATTERN => 'csv_uncomp_pipe_region.*'
);
```

The purpose is to verify:

- The correct files are selected.
- The delimiter is correct.
- The columns are parsed correctly.
- The data looks like the expected schema.

---

# 11. Creating Target Tables with LIKE

Instead of manually defining a table:

```sql
CREATE TABLE region (
    ...
);
```

the lab uses:

```sql
CREATE OR REPLACE TABLE region
LIKE snowbearair_db.promo_catalog_sales.region;
```

This means:

> Create a new table based on the structure of the existing table.

## LIKE vs AS SELECT

### LIKE

```sql
CREATE TABLE new_table
LIKE old_table;
```

Creates a table based on another table's structure.

### AS SELECT

```sql
CREATE TABLE new_table AS
SELECT *
FROM old_table;
```

Creates a table from a query result.

---

# 12. COPY INTO

`COPY INTO` is the main Snowflake bulk-loading command.

Basic syntax:

```sql
COPY INTO <target_table>
FROM <stage>
PATTERN = '<regex>'
FILE_FORMAT = (
    FORMAT_NAME = <file_format>
);
```

Example:

```sql
COPY INTO region
FROM @training_db.traininglab.ed_stage/load/lab_files/
PATTERN = 'csv_uncomp_pipe_region_.*'
FILE_FORMAT = (
    FORMAT_NAME = sales_csv_uncomp_pipe_tbl
);
```

Read this as:

> Copy matching files from this stage into the REGION table using this file format.

---

# 13. COPY INTO Anatomy

```text
                 COPY INTO
                     |
                     v
               Target Table
                     ^
                     |
                File Format
                     ^
                     |
                   Stage
                     ^
                     |
                  Pattern
                     |
                selects files
```

The core components are:

```text
Target table
+
Stage
+
File selection
+
File format
```

---

# 14. Load Other Uncompressed Tables

The same file format can be reused for all pipe-delimited files.

Example:

```sql
COPY INTO nation
FROM @training_db.traininglab.ed_stage/load/lab_files/
PATTERN = 'csv_uncomp_pipe_nation_.*'
FILE_FORMAT = (
    FORMAT_NAME = sales_csv_uncomp_pipe_tbl
);
```

Customer:

```sql
COPY INTO customer
FROM @training_db.traininglab.ed_stage/load/lab_files/
PATTERN = 'csv_uncomp_pipe_customer_.*'
FILE_FORMAT = (
    FORMAT_NAME = sales_csv_uncomp_pipe_tbl
);
```

Supplier:

```sql
COPY INTO supplier
FROM @training_db.traininglab.ed_stage/load/lab_files/
PATTERN = 'csv_uncomp_pipe_supplier_.*'
FILE_FORMAT = (
    FORMAT_NAME = sales_csv_uncomp_pipe_tbl
);
```

The important idea:

> Same physical file format, different file-selection pattern.

---

# 15. Part II — Compressed Files

The second section uses:

```text
ORDERS
LINEITEM
PARTSUPP
```

The files are:

```text
CSV
GZIP compressed
Comma-delimited
```

Create:

```sql
CREATE OR REPLACE FILE FORMAT sales_csv_gzip_comma_gz
  TYPE = CSV
  COMPRESSION = GZIP
  FIELD_DELIMITER = ','
  ERROR_ON_COLUMN_COUNT_MISMATCH = FALSE;
```

Notice the difference:

| Part | Type | Compression | Delimiter |
|---|---|---|---|
| I | CSV | NONE | `|` |
| II | CSV | GZIP | `,` |
| III | CSV | GZIP | `,` |

The file format describes the physical source file.

---

# 16. Loading ORDERS

```sql
COPY INTO orders
FROM @training_db.traininglab.ed_stage/load/lab_files/
PATTERN = 'csv_gzip_comma_orders_.*'
FILE_FORMAT = (
    FORMAT_NAME = sales_csv_gzip_comma_gz
);
```

---

# 17. Loading LINEITEM

```sql
COPY INTO lineitem
FROM @training_db.traininglab.ed_stage/load/lab_files/
PATTERN = 'csv_gzip_comma_lineitem_.*'
FILE_FORMAT = (
    FORMAT_NAME = sales_csv_gzip_comma_gz
);
```

---

# 18. Loading PARTSUPP

```sql
COPY INTO partsupp
FROM @training_db.traininglab.ed_stage/load/lab_files/
PATTERN = 'csv_gzip_comma_partsupp_.*'
FILE_FORMAT = (
    FORMAT_NAME = sales_csv_gzip_comma_gz
);
```

The same file format can be reused because the files have the same physical structure.

---

# 19. Compressed Files

You do not need to manually download and unzip GZIP files.

Conceptually:

```text
.gz file
   |
   v
Snowflake
   |
   +-- decompress
   |
   +-- parse
   |
   v
Target table
```

The file format specifies:

```sql
COMPRESSION = GZIP
```

---

# 20. Basic Validation

After loading, check row counts.

```sql
SELECT COUNT(*)
FROM ORDERS;
```

A stronger validation compares table rows with source-file rows:

```sql
SELECT
    'ORDERS' AS NAME,
    COUNT(*) AS TBL_ROW_COUNT,
    (
        SELECT COUNT(1)
        FROM @training_db.traininglab.ed_stage/load/lab_files/
        (
            PATTERN => 'csv_gzip_comma_orders_.*'
        )
    ) AS FILE_ROW_COUNT
FROM ORDERS;
```

Ideally:

```text
TBL_ROW_COUNT = FILE_ROW_COUNT
```

---

# 21. Why Row Count Is Not Enough

Suppose:

```text
Source = 1,000,000 rows
Target = 1,000,000 rows
```

The counts match, but the target could still contain:

- Missing rows
- Duplicate rows
- Incorrect values
- Wrong data types
- Invalid business values

Therefore production validation may include:

```text
Row count
+
NULL checks
+
Duplicate checks
+
Data-type checks
+
Business-rule checks
+
Referential-integrity checks
+
Aggregate checks
```

---

# 22. COPY INTO Output

When `COPY INTO` runs, inspect its result.

The result can provide information such as:

```text
file
status
rows_parsed
rows_loaded
error_limit
errors_seen
first_error
```

Do not ignore this output when debugging a load.

---

# 23. ON_ERROR

A major concept to know is:

```sql
ON_ERROR
```

Example:

```sql
COPY INTO region
FROM @stage
FILE_FORMAT = (
    FORMAT_NAME = my_format
)
ON_ERROR = 'CONTINUE';
```

Common behaviors include:

```text
ABORT_STATEMENT
CONTINUE
SKIP_FILE
SKIP_FILE_n
```

## ABORT_STATEMENT

Default behavior:

```sql
ON_ERROR = 'ABORT_STATEMENT'
```

The statement aborts when errors are encountered.

Use this when data quality should be strict.

## CONTINUE

```sql
ON_ERROR = 'CONTINUE'
```

Allows the load to continue while reporting errors.

Use carefully because bad records may be skipped.

## SKIP_FILE

```sql
ON_ERROR = 'SKIP_FILE'
```

Skips an entire file if it encounters an error.

Conceptually:

```text
File A -> good -> load
File B -> bad  -> skip
File C -> good -> load
```

---

# 24. VALIDATION_MODE

You can validate files before actually loading them.

Example:

```sql
COPY INTO region
FROM @training_db.traininglab.ed_stage/load/lab_files/
FILE_FORMAT = (
    FORMAT_NAME = sales_csv_uncomp_pipe_tbl
)
VALIDATION_MODE = 'RETURN_ERRORS';
```

Think:

```text
Validate
   |
   v
Fix problems
   |
   v
COPY
```

rather than:

```text
COPY
   |
   v
Failure
   |
   v
Debug
```

---

# 25. VALIDATE Function

After a load, Snowflake also provides the `VALIDATE` function for examining errors encountered during loading.

Conceptually:

```text
Before load
    |
    v
VALIDATION_MODE

After load
    |
    v
VALIDATE()
```

Both are useful, but they serve different stages of the workflow.

---

# 26. Load History

Load history is important in production environments.

It helps answer:

```text
Which files were loaded?
When were they loaded?
Into which table?
How many rows were loaded?
Did the load fail?
```

This is essential when building repeatable data pipelines.

---

# 27. Duplicate File Loading

Snowflake tracks previously loaded files for bulk loading operations.

If you run the same `COPY INTO` against unchanged files again, Snowflake generally avoids loading those files again.

However, this does not mean duplicates are impossible.

Duplicates can still occur through:

- Changed source files
- Different file paths
- Manual INSERT statements
- Different ingestion processes
- Transformation logic
- Poor pipeline design

Always design ingestion with idempotency and validation in mind.

---

# 28. Part III — Transform Data During Loading

The most advanced section of the challenge is loading data while transforming it.

Normally:

```text
File
 |
 v
COPY INTO
 |
 v
Table
```

With transformation:

```text
File
 |
 v
SELECT / Transform
 |
 v
COPY INTO
 |
 v
Table
```

Snowflake supports `COPY INTO ... FROM (SELECT ...)` transformations.

---

# 29. SPLIT_PART

The lab uses `SPLIT_PART()`.

Syntax:

```sql
SPLIT_PART(string, delimiter, partNumber)
```

Example:

```sql
SELECT SPLIT_PART(
    'Electronics-Large-Red-Mobile',
    '-',
    1
);
```

Result:

```text
Electronics
```

Second part:

```sql
SELECT SPLIT_PART(
    'Electronics-Large-Red-Mobile',
    '-',
    2
);
```

Result:

```text
Large
```

Third:

```sql
SELECT SPLIT_PART(
    'Electronics-Large-Red-Mobile',
    '-',
    3
);
```

Result:

```text
Red
```

Fourth:

```sql
SELECT SPLIT_PART(
    'Electronics-Large-Red-Mobile',
    '-',
    4
);
```

Result:

```text
Mobile
```

---

# 30. TRIM

The lab uses:

```sql
TRIM(SPLIT_PART($2, '-', 1))
```

Suppose:

```text
Electronics - Large - Red - Mobile
```

contains spaces.

`TRIM()` removes leading and trailing whitespace.

Example:

```sql
SELECT TRIM('  Santosh  ');
```

Result:

```text
Santosh
```

---

# 31. PART Transformation

Suppose source fields are:

```text
$1 = part key
$2 = part name
$3 = size
$4 = retail price
```

We want:

```text
P_PARTKEY
P_SIZE
P_RETAILPRICE
P_NAME
P_NAME_DEPARTMENT
P_NAME_SIZE
P_NAME_COLOR
P_NAME_CATEGORY
```

Mapping:

```text
SOURCE                  TARGET
---------------------------------------------
$1                  ->  P_PARTKEY
$3                  ->  P_SIZE
$4                  ->  P_RETAILPRICE
$2                  ->  P_NAME
split($2,1)         ->  P_NAME_DEPARTMENT
split($2,2)         ->  P_NAME_SIZE
split($2,3)         ->  P_NAME_COLOR
split($2,4)         ->  P_NAME_CATEGORY
```

---

# 32. Transformation SELECT

Example:

```sql
SELECT
    $1,
    $3,
    $4,
    $2,
    TRIM(SPLIT_PART($2, '-', 1)),
    TRIM(SPLIT_PART($2, '-', 2)),
    TRIM(SPLIT_PART($2, '-', 3)),
    TRIM(SPLIT_PART($2, '-', 4))
FROM @training_db.traininglab.ed_stage/load/lab_files/
(
    FILE_FORMAT => 'sales_csv_gzip_comma_gz',
    PATTERN => 'csv_gzip_comma_part_.*'
);
```

The SELECT:

1. Reorders fields.
2. Keeps the original part name.
3. Splits the name into four components.
4. Trims whitespace.

---

# 33. COPY INTO with SELECT

Full pattern:

```sql
COPY INTO PART
FROM (
    SELECT
        $1,
        $3,
        $4,
        $2,
        TRIM(SPLIT_PART($2, '-', 1)),
        TRIM(SPLIT_PART($2, '-', 2)),
        TRIM(SPLIT_PART($2, '-', 3)),
        TRIM(SPLIT_PART($2, '-', 4))
    FROM @training_db.traininglab.ed_stage/load/lab_files/
    (
        FILE_FORMAT => 'sales_csv_gzip_comma_gz',
        PATTERN => 'csv_gzip_comma_part_.*'
    )
);
```

Mental model:

```text
STAGE FILE
    |
    v
$1 $2 $3 $4
    |
    v
SELECT
    |
    +-- reorder
    |
    +-- split
    |
    +-- trim
    |
    v
8 output values
    |
    v
COPY INTO
    |
    v
PART
```

---

# 34. Target Column Order Matters

Suppose the target is:

```sql
CREATE OR REPLACE TABLE PART (
   P_PARTKEY NUMBER(38,0),
   P_SIZE NUMBER(38,0),
   P_RETAILPRICE NUMBER(12,2),
   P_NAME VARCHAR(100),
   P_NAME_DEPARTMENT VARCHAR(25),
   P_NAME_SIZE VARCHAR(10),
   P_NAME_COLOR VARCHAR(25),
   P_NAME_CATEGORY VARCHAR(100)
);
```

There are eight columns.

The SELECT should produce eight values in the corresponding order.

---

# 35. Explicit Target Columns

You can make mappings clearer by explicitly specifying target columns.

Example:

```sql
COPY INTO customers
(
    customer_id,
    customer_name,
    signup_date
)
FROM (
    SELECT
        $1,
        TRIM($2),
        TRY_TO_DATE($3)
    FROM @stage
    (
        FILE_FORMAT => 'my_csv_format'
    )
);
```

This can make complex transformations easier to understand and maintain.

---

# 36. Common Transformations

Transformations during loading can include:

```text
CAST()
TRY_CAST()
TRIM()
UPPER()
LOWER()
SUBSTR()
SPLIT_PART()
REGEXP_REPLACE()
TO_DATE()
TO_TIMESTAMP()
COALESCE()
CASE
```

Example:

```sql
COPY INTO customers
FROM (
    SELECT
        $1::NUMBER,
        UPPER(TRIM($2)),
        TRY_TO_DATE($3)
    FROM @stage
    (
        FILE_FORMAT => 'my_csv_format'
    )
);
```

---

# 37. Metadata Columns

Snowflake provides metadata columns when querying staged files.

Useful examples:

```text
METADATA$FILENAME
METADATA$FILE_ROW_NUMBER
METADATA$FILE_CONTENT_KEY
METADATA$FILE_LAST_MODIFIED
```

Example:

```sql
SELECT
    METADATA$FILENAME AS FILE_NAME,
    METADATA$FILE_ROW_NUMBER AS ROW_NUMBER,
    $1,
    $2,
    $3
FROM @training_db.traininglab.ed_stage/load/lab_files/
(
    FILE_FORMAT => 'sales_csv_uncomp_pipe_tbl',
    PATTERN => 'csv_uncomp_pipe_region.*'
)
LIMIT 20;
```

This helps answer:

> Which file did this row come from?

and:

> Which row in the source file was it?

---

# 38. FILES vs PATTERN

The lab emphasizes `PATTERN`, but `FILES` is another important option.

## PATTERN

```sql
COPY INTO region
FROM @stage
PATTERN = 'region_.*'
FILE_FORMAT = (
    FORMAT_NAME = my_format
);
```

Use when selecting files using a regular expression.

## FILES

```sql
COPY INTO region
FROM @stage
FILES = (
    'region_001.csv',
    'region_002.csv'
)
FILE_FORMAT = (
    FORMAT_NAME = my_format
);
```

Use when you know exactly which files to load.

Remember:

```text
FILES   = explicit filenames
PATTERN = regular-expression selection
```

---

# 39. Header Rows

Suppose a file contains:

```text
id,name,age
1,Santosh,22
2,Rahul,23
```

The first row is a header.

Use:

```sql
SKIP_HEADER = 1
```

Example:

```sql
CREATE FILE FORMAT my_csv
TYPE = CSV
FIELD_DELIMITER = ','
SKIP_HEADER = 1;
```

Without this, the header may be interpreted as a data record.

---

# 40. Quoted CSV Fields

Suppose:

```text
1,"Santosh Singh","Delhi, India"
```

The comma inside:

```text
"Delhi, India"
```

should not be interpreted as a field separator.

Use:

```sql
FIELD_OPTIONALLY_ENCLOSED_BY = '"'
```

Example:

```sql
CREATE FILE FORMAT my_csv
TYPE = CSV
FIELD_DELIMITER = ','
FIELD_OPTIONALLY_ENCLOSED_BY = '"';
```

---

# 41. NULL Handling

If a source contains:

```text
1,Santosh,NULL
```

you can configure:

```sql
NULL_IF = ('NULL', 'null', '');
```

This tells Snowflake which string values should be interpreted as SQL `NULL`.

---

# 42. Data Type Conversion

Suppose:

```text
100,2026-08-13,99.99
```

Target types are:

```text
NUMBER
DATE
NUMBER
```

You can transform explicitly:

```sql
SELECT
    $1::NUMBER,
    TO_DATE($2),
    $3::NUMBER(10,2)
FROM @stage;
```

For safer conversion:

```sql
SELECT
    TRY_TO_NUMBER($1),
    TRY_TO_DATE($2)
FROM @stage;
```

`TRY_` conversion functions return `NULL` when conversion fails instead of raising a conversion error.

---

# 43. Loading Workflow — Best Practice

A good workflow is:

```text
1. LIST
   |
2. Inspect files
   |
3. Define FILE FORMAT
   |
4. Query staged files
   |
5. Validate parsing
   |
6. COPY INTO
   |
7. Inspect COPY result
   |
8. Validate row counts
   |
9. Validate data quality
   |
10. Check load history
```

This is much better than blindly running `COPY INTO`.

---

# 44. Challenge Lab Relationship Validation

The lab joins the loaded tables to verify that their relationships work.

Conceptually:

```text
REGION
   |
   | R_REGIONKEY
   v
NATION
   |
   | N_NATIONKEY
   v
CUSTOMER
   |
   | C_CUSTKEY
   v
ORDERS
   |
   | O_ORDERKEY
   v
LINEITEM
   |
   +--------------+
   |              |
   v              v
PART          SUPPLIER
   |              |
   +------ PARTSUPP
```

Example:

```sql
SELECT
    R.R_NAME,
    N.N_NAME,
    C.C_FIRSTNAME,
    C.C_LASTNAME,
    O.O_ORDERKEY,
    PS.PS_AVAILQTY,
    S.S_NAME
FROM REGION R
INNER JOIN NATION N
    ON R.R_REGIONKEY = N.N_REGIONKEY
INNER JOIN CUSTOMER C
    ON N.N_NATIONKEY = C.C_NATIONKEY
INNER JOIN ORDERS O
    ON C.C_CUSTKEY = O.O_CUSTKEY
INNER JOIN LINEITEM L
    ON O.O_ORDERKEY = L.L_ORDERKEY
INNER JOIN PARTSUPP PS
    ON L.L_PARTKEY = PS.PS_PARTKEY
INNER JOIN SUPPLIER S
    ON L.L_SUPPKEY = S.S_SUPPKEY
LIMIT 100;
```

This checks whether the loaded data preserves expected relationships.

---

# 45. Lab Row-Count Validation

The challenge gives expected counts such as:

```text
ORDERS       -> 1,500,000
LINEITEM     -> 6,001,215
PARTSUPP     ->   800,000
```

The important lesson is not to memorize the numbers.

The important lesson is:

```text
Expected source records
        =
Loaded records
```

Use these numbers as validation targets for this particular lab environment.

---

# 46. Important Lab Inconsistency

One validation query in the lab appears to use:

```sql
PATTERN => 'csv_uncomp_pipe_part_.*'
```

while Part III loads PART from:

```sql
PATTERN => 'csv_gzip_comma_part_.*'
```

The validation should logically use:

```sql
SELECT
    'PART' AS NAME,
    COUNT(*) AS TBL_ROW_COUNT,
    (
        SELECT COUNT(1)
        FROM @training_db.traininglab.ed_stage/load/lab_files/
        (
            PATTERN => 'csv_gzip_comma_part_.*'
        )
    ) AS FILE_ROW_COUNT
FROM PART;
```

This is an important example of why you should understand the SQL rather than blindly copy lab answers.

---

# 47. Another Lab Inconsistency — PART Columns

One version of the challenge describes PART with seven columns, while the final transformation requires eight values.

The transformation clearly contains:

```text
$1
$3
$4
$2
split($2,1)
split($2,2)
split($2,3)
split($2,4)
```

That is eight output fields.

The logical target structure is therefore:

```text
P_PARTKEY
P_SIZE
P_RETAILPRICE
P_NAME
P_NAME_DEPARTMENT
P_NAME_SIZE
P_NAME_COLOR
P_NAME_CATEGORY
```

Use the eight-column version when reproducing the transformation.

---

# 48. Production Mental Model

Think of the complete process as:

```text
                 SOURCE
                   |
                   v
            CSV / GZIP / etc.
                   |
                   v
                 STAGE
                   |
          +--------+--------+
          |                 |
        LIST              SELECT
          |                 |
          |            Inspect data
          |                 |
          +--------+--------+
                   |
                   v
              FILE FORMAT
                   |
       +-----------+-----------+
       |           |           |
   delimiter   compression   options
       |           |           |
       +-----------+-----------+
                   |
                   v
                PATTERN
                   |
              choose files
                   |
                   v
              COPY INTO
                   |
          +--------+--------+
          |                 |
       direct          transformation
          |                 |
          |              SELECT
          |                 |
          +--------+--------+
                   |
                   v
                TABLE
                   |
                   v
              VALIDATION
                   |
       +-----------+-----------+
       |           |           |
    row count   data quality  joins
```

---

# 49. Practice Lab 1 — Pipe Format

Create:

```text
my_pipe_format
```

Requirements:

```text
TYPE = CSV
COMPRESSION = NONE
FIELD_DELIMITER = |
```

Then inspect REGION files using the format.

Try to write the SQL without looking at the solution.

---

# 50. Practice Lab 2 — Load REGION

Create:

```text
REGION_PRACTICE
```

using the existing REGION table structure.

Then load:

```text
csv_uncomp_pipe_region_*
```

into it.

Finally check:

```sql
SELECT COUNT(*)
FROM REGION_PRACTICE;
```

---

# 51. Practice Lab 3 — Validate Source vs Target

Write a query returning:

```text
NAME
TABLE_COUNT
FILE_COUNT
```

Expected structure:

```text
REGION_PRACTICE | X | X
```

Your goal is:

```text
TABLE_COUNT = FILE_COUNT
```

---

# 52. Practice Lab 4 — GZIP

Create:

```text
my_gzip_csv
```

with:

```text
TYPE = CSV
COMPRESSION = GZIP
FIELD_DELIMITER = ,
```

Then load ORDERS into:

```text
ORDERS_PRACTICE
```

---

# 53. Practice Lab 5 — Transformation

Create:

```text
PART_PRACTICE
```

with:

```text
P_PARTKEY
P_SIZE
P_RETAILPRICE
P_NAME
P_NAME_DEPARTMENT
P_NAME_SIZE
P_NAME_COLOR
P_NAME_CATEGORY
```

Then split the part-name field using:

```sql
SPLIT_PART()
```

and trim each extracted value using:

```sql
TRIM()
```

Load the transformed data using:

```sql
COPY INTO ... FROM (
    SELECT ...
    FROM @stage (...)
);
```

---

# 54. Practice Lab 6 — Error Handling

Intentionally create a target table whose column count does not match the source.

Try a load.

Then experiment with:

```sql
ON_ERROR = 'ABORT_STATEMENT'
```

and:

```sql
ON_ERROR = 'CONTINUE'
```

Then use:

```sql
VALIDATION_MODE = 'RETURN_ERRORS'
```

to inspect problems.

---

# 55. Practice Lab 7 — Metadata

Run:

```sql
SELECT
    METADATA$FILENAME,
    METADATA$FILE_ROW_NUMBER,
    $1,
    $2
FROM @training_db.traininglab.ed_stage/load/lab_files/
(
    FILE_FORMAT => 'sales_csv_gzip_comma_gz',
    PATTERN => 'csv_gzip_comma_orders_.*'
)
LIMIT 20;
```

Understand:

```text
Which file?
Which row?
What data?
```

---

# 56. Practice Lab 8 — Header Handling

Create a file format for:

```text
id,name,age
1,Santosh,22
2,Rahul,23
```

Use:

```sql
SKIP_HEADER = 1
```

Verify that only the two data rows are parsed.

---

# 57. Practice Lab 9 — Quoted Values

Create a file format that correctly handles:

```text
1,"Santosh Singh","Delhi, India"
```

Use:

```sql
FIELD_OPTIONALLY_ENCLOSED_BY = '"'
```

Verify that:

```text
$3 = Delhi, India
```

rather than splitting it into two fields.

---

# 58. Practice Lab 10 — Safe Conversion

Test:

```sql
SELECT
    TRY_TO_NUMBER($1),
    TRY_TO_DATE($2)
FROM @stage (...);
```

Try valid and invalid source values.

Observe how `TRY_` conversion functions behave.

---

# 59. Interview Questions

## Q1. What is a stage?

A location where data files are stored before loading them into Snowflake.

## Q2. What is a file format?

A configuration describing how Snowflake should parse source files.

## Q3. What does `$1` mean?

The first field of a staged file.

## Q4. Why use PATTERN?

To select files based on a regular expression.

## Q5. What does GZIP mean in a file format?

The source files are GZIP compressed.

## Q6. What does FIELD_DELIMITER do?

Defines the character separating fields.

## Q7. What is COPY INTO?

Snowflake's bulk-loading command for loading staged data into a table.

## Q8. Can COPY INTO transform data?

Yes. `COPY INTO ... FROM (SELECT ...)` supports transformations during loading.

## Q9. What does SPLIT_PART do?

Extracts a specific component of a delimited string.

## Q10. Why query staged files before loading?

To verify parsing, file structure, selected files, and data quality.

## Q11. FILES vs PATTERN?

`FILES` explicitly names files; `PATTERN` selects files using a regular expression.

## Q12. How can you validate data before loading?

Use `VALIDATION_MODE`, such as:

```sql
VALIDATION_MODE = 'RETURN_ERRORS'
```

## Q13. How can you inspect source-file information?

Use staged-file metadata such as:

```text
METADATA$FILENAME
METADATA$FILE_ROW_NUMBER
```

---

# 60. What to Memorize

Do not memorize the entire challenge lab.

Memorize this workflow:

```text
STAGE
  |
LIST
  |
FILE FORMAT
  |
SELECT $1,$2,...
  |
PATTERN
  |
COPY INTO
  |
VALIDATE
```

Core file-format syntax:

```sql
CREATE FILE FORMAT format_name
TYPE = CSV
COMPRESSION = ...
FIELD_DELIMITER = ...;
```

Inspect staged data:

```sql
SELECT $1, $2, $3
FROM @stage
(
    FILE_FORMAT => 'format_name',
    PATTERN => 'regex'
);
```

Load directly:

```sql
COPY INTO table_name
FROM @stage
PATTERN = 'regex'
FILE_FORMAT = (
    FORMAT_NAME = format_name
);
```

Load with transformation:

```sql
COPY INTO table_name
FROM (
    SELECT
        $1,
        TRIM($2),
        SPLIT_PART($3, '-', 1)
    FROM @stage
    (
        FILE_FORMAT => 'format_name',
        PATTERN => 'regex'
    )
);
```

---

# 61. The Three Most Important Concepts

## Stage

```text
WHERE THE FILE IS
```

## File Format

```text
HOW THE FILE IS STRUCTURED
```

## COPY INTO

```text
HOW THE FILE GETS INTO THE TABLE
```

Therefore:

```text
@stage
   +
file format
   +
COPY INTO
   |
   v
table
```

---

# 62. Quick Revision Sheet

```text
STAGE
  -> Stores source files

LIST
  -> Lists staged files

FILE FORMAT
  -> Describes file structure

$1, $2, $3
  -> Source fields

PATTERN
  -> Regex-based file selection

FILES
  -> Explicit file selection

COPY INTO
  -> Bulk loading

ON_ERROR
  -> Controls load-error behavior

VALIDATION_MODE
  -> Validates before loading

VALIDATE()
  -> Helps inspect load errors

METADATA$FILENAME
  -> Source filename

METADATA$FILE_ROW_NUMBER
  -> Source row number

SPLIT_PART()
  -> Splits a string

TRIM()
  -> Removes surrounding spaces

COPY INTO FROM (SELECT ...)
  -> Transform during load
```

---

# 63. Official Snowflake Documentation

Keep these official references with this module:

- Snowflake data loading
- Querying staged files
- `COPY INTO <table>`
- Transforming data during loading
- Preparing data for loading
- Loading from internal stages

For your GitHub Snowflake documentation project, the most important concepts to connect to the next modules are:

```text
Stages
   |
File Formats
   |
COPY INTO
   |
Data Validation
   |
Load History
   |
Transformations
```

This module forms the foundation for more advanced Snowflake ingestion topics such as external stages, storage integrations, Snowpipe, Snowpipe Streaming, and continuous data pipelines.
