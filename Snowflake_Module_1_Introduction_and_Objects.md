# ❄️ Snowflake Module 1 --- Introduction to Snowflake & Snowflake Objects

> **Goal:** Build a strong foundation in Snowflake before moving to data
> loading, unloading, functions, transactions, Time Travel, caching, and
> Snowsight visualizations.

------------------------------------------------------------------------

## 📚 What This Module Covers

This module is based on the **Introduction to Snowflake and Snowflake
Objects** lab and expands the concepts needed to understand the lab
properly.

### Covered in the lab

-   Snowsight
-   Worksheets
-   Sessions
-   Worksheet context
-   Roles
-   Virtual warehouses
-   Databases
-   Schemas
-   Tables
-   `CREATE`
-   `USE`
-   `SHOW`
-   `SELECT`
-   `INSERT`
-   `JOIN`
-   `LIMIT`
-   `TOP`
-   User defaults
-   Session variables
-   Stored procedure basics

### Important supporting concepts

-   Snowflake architecture
-   Storage vs. compute
-   Warehouse sizing
-   Auto-suspend / auto-resume
-   Fully qualified object names
-   Context functions
-   DDL vs. DML vs. DCL vs. TCL
-   Role vs. user
-   Privileges
-   `GRANT` / `REVOKE`
-   `DESCRIBE`
-   Metadata operations
-   Sessions

------------------------------------------------------------------------

# 1. Snowflake Big Picture

Snowflake separates **storage** from **compute**.

``` text
                         SNOWFLAKE
                             |
            +----------------+----------------+
            |                |                |
         STORAGE           COMPUTE       CLOUD SERVICES
            |                |                |
       Databases         Warehouses       Authentication
       Schemas            CPU/Memory       Metadata
       Tables             Query execution  Security
       Stages                              Optimization
```

## Three Major Layers

### 1. Storage

This is where persistent data is stored.

``` text
Database
   ↓
Schema
   ↓
Table
   ↓
Rows + Columns
```

### 2. Compute

A **Virtual Warehouse** provides compute resources used to execute SQL
and data-processing operations.

``` text
SQL Query
   ↓
Virtual Warehouse
   ↓
CPU + Memory
   ↓
Query Result
```

### 3. Cloud Services

The cloud services layer handles activities such as:

-   Authentication
-   Metadata management
-   Access control
-   Query coordination
-   Infrastructure services

------------------------------------------------------------------------

# 2. Snowflake Account

A Snowflake account is the overall environment in which users, roles,
warehouses, databases, and other objects exist.

``` text
Snowflake Account
│
├── Users
├── Roles
├── Warehouses
├── Databases
│   ├── Schemas
│   │   ├── Tables
│   │   ├── Views
│   │   └── Stages
│
└── Other Snowflake Objects
```

------------------------------------------------------------------------

# 3. Database

A **database** is a container for schemas.

``` text
DATABASE
   |
   +── SCHEMA 1
   +── SCHEMA 2
   +── SCHEMA 3
```

## Create a Database

``` sql
CREATE OR REPLACE DATABASE LEARNING_DB;
```

### `OR REPLACE`

``` sql
CREATE OR REPLACE DATABASE my_db;
```

This means Snowflake creates the object if it does not exist and
replaces it if it already exists.

> ⚠️ Be careful with `OR REPLACE` because replacing an existing object
> can have destructive consequences.

------------------------------------------------------------------------

# 4. Schema

A **schema** organizes objects inside a database.

``` text
Database
   |
   +── Schema A
   |      +── Table
   |      +── View
   |      +── Stage
   |
   +── Schema B
          +── Table
          +── View
```

## Create a Schema

``` sql
CREATE OR REPLACE SCHEMA LEARNING_DB.PRACTICE;
```

## Use a Schema

``` sql
USE SCHEMA LEARNING_DB.PRACTICE;
```

------------------------------------------------------------------------

# 5. Table

A table contains structured data organized into rows and columns.

