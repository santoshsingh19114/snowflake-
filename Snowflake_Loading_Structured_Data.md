# Snowflake --- Loading Structured Data

> Complete learning notes for the **Loading Structured Data** module,
> expanded beyond the lab to cover important exam, interview, and
> hands-on gaps.

------------------------------------------------------------------------

## 1. Learning Objectives

By the end of this chapter, you should understand:

-   Structured data and common file formats
-   How data moves into Snowflake
-   Internal and external stages
-   User, table, and named stages
-   File formats
-   CSV / pipe-delimited files
-   Compression and GZIP
-   `LIST`
-   `DESCRIBE STAGE`
-   `$1`, `$2`, `$3` staged-file references
-   `COPY INTO`
-   `FILES`
-   `PATTERN`
-   `VALIDATION_MODE`
-   `ON_ERROR`
-   `ABORT_STATEMENT`
-   `CONTINUE`
-   `SKIP_FILE`
-   `SKIP_FILE_n`
-   Transformations during loading
-   Data type conversion and casting
-   Header, quote, and NULL handling
-   `PUT` and `GET`
-   `FORCE = TRUE`
-   Load validation and load history
-   Common loading mistakes
-   Hands-on practice
-   Accenture-style MCQs
-   Interview questions

------------------------------------------------------------------------

# 2. Big Picture: How Data Gets Into Snowflake

The basic loading architecture is:

``` text
Source File
    |
    v
  Stage
    |
    v
File Format
    |
    v
COPY INTO
    |
    v
Snowflake Table
```

A more complete production workflow is:

``` text
Source
  |
  +------------------+
  |                  |
Local Files      Cloud Storage
  |                  |
 PUT                External Stage
  |                  |
  +--------+---------+
           |
           v
         Stage
           |
           v
      File Format
           |
           v
        Preview
           |
           v
       Validate
           |
           v
       COPY INTO
           |
     +-----+------+
     |            |
 Success        Errors
     |            |
     v            v
  Table       ON_ERROR
                  |
       +----------+----------+
       |          |          |
     ABORT     CONTINUE   SKIP_FILE
```

The key idea is:

> **Stage = WHERE the file is**\
> **File format = HOW Snowflake should read it**\
> **COPY INTO = LOAD the data**

------------------------------------------------------------------------

# 3. What Is Structured Data?

Structured data is data organized according to a predefined structure or
schema.

Example:

``` text
101|Santosh|India|21
102|Rahul|India|22
103|Aman|India|23
```

The corresponding table could be:

``` sql
CREATE TABLE students (
    id INTEGER,
    name VARCHAR,
    country VARCHAR,
    age INTEGER
);
```

Each field has a known meaning and data type.

## Common Structured Data Formats

  Format           Example
  ---------------- -----------------------------------------
  CSV              `1,Santosh,India`
  TSV              `1<TAB>Santosh<TAB>India`
  Pipe-delimited   `1|Santosh|India`
  Fixed-width      Fields occupy fixed character positions
  `.tbl`           Commonly delimiter-separated
  `.csv.gz`        GZIP-compressed CSV
  `.tbl.gz`        GZIP-compressed TBL

------------------------------------------------------------------------

# 4. Snowflake Stages

A **stage** is a location or access point for files before they are
loaded into Snowflake tables.

Think of a stage as a:

> **Waiting room for files**

The normal flow is:

``` text
File
  |
  v
Stage
  |
  v
COPY INTO
  |
  v
Table
```

------------------------------------------------------------------------

# 5. Types of Stages

Snowflake supports:

1.  Internal stages
2.  External stages

Snowflake also provides special internal stages associated with users
and tables.

``` text
STAGES
│
├── Internal
│   ├── User stage
│   ├── Table stage
│   └── Named internal stage
│
└── External
    └── Cloud storage
        ├── Amazon S3
        ├── Microsoft Azure
        └── Google Cloud Storage
```

------------------------------------------------------------------------

# 6. Internal vs External Stages

## Internal Stage

Files are stored in Snowflake-managed storage.

``` text
Your File
    |
    v
Snowflake Internal Stage
    |
    v
COPY INTO
    |
    v
Table
```

## External Stage

Files remain in external cloud storage.

``` text
Your File
    |
    v
S3 / Azure / GCS
    |
    v
Snowflake External Stage
    |
    v
COPY INTO
    |
    v
Table
```

The lab uses an external stage such as:

``` sql
@training_db.traininglab.ed_stage
```

------------------------------------------------------------------------

# 7. Understanding `@`

The `@` symbol indicates a Snowflake stage reference.

Examples:

``` sql
@my_stage
```

``` sql
@database.schema.my_stage
```

Example from the lab:

``` sql
@training_db.traininglab.ed_stage
```

Breakdown:

``` text
training_db
     |
  Database

traininglab
     |
   Schema

ed_stage
     |
   Stage
```

------------------------------------------------------------------------

# 8. Special Stage Syntax

Three important syntaxes:

  Syntax           Meaning
  ---------------- -------------
  `@~`             User stage
  `@%table_name`   Table stage
  `@named_stage`   Named stage

Examples:

``` sql
LIST @~;
```

``` sql
LIST @%students;
```

``` sql
LIST @my_stage;
```

### Memorize

``` text
@~
    -> User stage

@%table
    -> Table stage

@stage
    -> Named stage
```

------------------------------------------------------------------------

# 9. Lab Setup

A typical lab starts by establishing Snowflake context.

``` sql
USE ROLE fund_role;

CREATE WAREHOUSE IF NOT EXISTS ZEBRA_fund_wh
INITIALLY_SUSPENDED=TRUE;

USE WAREHOUSE ZEBRA_fund_wh;

CREATE DATABASE IF NOT EXISTS ZEBRA_fund_db;

CREATE SCHEMA IF NOT EXISTS ZEBRA_fund_db.my_schema;

USE SCHEMA ZEBRA_fund_db.my_schema;
```

Think of the hierarchy as:

``` text
Role
  |
  v
Warehouse
  |
  v
Database
  |
  v
Schema
  |
  v
Table
```