``` sql
CREATE OR REPLACE TABLE employees (
    employee_id INTEGER,
    employee_name VARCHAR,
    salary NUMBER
);
```

Conceptually:

``` text
employees
+-------------+---------------+--------+
| employee_id | employee_name | salary |
+-------------+---------------+--------+
| 1           | Santosh       | 50000  |
| 2           | Rahul         | 60000  |
+-------------+---------------+--------+
```

## Insert Data

``` sql
INSERT INTO employees
VALUES
    (1, 'Santosh', 50000),
    (2, 'Rahul', 60000);
```

## Query Data

``` sql
SELECT *
FROM employees;
```

------------------------------------------------------------------------

# 6. Object Naming

Snowflake commonly identifies objects using:

``` text
database.schema.object
```

For a table:

``` text
database.schema.table
```

Example:

``` sql
SELECT *
FROM LEARNING_DB.PRACTICE.EMPLOYEES;
```

Here:

``` text
LEARNING_DB  → Database
PRACTICE     → Schema
EMPLOYEES    → Table
```

------------------------------------------------------------------------

# 7. Fully Qualified Names

A fully qualified table name contains:

``` text
DATABASE.SCHEMA.TABLE
```

Example:

``` sql
SELECT *
FROM SNOWBEARAIR_DB.PROMO_CATALOG_SALES.CUSTOMER;
```

This explicitly tells Snowflake exactly which object you want.

## Why use it?

It avoids ambiguity and reduces dependence on the current
database/schema context.

------------------------------------------------------------------------

# 8. Snowflake Session Context

A Snowflake worksheet operates within a session.

The session has a context consisting of:

``` text
SESSION CONTEXT
      |
      +── ROLE
      +── WAREHOUSE
      +── DATABASE
      +── SCHEMA
```

## Why Context Matters

Suppose your context is:

``` text
DATABASE = LEARNING_DB
SCHEMA   = PRACTICE
```

Then:

``` sql
SELECT *
FROM employees;
```

is interpreted as:

``` sql
SELECT *
FROM LEARNING_DB.PRACTICE.EMPLOYEES;
```

If the table exists elsewhere, use its fully qualified name.

------------------------------------------------------------------------

# 9. Setting Context

## Set Role

``` sql
USE ROLE ANALYST;
```

## Set Warehouse

``` sql
USE WAREHOUSE LEARNING_WH;
```

## Set Database

``` sql
USE DATABASE LEARNING_DB;
```

## Set Schema

``` sql
USE SCHEMA LEARNING_DB.PRACTICE;
```

------------------------------------------------------------------------

# 10. Checking Current Context

A useful practice not emphasized enough in the basic lab is checking the
context programmatically.

``` sql
SELECT
    CURRENT_ROLE() AS ROLE,
    CURRENT_WAREHOUSE() AS WAREHOUSE,
    CURRENT_DATABASE() AS DATABASE,
    CURRENT_SCHEMA() AS SCHEMA;
```

Example result:

``` text
ROLE       | WAREHOUSE   | DATABASE    | SCHEMA
-----------|-------------|-------------|---------
ANALYST    | LEARNING_WH | LEARNING_DB | PRACTICE
```

This is especially useful when debugging SQL.

------------------------------------------------------------------------

# 11. Virtual Warehouse

A **virtual warehouse** provides compute resources.

It is **not where your permanent table data is stored**.

Think:

``` text
Table
  |
  | Persistent data
  ↓
Storage

Warehouse
  |
  | CPU + Memory
  ↓
Query Processing
```

## Create a Warehouse

``` sql
CREATE OR REPLACE WAREHOUSE LEARNING_WH
WITH
    WAREHOUSE_SIZE = XSMALL;
```

## Use the Warehouse

``` sql
USE WAREHOUSE LEARNING_WH;
```

------------------------------------------------------------------------

# 12. Why Do We Need a Warehouse?

When you run a compute-intensive SQL operation:

``` sql
SELECT *
FROM employees;
```

Snowflake uses a warehouse to execute the query.