------------------------------------------------------------------------

# 10. Creating a Target Table

Example:

``` sql
CREATE OR REPLACE TABLE REGION (
    R_REGIONKEY NUMBER(38,0) NOT NULL,
    R_NAME      VARCHAR(25) NOT NULL,
    R_COMMENT   VARCHAR(152)
);
```

Suppose the source file contains:

``` text
0|AFRICA|lar deposits...
1|AMERICA|hses...
2|ASIA|ges...
3|EUROPE|ly regular...
4|MIDDLE EAST|u...
```

The positional mapping is:

``` text
$1 -> R_REGIONKEY
$2 -> R_NAME
$3 -> R_COMMENT
```

------------------------------------------------------------------------

# 11. `LIST`

Use `LIST` to see files available at a stage.

Example:

``` sql
LIST @training_db.traininglab.ed_stage/load/lab_files/region.tbl;
```

You can also list a directory:

``` sql
LIST @training_db.traininglab.ed_stage/load/lab_files/;
```

`LIST` is useful for checking:

-   Whether a file exists
-   File name
-   File size
-   File location/path
-   Available staged files

Typical workflow:

``` sql
LIST @my_stage;
```

------------------------------------------------------------------------

# 12. `DESCRIBE STAGE`

Use:

``` sql
DESCRIBE STAGE training_db.traininglab.ed_stage;
```

This lets you inspect stage properties.

One important property is the stage's default file format.

For example, a stage may have:

``` text
FIELD_DELIMITER = ,
```

while your actual file uses:

``` text
|
```

That mismatch must be handled with an appropriate file format.

------------------------------------------------------------------------

# 13. File Formats

A **file format** tells Snowflake how to interpret a source file.

Think:

> "How should Snowflake read this file?"

For:

``` text
1|Santosh|India
```

Snowflake needs to know that:

``` text
|
```

is the delimiter.

A file format can define things such as:

-   File type
-   Field delimiter
-   Record delimiter
-   Compression
-   Header handling
-   Quote handling
-   Escape rules
-   NULL representation
-   Column-count behavior

------------------------------------------------------------------------

# 14. Creating a File Format

The lab uses:

``` sql
CREATE OR REPLACE FILE FORMAT MYPIPEFORMAT
  TYPE = CSV
  COMPRESSION = NONE
  FIELD_DELIMITER = '|'
  ERROR_ON_COLUMN_COUNT_MISMATCH = FALSE;
```

Let's understand each option.

------------------------------------------------------------------------

# 15. `TYPE = CSV`

``` sql
TYPE = CSV
```

This tells Snowflake to use CSV-style parsing rules.

CSV does not necessarily mean the delimiter must be a comma.

For example:

``` text
FIELD_DELIMITER = ','
```

or:

``` text
FIELD_DELIMITER = '|'
```

or:

``` text
FIELD_DELIMITER = ';'
```

are all possible configurations.

------------------------------------------------------------------------

# 16. `FIELD_DELIMITER`

``` sql
FIELD_DELIMITER = '|'
```

This tells Snowflake that fields are separated by `|`.

For:

``` text
1|Santosh|India|21
```

Snowflake sees:

``` text
$1 = 1
$2 = Santosh
$3 = India
$4 = 21
```

If you use the wrong delimiter, Snowflake may treat the entire row as
one field.

------------------------------------------------------------------------

# 17. `$1`, `$2`, `$3`, ...

When querying a staged file, Snowflake allows positional references:

``` sql
$1
$2
$3
...
```

Example:

``` sql
SELECT
    $1,
    $2,
    $3
FROM @my_stage/file.csv;
```

For:

``` text
10|Santosh|India
```

the mapping is:

``` text
$1 -> 10
$2 -> Santosh
$3 -> India
```

This is extremely useful for previewing and transforming staged data.

------------------------------------------------------------------------

# 18. Previewing Staged Data

Before loading, inspect the file:

``` sql
SELECT
    $1,
    $2,
    $3
FROM @my_stage
(FILE_FORMAT => 'MYPIPEFORMAT');
```

Or:

``` sql
SELECT
    $1,
    $2,
    $3
FROM @my_stage/path/
(FILE_FORMAT => 'MYPIPEFORMAT');
```

This lets you answer:

-   Are fields split correctly?
-   Is the delimiter correct?
-   Are values in the expected columns?
-   Is the file format correct?

------------------------------------------------------------------------

# 19. Why the Wrong Delimiter Causes Problems

Suppose the file is:

``` text
1|AFRICA|comment
```

but the format says:

``` sql
FIELD_DELIMITER = ','
```

Snowflake does not find commas.

So it can interpret the row as one field:

``` text
$1 = 1|AFRICA|comment
```

instead of:

``` text
$1 = 1
$2 = AFRICA
$3 = comment
```

Therefore:

> **Always make sure the file format matches the actual file.**

------------------------------------------------------------------------

# 20. Compression

The lab includes:

``` text
region.tbl
region.tbl.gz
```

The second file is GZIP-compressed.

Snowflake supports multiple compression types.

Common examples include:

-   GZIP
-   Brotli
-   Zstandard
-   Deflate
-   Raw Deflate
-   Snappy

------------------------------------------------------------------------

# 21. Explicit GZIP File Format

Example:

``` sql
CREATE OR REPLACE FILE FORMAT MYGZIPPIPEFORMAT
TYPE = CSV
COMPRESSION = GZIP
FIELD_DELIMITER = '|'
ERROR_ON_COLUMN_COUNT_MISMATCH = FALSE;
```

This tells Snowflake:

``` text
Type        -> CSV
Delimiter   -> |
Compression -> GZIP
```

Then:

``` sql
COPY INTO REGION
FROM @training_db.traininglab.ed_stage/load/lab_files/
FILES = ('region.tbl.gz')
FILE_FORMAT = (
    FORMAT_NAME = MYGZIPPIPEFORMAT
);
```

------------------------------------------------------------------------

# 22. What Happens if Compression Is Wrong?

If you use:

``` sql
COMPRESSION = GZIP
```

but provide an uncompressed file:

``` text
region.tbl
```

Snowflake expects GZIP-compressed content.

This can cause a load error.

The important lesson:

``` text
File compression
       must match
File format expectations
```

------------------------------------------------------------------------

# 23. Compression Auto-Detection

You can omit an explicit compression setting when appropriate.

Example:

``` sql
CREATE OR REPLACE FILE FORMAT AUTO_FORMAT
TYPE = CSV
FIELD_DELIMITER = '|'
ERROR_ON_COLUMN_COUNT_MISMATCH = FALSE;
```

This can allow Snowflake to detect supported compression automatically.

This can work with both:

``` text
region.tbl
```

and:

``` text
region.tbl.gz
```

when the file content and other parsing options are correct.

------------------------------------------------------------------------

# 24. `COPY INTO`

`COPY INTO` is the core Snowflake command for loading staged files into
a table.

Basic syntax:

``` sql
COPY INTO <table>
FROM <stage>;
```

Example:

``` sql
COPY INTO REGION
FROM @my_stage;
```

With a file format:

``` sql
COPY INTO REGION
FROM @my_stage
FILE_FORMAT = (
    FORMAT_NAME = MYPIPEFORMAT
);
```

------------------------------------------------------------------------

# 25. Understanding a `COPY INTO` Statement

Example:

``` sql
COPY INTO REGION
FROM @training_db.traininglab.ed_stage/load/lab_files/
FILES = ('region.tbl')
FILE_FORMAT = (
    FORMAT_NAME = MYPIPEFORMAT
);
```

Breakdown:

``` text
COPY INTO REGION
       |
       +-- Target table

FROM @stage/path
       |
       +-- Source location

FILES = ('region.tbl')
       |
       +-- Exact file to load

FILE_FORMAT = (...)
       |
       +-- How to parse the file
```

------------------------------------------------------------------------

# 26. `FILES`

Use `FILES` when you want to specify exact files.

Example:

``` sql
COPY INTO REGION
FROM @my_stage
FILES = ('region.tbl')
FILE_FORMAT = (
    FORMAT_NAME = MYPIPEFORMAT
);
```

Multiple files:

``` sql
COPY INTO REGION
FROM @my_stage
FILES = (
    'region1.tbl',
    'region2.tbl',
    'region3.tbl'
)
FILE_FORMAT = (
    FORMAT_NAME = MYPIPEFORMAT
);
```

Think:

> `FILES` = exact file selection.

------------------------------------------------------------------------

# 27. `PATTERN`

`PATTERN` allows regular-expression-based file selection.

Example:

``` sql
COPY INTO REGION
FROM @my_stage
PATTERN = '.*[.]csv'
FILE_FORMAT = (
    FORMAT_NAME = MYPIPEFORMAT
);
```

This can select files such as:

``` text
customers.csv
orders.csv
products.csv
```

but not:

``` text
customers.json
```

Think:

> `PATTERN` = regex-based file selection.

------------------------------------------------------------------------

# 28. `FILES` vs `PATTERN`

  Feature          `FILES`          `PATTERN`
  ---------------- ---------------- ------------------------
  Exact files      Yes              No
  Regex matching   No               Yes
  Best for         Specific files   Dynamic file selection
  Example          `'region.tbl'`   `'.*[.]csv'`

------------------------------------------------------------------------

# 29. `VALIDATION_MODE`

A professional loading workflow should validate data before loading when
appropriate.

`VALIDATION_MODE` allows Snowflake to inspect staged data without
performing the normal load.

Example:

``` sql
COPY INTO NATION
FROM @my_stage
FILES = ('nation.tbl')
FILE_FORMAT = (
    FORMAT_NAME = MYPIPEFORMAT
)
VALIDATION_MODE = RETURN_ALL_ERRORS;
```

The important idea is:

``` text
VALIDATION_MODE
       |
       v
Check the data
       |
       v
Do not perform the normal load
```

------------------------------------------------------------------------

# 30. `RETURN_ALL_ERRORS`

Example:

``` sql
COPY INTO NATION
FROM @my_stage
VALIDATION_MODE = RETURN_ALL_ERRORS;
```

This asks Snowflake to report all detected errors from the validation.

Useful for:

-   Bad data
-   Datatype conversion errors
-   File parsing issues
-   Troubleshooting before loading

------------------------------------------------------------------------

# 31. `RETURN_ERRORS`

Example:

``` sql
COPY INTO NATION
FROM @my_stage
VALIDATION_MODE = RETURN_ERRORS;
```

This is another validation mode for returning errors.

For exams, remember:

> Validation modes are for checking staged data rather than performing
> the normal table load.

------------------------------------------------------------------------

# 32. `RETURN_10_ROWS`

The lab also demonstrates:

``` sql
VALIDATION_MODE = RETURN_10_ROWS;
```

This is useful when you want a limited sample during validation.

------------------------------------------------------------------------

# 33. Column Position Matters

Suppose the file contains:

``` text
1|ALGERIA|0|Good
```

and the table is:

``` sql
CREATE OR REPLACE TABLE nation (
    NATION_KEY INTEGER,
    REGION_KEY INTEGER,
    NATION VARCHAR,
    COMMENTS VARCHAR
);
```

Snowflake maps positionally:

``` text
$1 -> NATION_KEY
$2 -> REGION_KEY
$3 -> NATION
$4 -> COMMENTS
```

But:

``` text
$2 = ALGERIA
```

and:

``` text
REGION_KEY = INTEGER
```

So Snowflake attempts:

``` text
ALGERIA -> INTEGER
```

which causes a conversion error.

------------------------------------------------------------------------

# 34. File Order vs Table Order

Suppose the file is:

``` text
1|India|5
```

and the table is:

``` text
id
country
region_id
```

Mapping:

``` text
$1 -> id
$2 -> country
$3 -> region_id
```

Correct.

But if the table is:

``` text
id
region_id
country
```

then:

``` text
$1 -> id
$2 -> region_id
$3 -> country
```

Now:

``` text
India -> INTEGER
```

causes an error.

Therefore:

> **When loading without an explicit transformation/mapping, column
> position is critical.**