``` text
SQL
 ↓
Warehouse
 ↓
CPU / Memory
 ↓
Process Data
 ↓
Result
```

This separation of storage and compute allows compute resources to be
managed independently from stored data.

------------------------------------------------------------------------

# 13. Warehouse States

A warehouse can be:

``` text
RUNNING
   🟢

SUSPENDED
   ⚪
```

### Running

Compute resources are active.

### Suspended

The warehouse is not actively processing queries.

## Suspend a Warehouse

``` sql
ALTER WAREHOUSE LEARNING_WH SUSPEND;
```

## Resume a Warehouse

``` sql
ALTER WAREHOUSE LEARNING_WH RESUME;
```

------------------------------------------------------------------------

# 14. Auto-Suspend and Auto-Resume

You generally don't want a warehouse running unnecessarily.

Example:

``` sql
CREATE OR REPLACE WAREHOUSE LEARNING_WH
WITH
    WAREHOUSE_SIZE = XSMALL
    AUTO_SUSPEND = 300
    AUTO_RESUME = TRUE;
```

Meaning:

``` text
No activity
    ↓
300 seconds
    ↓
Warehouse suspends
```

When a new query arrives:

``` text
Query
  ↓
AUTO_RESUME
  ↓
Warehouse starts
  ↓
Query executes
```

------------------------------------------------------------------------

# 15. Warehouse Size

Snowflake provides warehouse sizes such as:

``` text
XSMALL
SMALL
MEDIUM
LARGE
XLARGE
...
```

Conceptually:

``` text
XSMALL → Less compute
SMALL  → More compute
MEDIUM → More compute
LARGE  → More compute
```

A larger warehouse generally provides more compute capacity, but can
also increase cost.

> **Important:** Bigger does not automatically mean better. Choose a
> size based on workload requirements.

------------------------------------------------------------------------

# 16. Roles

A **role** controls what a user can access and what actions they can
perform through granted privileges.

Think:

``` text
USER
 ↓
ROLE
 ↓
PRIVILEGES
 ↓
OBJECTS
```

Example:

``` text
Santosh
   ↓
ANALYST
   ↓
SELECT
   ↓
EMPLOYEES
```

------------------------------------------------------------------------

# 17. User vs. Role

This is a common interview question.

### User

Represents **who you are**.

### Role

Represents **what permissions you have**.

``` text
USER = Santosh

ROLE = ANALYST

ANALYST
   ↓
Can SELECT
Cannot DELETE
```

------------------------------------------------------------------------

# 18. Changing the Active Role

``` sql
USE ROLE ANALYST;
```

The active role affects what you can see and what you can do.

For example:

``` text
ROLE A
 ↓
More privileges
 ↓
More accessible objects

ROLE B
 ↓
Fewer privileges
 ↓
Fewer accessible objects
```

> **Lab note:** The provided lab text contains an inconsistency in one
> role-changing step: the prose refers to changing to `PUBLIC`, while
> the SQL shown uses `USE ROLE admin;`. Do not memorize the specific lab
> command; understand the general `USE ROLE <role_name>` concept.

------------------------------------------------------------------------

# 19. SHOW Commands

`SHOW` commands are useful for inspecting Snowflake metadata.

Examples:

``` sql
SHOW DATABASES;
```

``` sql
SHOW SCHEMAS;
```

``` sql
SHOW TABLES;
```

``` sql
SHOW WAREHOUSES;
```

``` sql
SHOW ROLES;
```

The lab specifically demonstrates:

``` sql
SHOW TABLES;
```

and notes that this metadata operation does not require the warehouse to
process table data.

------------------------------------------------------------------------

# 20. DESCRIBE

Use `DESCRIBE` to inspect an object's structure.

``` sql
DESCRIBE TABLE employees;
```

Short form:

``` sql
DESC TABLE employees;
```

This helps you inspect information such as:

-   Column names
-   Data types
-   Nullable information
-   Defaults
-   Other table metadata

------------------------------------------------------------------------