------------------------------------------------------------------------

# 35. Data Transformation During Loading

Snowflake can transform staged data during `COPY INTO`.

Basic load:

``` sql
COPY INTO my_table
FROM @my_stage;
```

Transformation load:

``` sql
COPY INTO my_table
FROM (
    SELECT
        $1,
        UPPER($2),
        $3
    FROM @my_stage
);
```

This allows operations such as:

-   `UPPER()`
-   `LOWER()`
-   `TRIM()`
-   `CAST`
-   `CASE`
-   Date conversion
-   String manipulation
-   Conditional transformations

------------------------------------------------------------------------

# 36. Example Transformation

Suppose the file contains:

``` text
1|santosh
2|rahul
3|aman
```

Target:

``` sql
CREATE TABLE users (
    id INTEGER,
    name VARCHAR
);
```

Load:

``` sql
COPY INTO users
FROM (
    SELECT
        $1::INTEGER,
        UPPER($2)
    FROM @my_stage
)
FILE_FORMAT = (
    FORMAT_NAME = my_format
);
```

Result:

``` text
1 | SANTOSH
2 | RAHUL
3 | AMAN
```

------------------------------------------------------------------------

# 37. `CASE` During Loading

Example:

``` sql
COPY INTO nation
FROM (
    SELECT
        $1,
        $2,
        CASE
            WHEN $3 = '1' THEN 'AMERICA'
            ELSE $3
        END,
        $4
    FROM @my_stage
);
```

This demonstrates an important concept:

> A transformation can itself introduce datatype or logic errors.

Always make sure the transformed value matches the destination column's
data type.

------------------------------------------------------------------------

# 38. Casting During Load

If a field is textual but the target column is numeric:

``` sql
$1::INTEGER
```

or:

``` sql
CAST($1 AS INTEGER)
```

Example:

``` sql
COPY INTO students
FROM (
    SELECT
        $1::INTEGER,
        $2::VARCHAR,
        $3::INTEGER
    FROM @my_stage
)
FILE_FORMAT = (
    FORMAT_NAME = my_format
);
```

------------------------------------------------------------------------

# 39. Error Handling with `ON_ERROR`

`ON_ERROR` controls what Snowflake should do when errors are encountered
during loading.

Important options:

``` text
ABORT_STATEMENT
CONTINUE
SKIP_FILE
SKIP_FILE_n
```

------------------------------------------------------------------------

# 40. `ON_ERROR = ABORT_STATEMENT`

This is the default behavior.

``` sql
COPY INTO nation
FROM @my_stage
ON_ERROR = ABORT_STATEMENT;
```

Meaning:

> Abort the load when an error occurs.

Conceptually:

``` text
25 rows
5 bad rows
20 good rows

ABORT_STATEMENT
       |
       v
Load fails
```

------------------------------------------------------------------------

# 41. `ON_ERROR = CONTINUE`

Example:

``` sql
COPY INTO nation
FROM @my_stage
ON_ERROR = CONTINUE;
```

Meaning:

> Continue loading valid rows and reject problematic rows.

Suppose:

``` text
25 total rows
5 errors
20 valid rows
```

Conceptually:

``` text
25 rows
 |
 +-- 20 good -> loaded
 |
 +-- 5 bad   -> rejected
```

This results in a partial load.

------------------------------------------------------------------------

# 42. `ON_ERROR = SKIP_FILE`

This operates at the file level.

Conceptually:

``` text
File
 |
 v
Errors?
 |
 +-- No --> Load
 |
 +-- Yes -> Skip file
```

Unlike `CONTINUE`, it can prevent the entire file from being loaded.

------------------------------------------------------------------------

# 43. `SKIP_FILE_n`

Example:

``` sql
ON_ERROR = SKIP_FILE_4
```

This means the file is skipped when the error threshold is reached.

If:

``` text
Errors = 5
Threshold = 4
```

then:

``` text
5 >= 4
```

so the file is skipped.

------------------------------------------------------------------------

# 44. `SKIP_FILE_6`

If:

``` sql
ON_ERROR = SKIP_FILE_6
```

and:

``` text
Errors = 5
```

then:

``` text
5 < 6
```

so the threshold is not reached.

Valid rows can therefore be loaded according to the load behavior.

------------------------------------------------------------------------

# 45. Error Handling Comparison

Suppose:

``` text
Total rows = 100
Bad rows = 10
Good rows = 90
```

### `ABORT_STATEMENT`

``` text
Loaded = 0
```

### `CONTINUE`

``` text
Loaded = 90
Rejected = 10
```

### `SKIP_FILE_5`

``` text
10 >= 5
File skipped
Loaded = 0
```

### `SKIP_FILE_20`

``` text
10 < 20
Threshold not reached
Valid rows can load
```

Memorize:

``` text
ABORT_STATEMENT
-> Fail the load

CONTINUE
-> Continue with valid rows

SKIP_FILE_n
-> File-level error threshold
```

------------------------------------------------------------------------

# 46. `ERROR_ON_COLUMN_COUNT_MISMATCH`

Example:

``` sql
ERROR_ON_COLUMN_COUNT_MISMATCH = FALSE
```

This option controls how Snowflake handles rows where the number of
fields does not match the expected number of columns.

Example table:

``` text
3 columns
```

File row:

``` text
1|Santosh
```

has two fields.

Another row:

``` text
1|Santosh|India|21
```

has four fields.

Do not confuse this with a datatype error.

For example:

``` text
ALGERIA -> INTEGER
```

is a conversion/type problem.

------------------------------------------------------------------------

# 47. Header Handling

Real CSV files often contain headers:

``` text
ID,NAME,COUNTRY
1,Santosh,India
2,Rahul,India
```

Use:

``` sql
SKIP_HEADER = 1
```

Example:

``` sql
CREATE OR REPLACE FILE FORMAT my_csv
TYPE = CSV
FIELD_DELIMITER = ','
SKIP_HEADER = 1;
```

Snowflake skips:

``` text
ID,NAME,COUNTRY
```

and loads the data rows.

------------------------------------------------------------------------

# 48. Quoted CSV Fields