# 21. SELECT

The basic query:

``` sql
SELECT *
FROM employees;
```

Instead of selecting every column, prefer selecting only what you need:

``` sql
SELECT employee_id, employee_name, salary
FROM employees;
```

------------------------------------------------------------------------

# 22. LIMIT

`LIMIT` restricts the number of rows returned.

``` sql
SELECT *
FROM employees
LIMIT 5;
```

Meaning:

> Return at most 5 rows.

------------------------------------------------------------------------

# 23. TOP

Snowflake also supports `TOP`.

``` sql
SELECT TOP 5
    employee_name,
    salary
FROM employees
ORDER BY salary DESC;
```

Execution concept:

``` text
ORDER BY salary DESC
        ↓
Highest salary first
        ↓
TOP 5
        ↓
Return 5 rows
```

------------------------------------------------------------------------

# 24. ORDER BY

Use `ORDER BY` to sort query results.

Ascending:

``` sql
SELECT *
FROM employees
ORDER BY salary ASC;
```

Descending:

``` sql
SELECT *
FROM employees
ORDER BY salary DESC;
```

------------------------------------------------------------------------

# 25. JOIN

The lab also introduces a simple join between customers and orders.

Conceptually:

``` text
CUSTOMER
customer_id
     |
     | JOIN
     |
ORDERS
customer_id
```

Example:

``` sql
SELECT
    c.c_firstname,
    c.c_lastname,
    o.o_orderkey,
    o.o_totalprice
FROM promo_catalog_sales.orders o
JOIN promo_catalog_sales.customer c
    ON o.o_custkey = c.c_custkey
ORDER BY o.o_totalprice DESC
LIMIT 10;
```

The join condition connects related records from the two tables.

------------------------------------------------------------------------

# 26. DDL vs DML vs DQL vs DCL vs TCL

This is an important SQL foundation.

  Category   Purpose                 Examples
  ---------- ----------------------- ---------------------------------------
  DDL        Define/change objects   `CREATE`, `ALTER`, `DROP`
  DML        Manipulate table data   `INSERT`, `UPDATE`, `DELETE`, `MERGE`
  DQL        Query data              `SELECT`
  DCL        Control permissions     `GRANT`, `REVOKE`
  TCL        Control transactions    `COMMIT`, `ROLLBACK`

You will use these categories throughout the remaining Snowflake
modules.

------------------------------------------------------------------------

# 27. Creating Objects

## Database

``` sql
CREATE DATABASE LEARNING_DB;
```

## Schema

``` sql
CREATE SCHEMA LEARNING_DB.PRACTICE;
```

## Table

``` sql
CREATE TABLE LEARNING_DB.PRACTICE.STUDENTS (
    student_id INTEGER,
    student_name VARCHAR,
    marks INTEGER
);
```

------------------------------------------------------------------------

# 28. Inserting Data

``` sql
INSERT INTO LEARNING_DB.PRACTICE.STUDENTS
VALUES
    (1, 'Santosh', 85),
    (2, 'Rahul', 72),
    (3, 'Amit', 91),
    (4, 'Priya', 88);
```

Query:

``` sql
SELECT *
FROM LEARNING_DB.PRACTICE.STUDENTS;
```

------------------------------------------------------------------------

# 29. User Defaults

If you normally work with the same role, warehouse, database, and
schema, you can configure defaults.

Example:

``` sql
ALTER USER my_user SET
    DEFAULT_ROLE = ANALYST
    DEFAULT_WAREHOUSE = LEARNING_WH
    DEFAULT_NAMESPACE = LEARNING_DB.PRACTICE;
```

Conceptually:

``` text
New Session
    ↓
Default Role
Default Warehouse
Default Database
Default Schema
```

`DEFAULT_NAMESPACE` represents:

``` text
DATABASE.SCHEMA
```

------------------------------------------------------------------------

# 30. Session Variables

Snowflake supports session variables.

Set variables:

``` sql
SET myfirstname = 'Santosh';
SET mylastname = 'Singh';
SET myemail = 'example@example.com';
```

Reference them with `$`:

``` sql
SELECT
    $myfirstname,
    $mylastname,
    $myemail;
```

Pattern:

``` text
SET variable
     ↓
$variable
```

------------------------------------------------------------------------

# 31. Stored Procedure Basics

A stored procedure contains reusable logic that can be executed using
`CALL`.

Example:

``` sql
CALL training_db.common.update_profile(
    CURRENT_USER(),
    $myfirstname,
    $mylastname,
    $myemail
);
```

For this module, understand:

``` text
CALL procedure(...)
        ↓
Execute predefined procedural logic
```

Detailed stored procedure development can be studied separately.

------------------------------------------------------------------------

# 32. Snowsight

Snowsight is Snowflake's web interface.

Important areas include:

-   Home
-   Navigation bar
-   Worksheets
-   Object browser
-   SQL editor
-   Settings
-   User menu

The key technical idea is:

> **Snowsight is the interface; SQL is what you use to work with
> Snowflake objects and data.**

------------------------------------------------------------------------

# 33. Worksheet = Session

A worksheet is connected to a Snowflake session.

Think:

``` text
Worksheet A
    ↓
Session A
    ↓
Context A

Worksheet B
    ↓
Session B
    ↓
Context B
```

Each session has its own active context.

This is why you should always know:

``` sql
SELECT
    CURRENT_ROLE(),
    CURRENT_WAREHOUSE(),
    CURRENT_DATABASE(),
    CURRENT_SCHEMA();
```

------------------------------------------------------------------------

# 34. Complete Hands-On Lab

## Step 1 --- Create Warehouse

``` sql
CREATE OR REPLACE WAREHOUSE LEARNING_WH
WITH
    WAREHOUSE_SIZE = XSMALL
    AUTO_SUSPEND = 300
    AUTO_RESUME = TRUE;
```

## Step 2 --- Use Warehouse

``` sql
USE WAREHOUSE LEARNING_WH;
```

## Step 3 --- Create Database

``` sql
CREATE OR REPLACE DATABASE LEARNING_DB;
```

## Step 4 --- Create Schema

``` sql
CREATE OR REPLACE SCHEMA LEARNING_DB.PRACTICE;
```

## Step 5 --- Set Context

``` sql
USE DATABASE LEARNING_DB;

USE SCHEMA PRACTICE;
```

## Step 6 --- Verify Context

``` sql
SELECT
    CURRENT_ROLE() AS ROLE,
    CURRENT_WAREHOUSE() AS WAREHOUSE,
    CURRENT_DATABASE() AS DATABASE,
    CURRENT_SCHEMA() AS SCHEMA;
```

## Step 7 --- Create Table

``` sql
CREATE OR REPLACE TABLE students (
    student_id INTEGER,
    student_name VARCHAR,
    course VARCHAR,
    marks INTEGER
);
```

## Step 8 --- Insert Data

``` sql
INSERT INTO students
VALUES
    (1, 'Santosh', 'Snowflake', 85),
    (2, 'Rahul', 'Snowflake', 72),
    (3, 'Amit', 'SQL', 91),
    (4, 'Priya', 'Snowflake', 88);
```

## Step 9 --- Query

``` sql
SELECT *
FROM students;
```

## Step 10 --- Inspect Metadata

``` sql
SHOW TABLES;
```

``` sql
DESCRIBE TABLE students;
```

------------------------------------------------------------------------

# 35. Context Practice

Create another schema:

``` sql
CREATE OR REPLACE SCHEMA LEARNING_DB.TEST;
```

Switch to it:

``` sql
USE SCHEMA LEARNING_DB.TEST;
```

Now run:

``` sql
SELECT *
FROM students;
```

The query should fail because `students` exists in:

``` text
LEARNING_DB.PRACTICE
```

not:

``` text
LEARNING_DB.TEST
```

But this works:

``` sql
SELECT *
FROM LEARNING_DB.PRACTICE.students;
```