Real CSV data can contain commas inside values:

``` text
1,"Santosh, Kumar","India"
```

To correctly handle optionally quoted fields:

``` sql
CREATE OR REPLACE FILE FORMAT my_csv
TYPE = CSV
FIELD_DELIMITER = ','
FIELD_OPTIONALLY_ENCLOSED_BY = '"';
```

This is important when working with real-world CSV files.

------------------------------------------------------------------------

# 49. NULL Handling

A file might contain:

``` text
1|Santosh|NULL
```

or:

``` text
1|Santosh|
```

You can configure NULL representations.

Example:

``` sql
CREATE OR REPLACE FILE FORMAT my_format
TYPE = CSV
FIELD_DELIMITER = '|'
NULL_IF = ('NULL', 'null', '');
```

This tells Snowflake to treat the specified values as SQL `NULL`.

------------------------------------------------------------------------

# 50. Named vs Inline File Formats

## Named File Format

``` sql
CREATE OR REPLACE FILE FORMAT my_csv
TYPE = CSV
FIELD_DELIMITER = '|';
```

Use:

``` sql
COPY INTO my_table
FROM @my_stage
FILE_FORMAT = (
    FORMAT_NAME = my_csv
);
```

Good for:

-   Reuse
-   Standardization
-   Multiple pipelines
-   Consistency

## Inline File Format

``` sql
COPY INTO my_table
FROM @my_stage
FILE_FORMAT = (
    TYPE = CSV
    FIELD_DELIMITER = '|'
);
```

Good for:

-   Testing
-   One-off loads
-   Quick experiments

------------------------------------------------------------------------

# 51. Stage Default File Format vs COPY File Format

A stage can have a default file format.

For example:

``` text
Stage default:
FIELD_DELIMITER = ,
```

But your file might use:

``` text
|
```

You can specify another format during the load:

``` sql
COPY INTO region
FROM @stage
FILE_FORMAT = (
    FORMAT_NAME = MYPIPEFORMAT
);
```

The explicit load configuration can therefore handle the actual file
correctly.

------------------------------------------------------------------------

# 52. `PUT`

`PUT` uploads local files to a Snowflake stage.

Conceptually:

``` text
Local Computer
      |
     PUT
      |
      v
Snowflake Stage
```

Example:

``` sql
PUT 'file:///tmp/students.csv'
@my_stage;
```

Remember:

``` text
PUT = Local -> Stage
```

------------------------------------------------------------------------

# 53. `GET`

`GET` downloads files from a Snowflake stage to a local location.

Example:

``` sql
GET @my_stage
'file:///tmp/download/';
```

Remember:

``` text
GET = Stage -> Local
```

------------------------------------------------------------------------

# 54. `PUT` vs `COPY INTO`

These are different operations.

``` text
Local File
    |
   PUT
    |
    v
  Stage
    |
 COPY INTO
    |
    v
 Table
```

So:

``` text
PUT
-> Upload file to stage

COPY INTO
-> Load staged data into table
```

------------------------------------------------------------------------

# 55. Named Internal Stage Example

Create a stage:

``` sql
CREATE OR REPLACE STAGE my_stage;
```

Upload:

``` sql
PUT 'file:///tmp/data.csv'
@my_stage;
```

Inspect:

``` sql
LIST @my_stage;
```

Create a format:

``` sql
CREATE OR REPLACE FILE FORMAT my_csv_format
TYPE = CSV
FIELD_DELIMITER = ','
SKIP_HEADER = 1;
```

Load:

``` sql
COPY INTO students
FROM @my_stage
FILE_FORMAT = (
    FORMAT_NAME = my_csv_format
);
```

------------------------------------------------------------------------

# 56. Table Stage

Every table has a table stage.

Reference it with:

``` sql
@%table_name
```

Example:

``` sql
LIST @%students;
```

Important:

``` text
@%students
```

is the table stage for `students`.

------------------------------------------------------------------------

# 57. User Stage

A user has a user stage.

Syntax:

``` sql
@~
```

Example:

``` sql
LIST @~;
```

Important:

``` text
@~ -> User stage
```

------------------------------------------------------------------------

# 58. `FORCE = TRUE`

Snowflake tracks load metadata to avoid unintentionally loading the same
files repeatedly.

If a file has already been loaded, a subsequent normal `COPY INTO` can
recognize it as already loaded.

To intentionally reload:

``` sql
COPY INTO students
FROM @my_stage
FILE_FORMAT = (
    FORMAT_NAME = my_csv_format
)
FORCE = TRUE;
```

Be careful:

> `FORCE = TRUE` can create duplicate data if you do not have a
> deduplication strategy.

------------------------------------------------------------------------

# 59. Load Metadata and History

Snowflake keeps load-related metadata.

This can help answer:

-   Which files were loaded?
-   When were they loaded?
-   How many rows were parsed?
-   How many rows were loaded?
-   Were there errors?

Snowflake provides load-history features such as `COPY_HISTORY`.

This is important in production pipelines because you need to know
whether ingestion succeeded.

------------------------------------------------------------------------

# 60. Load Result

After running `COPY INTO`, Snowflake returns load information such as:

``` text
file
status
rows_parsed
rows_loaded
errors_seen
first_error
```

Conceptually:

``` text
file           region.tbl
status         LOADED
rows_parsed    5
rows_loaded    5
errors_seen    0
```

For a partial load:

``` text
rows_parsed    25
rows_loaded    20
errors_seen    5
```

------------------------------------------------------------------------

# 61. Professional Loading Workflow

A robust workflow is:

``` text
1. LIST files
       |
       v
2. Inspect stage
       |
       v
3. Inspect/define file format
       |
       v
4. Preview staged data
       |
       v
5. Validate
       |
       v
6. COPY INTO
       |
       v
7. Inspect load result
       |
       v
8. Verify target table
       |
       v
9. Check load history/errors
```

Useful commands:

``` sql
LIST @my_stage;
```

``` sql
DESCRIBE STAGE my_stage;
```

``` sql
SELECT $1, $2, $3
FROM @my_stage
(FILE_FORMAT => 'my_format');
```