### What did you learn?

You just demonstrated why session context matters.

------------------------------------------------------------------------

# 36. Warehouse Practice

Suspend the warehouse:

``` sql
ALTER WAREHOUSE LEARNING_WH SUSPEND;
```

Check its metadata:

``` sql
SHOW WAREHOUSES;
```

Resume:

``` sql
ALTER WAREHOUSE LEARNING_WH RESUME;
```

------------------------------------------------------------------------

# 37. Query Practice

Run these in order:

### Practice 1

``` sql
SELECT *
FROM LEARNING_DB.PRACTICE.students;
```

### Practice 2

``` sql
SELECT student_name, marks
FROM LEARNING_DB.PRACTICE.students;
```

### Practice 3

``` sql
SELECT student_name, marks
FROM LEARNING_DB.PRACTICE.students
ORDER BY marks DESC;
```

### Practice 4

``` sql
SELECT TOP 2
    student_name,
    marks
FROM LEARNING_DB.PRACTICE.students
ORDER BY marks DESC;
```

### Practice 5

``` sql
SELECT *
FROM LEARNING_DB.PRACTICE.students
WHERE marks > 80;
```

------------------------------------------------------------------------

# 38. Mini Challenge 🧪

Create:

``` text
Database: AIRLINE_DB
Schema: FLIGHTS
Warehouse: AIRLINE_WH
Table: FLIGHT_DETAILS
```

The table should contain:

``` text
flight_id
airline
source
destination
price
```

Insert at least 5 records.

Then solve:

### Q1

Show all flights.

### Q2

Show flights where price \> 5000.

### Q3

Show the most expensive flight.

### Q4

Show the top 3 most expensive flights.

### Q5

Show the current role, warehouse, database, and schema.

### Q6

Show the table structure.

### Q7

Query the table using its fully qualified name.

------------------------------------------------------------------------

# 39. Mini Challenge --- Solution

### Q1

``` sql
SELECT *
FROM FLIGHT_DETAILS;
```

### Q2

``` sql
SELECT *
FROM FLIGHT_DETAILS
WHERE price > 5000;
```

### Q3

``` sql
SELECT *
FROM FLIGHT_DETAILS
ORDER BY price DESC
LIMIT 1;
```

### Q4

``` sql
SELECT TOP 3 *
FROM FLIGHT_DETAILS
ORDER BY price DESC;
```

### Q5

``` sql
SELECT
    CURRENT_ROLE(),
    CURRENT_WAREHOUSE(),
    CURRENT_DATABASE(),
    CURRENT_SCHEMA();
```

### Q6

``` sql
DESCRIBE TABLE FLIGHT_DETAILS;
```

### Q7

``` sql
SELECT *
FROM AIRLINE_DB.FLIGHTS.FLIGHT_DETAILS;
```

------------------------------------------------------------------------

# 40. Interview Questions

## Q1. What is a Snowflake virtual warehouse?

A virtual warehouse is a cluster of compute resources used to execute
SQL and data-processing operations. It is separate from Snowflake's
persistent storage.

## Q2. What is the difference between storage and compute?

**Storage** stores data.\
**Compute** processes data.

Snowflake allows them to scale independently.

## Q3. What is a database?

A database is a container for schemas.

## Q4. What is a schema?

A schema is a logical container inside a database for objects such as
tables, views, and stages.

## Q5. What is Snowflake session context?

It is the active role, warehouse, database, and schema used by the
session.

## Q6. Why is context important?

Because unqualified object names are resolved using the current
database/schema, the active role determines access, and the warehouse
provides compute for applicable operations.

## Q7. What is the difference between a user and a role?

A user identifies an individual/account identity. A role is a collection
of privileges that controls access.

## Q8. What is a fully qualified table name?

``` text
DATABASE.SCHEMA.TABLE
```

## Q9. What does `USE WAREHOUSE` do?

It selects the warehouse that the current session will use for
applicable compute operations.