``` sql
COPY INTO my_table
FROM @my_stage
VALIDATION_MODE = RETURN_ALL_ERRORS;
```

``` sql
COPY INTO my_table
FROM @my_stage;
```

``` sql
SELECT *
FROM my_table;
```

------------------------------------------------------------------------

# 62. Complete Practical Example

## Step 1 --- Create table

``` sql
CREATE OR REPLACE TABLE students (
    id INTEGER,
    name VARCHAR,
    country VARCHAR,
    age INTEGER
);
```

## Step 2 --- Create file format

Suppose the file contains:

``` text
1|Santosh|India|21
2|Rahul|India|22
3|Aman|India|23
```

Create:

``` sql
CREATE OR REPLACE FILE FORMAT student_pipe_format
TYPE = CSV
FIELD_DELIMITER = '|'
SKIP_HEADER = 0;
```

## Step 3 --- Inspect stage

``` sql
LIST @student_stage;
```

## Step 4 --- Preview

``` sql
SELECT
    $1,
    $2,
    $3,
    $4
FROM @student_stage
(FILE_FORMAT => 'student_pipe_format');
```

## Step 5 --- Validate

``` sql
COPY INTO students
FROM @student_stage
FILE_FORMAT = (
    FORMAT_NAME = student_pipe_format
)
VALIDATION_MODE = RETURN_ERRORS;
```

## Step 6 --- Load

``` sql
COPY INTO students
FROM @student_stage
FILE_FORMAT = (
    FORMAT_NAME = student_pipe_format
);
```

## Step 7 --- Verify

``` sql
SELECT *
FROM students;
```

------------------------------------------------------------------------

# 63. Practice Lab 1 --- Basic Structured Loading

Use this logical file:

``` text
1|Laptop|Dell|50000
2|Phone|Samsung|30000
3|Tablet|Apple|45000
```

Target table:

``` text
PRODUCT_ID
PRODUCT_NAME
BRAND
PRICE
```

### Tasks

-   Create the target table.
-   Create a pipe-delimited CSV file format.
-   List the stage.
-   Preview the file using `$1`, `$2`, `$3`, `$4`.
-   Validate the file.
-   Load the file using `COPY INTO`.
-   Query the final table.

### Starter Code

``` sql
CREATE OR REPLACE TABLE products (
    product_id INTEGER,
    product_name VARCHAR,
    brand VARCHAR,
    price INTEGER
);

CREATE OR REPLACE FILE FORMAT product_pipe_format
TYPE = CSV
FIELD_DELIMITER = '|';
```

Then complete:

``` sql
LIST @your_stage;

SELECT
    $1,
    $2,
    $3,
    $4
FROM @your_stage
(FILE_FORMAT => 'product_pipe_format');

COPY INTO products
FROM @your_stage
FILE_FORMAT = (
    FORMAT_NAME = product_pipe_format
)
VALIDATION_MODE = RETURN_ERRORS;

COPY INTO products
FROM @your_stage
FILE_FORMAT = (
    FORMAT_NAME = product_pipe_format
);

SELECT *
FROM products;
```

------------------------------------------------------------------------

# 64. Practice Lab 2 --- Error Handling

Suppose the source file is:

``` text
1|Santosh|21
2|Rahul|twenty
3|Aman|23
4|Ravi|twenty-five
```

Target:

``` sql
CREATE OR REPLACE TABLE employee (
    id INTEGER,
    name VARCHAR,
    age INTEGER
);
```

There are:

``` text
4 total rows
2 bad rows
2 good rows
```

Test:

### A. `ABORT_STATEMENT`

``` sql
COPY INTO employee
FROM @my_stage
ON_ERROR = ABORT_STATEMENT;
```

Expected:

``` text
Load fails.
```

### B. `CONTINUE`

``` sql
COPY INTO employee
FROM @my_stage
ON_ERROR = CONTINUE;
```

Expected:

``` text
Good rows -> loaded
Bad rows  -> rejected
```

### C. `SKIP_FILE_1`

``` sql
COPY INTO employee
FROM @my_stage
ON_ERROR = SKIP_FILE_1;
```

Since the file has errors reaching the threshold, the file is skipped.

### D. `SKIP_FILE_3`

``` sql
COPY INTO employee
FROM @my_stage
ON_ERROR = SKIP_FILE_3;
```

Two errors are below the threshold of three, so the file is not skipped
because of that threshold.

------------------------------------------------------------------------

# 65. Practice Lab 3 --- Fix a File Format

Source:

``` text
100|Laptop|Dell
101|Phone|Samsung
102|Tablet|Apple
```

Incorrect format:

``` sql
CREATE OR REPLACE FILE FORMAT wrong_format
TYPE = CSV
FIELD_DELIMITER = ',';
```

Preview:

``` sql
SELECT
    $1,
    $2,
    $3
FROM @my_stage
(FILE_FORMAT => 'wrong_format');
```

Question:

What does `$1` contain?

Expected concept:

``` text
100|Laptop|Dell
```

because Snowflake cannot find commas.

Fix:

``` sql
CREATE OR REPLACE FILE FORMAT correct_format
TYPE = CSV
FIELD_DELIMITER = '|';
```

Now:

``` text
$1 -> 100
$2 -> Laptop
$3 -> Dell
```

------------------------------------------------------------------------

# 66. Practice Lab 4 --- GZIP

Suppose the stage contains:

``` text
products.csv
products.csv.gz
```

Create:

``` sql
CREATE OR REPLACE FILE FORMAT gzip_format
TYPE = CSV
FIELD_DELIMITER = '|'
COMPRESSION = GZIP;
```

Question:

Can `gzip_format` safely be used for an uncompressed `products.csv`
file?

Answer:

``` text
No.
```

Why?

Because:

``` sql
COMPRESSION = GZIP
```

tells Snowflake to expect GZIP compression.

An alternative is:

``` sql
CREATE OR REPLACE FILE FORMAT auto_format
TYPE = CSV
FIELD_DELIMITER = '|';
```

when relying on supported compression detection.

------------------------------------------------------------------------

# 67. Practice Lab 5 --- Validation

Target table:

``` sql
CREATE TABLE test_load (
    id INTEGER,
    name VARCHAR,
    age INTEGER
);
```

Source:

``` text
1|Santosh|21
2|Rahul|twenty
3|Aman|22
```

Run:

``` sql
COPY INTO test_load
FROM @my_stage
VALIDATION_MODE = RETURN_ALL_ERRORS;
```

Then:

``` sql
SELECT *
FROM test_load;
```

Expected:

``` text
The validation operation itself does not load the rows.
```

------------------------------------------------------------------------

# 68. Practice Lab 6 --- Transformation

Source:

``` text
1|santosh
2|rahul
3|aman
```

Target:

``` sql
CREATE TABLE users (
    id INTEGER,
    name VARCHAR
);
```

Load uppercase:

``` sql
COPY INTO users
FROM (
    SELECT
        $1::INTEGER,
        UPPER($2)
    FROM @my_stage
)
FILE_FORMAT = (
    FORMAT_NAME = my_format
);
```

Expected:

``` text
1 | SANTOSH
2 | RAHUL
3 | AMAN
```

------------------------------------------------------------------------

# 69. Common Mistakes

## Mistake 1 --- Wrong delimiter

File:

``` text
1|India|Asia
```

Format:

``` sql
FIELD_DELIMITER = ','
```

Problem:

``` text
Fields are not split as intended.
```

------------------------------------------------------------------------

## Mistake 2 --- Wrong compression

Format:

``` sql
COMPRESSION = GZIP
```

File:

``` text
data.csv
```

Problem:

``` text
The format expects GZIP compression.
```

------------------------------------------------------------------------

## Mistake 3 --- Wrong column order

File:

``` text
1|India|5
```

Target order:

``` text
id
region_id
country
```

Snowflake can attempt:

``` text
India -> INTEGER
```

causing an error.

------------------------------------------------------------------------

## Mistake 4 --- Assuming validation loads data

``` sql
VALIDATION_MODE = RETURN_ERRORS
```

does not perform the normal table load.

------------------------------------------------------------------------

## Mistake 5 --- Confusing `CONTINUE` and `SKIP_FILE`

``` text
CONTINUE
-> Continue with valid rows

SKIP_FILE
-> Skip the file
```

------------------------------------------------------------------------

## Mistake 6 --- Using `FORCE = TRUE` carelessly

It can cause duplicate data if the same file is loaded repeatedly
without deduplication.

------------------------------------------------------------------------

# 70. Original Lab vs Important Gaps

The core lab covers:

  Topic                     Covered
  ----------------------- ---------
  External stage                Yes
  `LIST`                        Yes
  `DESCRIBE STAGE`              Yes
  File formats                  Yes
  Pipe delimiter                Yes
  GZIP                          Yes
  Compression detection         Yes
  `COPY INTO`                   Yes
  Validation                    Yes
  `RETURN_ALL_ERRORS`           Yes
  `RETURN_ERRORS`               Yes
  `RETURN_10_ROWS`              Yes
  `ON_ERROR`                    Yes
  `CONTINUE`                    Yes
  `ABORT_STATEMENT`             Yes
  `SKIP_FILE_n`                 Yes
  Transformations               Yes
  User stages                   Gap
  Table stages                  Gap
  Named stages                  Gap
  `PUT`                         Gap
  `GET`                         Gap
  Headers                       Gap
  CSV quoting                   Gap
  NULL handling                 Gap
  `PATTERN`                     Gap
  `FORCE`                       Gap
  Load history                  Gap
  Production workflow           Gap

------------------------------------------------------------------------

# 71. Exam Cheat Sheet

``` text
@~
    -> User stage

@%table
    -> Table stage

@stage
    -> Named stage
```

``` text
LIST
    -> See staged files
```

``` text
DESCRIBE STAGE
    -> Inspect stage properties
```

``` text
$1, $2, $3
    -> Fields from staged file
```

``` text
FILE FORMAT
    -> Rules for reading the file
```

``` text
COPY INTO
    -> Load staged data into table
```

``` text
FILES
    -> Select exact files
```

``` text
PATTERN
    -> Regex-based file selection
```

``` text
VALIDATION_MODE
    -> Validate without performing the normal load
```

``` text
ABORT_STATEMENT
    -> Fail the load
```

``` text
CONTINUE
    -> Continue with valid rows
```

``` text
SKIP_FILE_n
    -> File-level error threshold
```

``` text
PUT
    -> Local -> Stage
```

``` text
GET
    -> Stage -> Local
```

``` text
FORCE = TRUE
    -> Force reload
```

------------------------------------------------------------------------

# 72. Accenture-Style MCQs

## Q1. Which command loads staged files into a Snowflake table?

A. `LOAD INTO`\
B. `COPY INTO`\
C. `PUT INTO`\
D. `INSERT FILE`

**Answer: B --- `COPY INTO`**

------------------------------------------------------------------------

## Q2. What does `LIST @stage` do?

A. Loads files\
B. Deletes files\
C. Lists files in a stage\
D. Creates a stage

**Answer: C**

------------------------------------------------------------------------

## Q3. What does `$1` represent when querying a staged CSV file?

A. First row\
B. First field/column\
C. First file\
D. First table

**Answer: B**

------------------------------------------------------------------------

## Q4. Which parameter defines the field delimiter?

A. `ROW_DELIMITER`\
B. `FIELD_DELIMITER`\
C. `COLUMN_SEPARATOR`\
D. `DELIMITER_TYPE`

**Answer: B**

------------------------------------------------------------------------

## Q5. Which parameter is used to validate staged data without performing the normal load?

A. `CHECK_DATA`\
B. `VALIDATE`\
C. `VALIDATION_MODE`\
D. `VERIFY`

**Answer: C**

------------------------------------------------------------------------

## Q6. What does `ON_ERROR = CONTINUE` do?

A. Stops immediately\
B. Skips all rows\
C. Continues loading valid data despite errors\
D. Deletes the source file

**Answer: C**

------------------------------------------------------------------------

## Q7. What is the default `ON_ERROR` behavior?