## Q10. What does `SHOW TABLES` do?

It displays metadata about tables accessible in the relevant context.

## Q11. What does `DESC TABLE` do?

It describes the structure and metadata of a table.

## Q12. Why suspend a warehouse?

To stop active compute when it is not needed and avoid unnecessary
compute consumption.

------------------------------------------------------------------------

# 41. Quick Revision Sheet

``` text
DATABASE
    ↓
Contains schemas

SCHEMA
    ↓
Contains database objects

TABLE
    ↓
Contains structured data

WAREHOUSE
    ↓
Provides compute

ROLE
    ↓
Controls privileges

SESSION
    ↓
Has context

CONTEXT
    ↓
ROLE + WAREHOUSE + DATABASE + SCHEMA

FULLY QUALIFIED OBJECT
    ↓
DATABASE.SCHEMA.TABLE
```

### Most important commands

``` sql
CREATE DATABASE ...
CREATE SCHEMA ...
CREATE TABLE ...
CREATE WAREHOUSE ...

USE ROLE ...
USE WAREHOUSE ...
USE DATABASE ...
USE SCHEMA ...

SHOW DATABASES;
SHOW SCHEMAS;
SHOW TABLES;
SHOW WAREHOUSES;

DESCRIBE TABLE ...;

SELECT ...
INSERT INTO ...

ALTER WAREHOUSE ... SUSPEND;
ALTER WAREHOUSE ... RESUME;
```

------------------------------------------------------------------------

# 42. What You Should Know Before Module 2

Before moving to **Loading Structured Data**, you should be able to
explain:

-   What Snowflake is
-   Storage vs. compute
-   What a warehouse does
-   Database vs. schema vs. table
-   What a role does
-   What a session is
-   What context means
-   How to set context
-   How to check context
-   What a fully qualified name is
-   How to create a database
-   How to create a schema
-   How to create a table
-   How to insert data
-   How to query data
-   How `LIMIT` and `TOP` work
-   How `JOIN` works at a basic level
-   What `SHOW` does
-   What `DESCRIBE` does
-   Why warehouses can be suspended
-   What auto-suspend and auto-resume mean
-   Difference between DDL, DML, DQL, DCL, and TCL
-   Difference between user and role

------------------------------------------------------------------------

# 🎯 Final Mental Model

``` text
                         SNOWFLAKE
                             |
          +------------------+------------------+
          |                  |                  |
       STORAGE            COMPUTE         CLOUD SERVICES
          |                  |                  |
      DATABASE           WAREHOUSE         Security
          |               CPU/Memory        Metadata
        SCHEMA               |             Governance
          |                  |
        TABLE  ←────────── SQL
          |
        DATA


USER
  |
  ↓
ROLE
"What can I do?"
  |
  ↓
SESSION
  |
  +── WAREHOUSE
  |    "What compute do I use?"
  |
  +── DATABASE
  |    "Which database?"
  |
  +── SCHEMA
       "Which namespace?"

DATABASE.SCHEMA.TABLE
        ↓
      DATA
```

> **Core idea to remember:**\
> **Role controls access. Warehouse provides compute. Database and
> schema provide the namespace. Tables contain the data. The session
> context tells Snowflake which role, warehouse, database, and schema
> are currently active.**

------------------------------------------------------------------------

## 🚀 Next Module

The natural next step is:

**Module 2 --- Loading Structured Data**

You will learn:

``` text
Local File
    ↓
PUT
    ↓
Internal Stage
    ↓
File Format
    ↓
COPY INTO
    ↓
Snowflake Table
```

And we'll cover the gaps too:

-   Internal vs. external stages
-   Named stages
-   User/table stages
-   File formats
-   CSV options
-   `PUT`
-   `COPY INTO`
-   `LIST`
-   `VALIDATION_MODE`
-   Error handling
-   Load history
-   Duplicate file handling
-   `ON_ERROR`
-   Structured vs. semi-structured data
-   Hands-on loading lab
-   Interview questions
-   Practice challenge