A. `CONTINUE`\
B. `SKIP_FILE`\
C. `ABORT_STATEMENT`\
D. `RETURN_ERRORS`

**Answer: C**

------------------------------------------------------------------------

## Q8. What does `PUT` do?

A. Table -\> Stage\
B. Local -\> Stage\
C. Stage -\> Local\
D. Table -\> Local

**Answer: B**

------------------------------------------------------------------------

## Q9. What does `GET` do?

A. Local -\> Stage\
B. Stage -\> Local\
C. Table -\> Stage\
D. Database -\> Table

**Answer: B**

------------------------------------------------------------------------

## Q10. Which syntax represents a table stage?

A. `@~`\
B. `@%table_name`\
C. `@#table_name`\
D. `@table%`

**Answer: B**

------------------------------------------------------------------------

## Q11. Which syntax represents a user stage?

A. `@~`\
B. `@%`\
C. `@user`\
D. `@USER_STAGE`

**Answer: A**

------------------------------------------------------------------------

## Q12. What does `SKIP_FILE_4` conceptually mean?

A. Skip four files\
B. Skip the first four rows\
C. Skip the file when the error threshold of four is reached\
D. Retry four times

**Answer: C**

------------------------------------------------------------------------

# 73. Interview Questions

## Q1. How do you load a CSV file into Snowflake?

A good answer:

> I first make the file accessible through a stage, define an
> appropriate file format, preview or validate the staged data, and then
> use `COPY INTO` to load it into the target table. I also configure
> error handling and verify the load result.

------------------------------------------------------------------------

## Q2. What is the difference between a stage and a file format?

**Stage:**

> Specifies where the source files are located or accessed.

**File format:**

> Specifies how Snowflake should interpret those files.

Remember:

``` text
Stage      -> WHERE
File Format -> HOW
```

------------------------------------------------------------------------

## Q3. What is `COPY INTO`?

> `COPY INTO` is Snowflake's command for loading data from staged files
> into a table. It supports file selection, file formats, validation,
> error handling, and transformations.

------------------------------------------------------------------------

## Q4. What is `VALIDATION_MODE`?

> It allows staged data to be checked for errors without performing the
> normal table load.

------------------------------------------------------------------------

## Q5. Difference between `ABORT_STATEMENT` and `CONTINUE`?

``` text
ABORT_STATEMENT
-> Fail/abort the load

CONTINUE
-> Continue processing valid rows
```

------------------------------------------------------------------------

## Q6. Difference between `CONTINUE` and `SKIP_FILE`?

``` text
CONTINUE
-> Row-level tolerance

SKIP_FILE
-> File-level handling
```

------------------------------------------------------------------------

## Q7. Why use a file format?

> Snowflake needs parsing rules such as delimiter, compression, header
> handling, quoting, escaping, NULL representation, and other
> file-specific options.

------------------------------------------------------------------------

# 74. Final Mental Model

If you remember only one thing, remember:

``` text
                    FILE
                     |
                     v
                  STAGE
                     |
                     v
               FILE FORMAT
                     |
                     v
                  PREVIEW
                     |
                     v
                 VALIDATE
                     |
                     v
                 COPY INTO
                     |
          +----------+----------+
          |                     |
       SUCCESS                ERROR
          |                     |
          v                     v
        TABLE               ON_ERROR
                              |
              +---------------+---------------+
              |               |               |
            ABORT          CONTINUE       SKIP_FILE
```

The five most important concepts:

``` text
LIST
  -> Find files

SELECT $1,$2,...
  -> Preview staged data

CREATE FILE FORMAT
  -> Tell Snowflake how to parse

VALIDATION_MODE
  -> Check before loading

COPY INTO
  -> Actually load
```

And the major error-handling concept:

``` text
ON_ERROR
```

------------------------------------------------------------------------

# 75. Final Revision Card

``` text
STRUCTURED DATA
-> Data with defined structure/schema

STAGE
-> Location/access point for source files

FILE FORMAT
-> Rules for interpreting source files

LIST
-> Lists staged files

DESCRIBE STAGE
-> Shows stage properties

$1, $2, $3
-> Positional fields from staged files

COPY INTO
-> Loads staged data into tables

FILES
-> Select exact files

PATTERN
-> Regex-based file selection

VALIDATION_MODE
-> Validate staged data without normal loading

ON_ERROR
-> Controls load behavior on errors

ABORT_STATEMENT
-> Abort/fail load

CONTINUE
-> Continue with valid rows

SKIP_FILE_n
-> Skip file when error threshold is reached

PUT
-> Local -> Stage

GET
-> Stage -> Local

@~
-> User stage

@%table
-> Table stage

@stage
-> Named stage

FORCE = TRUE
-> Force reload
```

------------------------------------------------------------------------

# 76. What You Should Practice Before Moving On

Do not move to the next module until you can do these without looking at
the notes:

1.  Create a target table.
2.  Create a CSV file format.
3.  Explain `FIELD_DELIMITER`.
4.  Use `LIST`.
5.  Preview a staged file using `$1`, `$2`, `$3`.
6.  Explain why a wrong delimiter breaks parsing.
7.  Load a file using `COPY INTO`.
8.  Validate a file using `VALIDATION_MODE`.
9.  Explain `ABORT_STATEMENT`.
10. Explain `CONTINUE`.
11. Explain `SKIP_FILE_n`.
12. Explain `FILES` vs `PATTERN`.
13. Explain GZIP loading.
14. Explain `PUT` vs `COPY INTO`.
15. Explain `PUT` vs `GET`.
16. Explain `@~` and `@%table`.
17. Perform a transformation while loading.
18. Cast a staged value to a target datatype.
19. Explain why column order can cause datatype errors.
20. Explain why `FORCE = TRUE` can create duplicate data.

------------------------------------------------------------------------

## One-Line Summary

> **Snowflake structured-data loading is the process of making source
> files available through a stage, defining how Snowflake should parse
> them with a file format, optionally previewing and validating them,
> and then loading them into tables using `COPY INTO`, with configurable
> transformation and error-handling behavior.**
