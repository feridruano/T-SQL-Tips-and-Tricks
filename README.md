# T-SQL Tips and Tricks
A repository of useful T-SQL Tips and Tricks. Includes basic generalized queries that I'd rather write down than remember while being searchable.

## Get First or Last Record Using NOT EXISTS()

### Example 1: Get the first order sold by sales person.

```sql
SELECT *
FROM dbo.Orders
WHERE NOT EXISTS(SELECT *
                 FROM dbo.Orders T
                 WHERE T.SalesPerson = Orders.SalesPerson
                   AND T.OrderDate < Orders.OrderDate) -- First Record For Each Sales Person
```

### Example 2: For each year, find the order # that cumulatively reached over $10 million revenue. 

```sql
WITH CTE AS (
    SELECT DISTINCT
           YEAR(OrderDate)                                                      FiscalYear
         , OrderDate
         , SUM(SubTotal) OVER (PARTITION BY YEAR(OrderDate) ORDER BY OrderDate) RunningTotal
         , DENSE_RANK() OVER (PARTITION BY YEAR(OrderDate) ORDER BY OrderDate)  OrderNumber
    FROM Sales.SalesOrderHeader
)
SELECT *
FROM CTE
WHERE RunningTotal >= 10000000 -- Outer Query Predicate - Requires SAME Dataset as Inner Query
  AND NOT EXISTS(SELECT *
                 FROM CTE T
                 WHERE T.FiscalYear = CTE.FiscalYear
                   AND T.OrderDate < CTE.OrderDate -- First Record Over $10 Million
                   AND T.RunningTotal >= 10000000) -- Inner Query Predicate
```

#### How It Works

- At the moment, I need a better lyman's explanation of how `NOT EXISTS()` is used here because I forget how it works trying to figure it out each time.

### Tips

- Supported on all SQL Server versions.
- Remember, <u>data</u> within the record pertains to the <u>First or Last Record</u>.
- Additional predicates (filters) can be included. However, those predicates must be placed on both the inner and outer `WHERE` clause inorder to maintain the ***same dataset*** and retrieve the **<u>correct</u>** first or last record.
- `JOIN`s are allowed but can increase dataset complexity. Although, they'll help you figure out the limitations and possibilities here.
- `NOT EXISTS()` is allowed in `ON` statements for `JOIN`s. For example, joining the first or last record only to another dataset.



## Get Maximum or Minimum Value for 3+ Columns

### Example 1: Get the highest of three test scores using inline subquery.

```sql
SELECT StudentID
     , TestScore1 -- 1st Column
     , TestScore2 -- 2nd Column
     , TestScore3 -- 3rd Column
     , (SELECT MAX(TestScores) -- or MIN(), AVG(), Etc.
    FROM (VALUES (TestScore1)
               , (TestScore2)
               , (TestScore3) -- Add More Columns Here
         ) A(TestScores) -- Temporary Table Resulting in One 'TestScores' Column to Aggregate
) HighestTestScore -- Subquery Result Column Name
FROM dbo.TestScores
```

### Example 2: Get highest of three test scores using CROSS APPLY (Multiple Aggregates)

```sql
SELECT MIN(TestScores) AS LowestTestScore
     , MAX(TestScores) AS HighestTestScore
FROM dbo.TestScores
         CROSS APPLY (VALUES (TestScore1)
                           , (TestScore2)
                           , (TestScore3) -- Add More Columns Here
                      ) AS A (TestScores); -- Dataset Column Resulting in One 'TestScores'
```

The examples above both transpose multiple rows into a singular column to be aggregated using any of aggregate functions.

### Tips

- Supported on all SQL Server versions. For ***SQL Server 2022*** and above use `GREATEST()` or `LEASTEST()` instead.
- Subquery can only ever return one aggregate result. `CROSS APPLY` changes the dataset which allows multiple aggregates, but remember the dataset becomes modified!
- Most issues will be results of missing or extra parenthesis, specifically when writting out the `VALUES()`.
- Subquery starting structure necessary: `FROM (VALUES ...) A(COLUMN_NAME)`





## SQL Server Version Differences

### SQL Server 2022

- `Greatest()` and `Least()`
- `STIRNG_SPLIT()` - Improved with **Ordinal** determinism
- `GENERATE_SERIES()` - Create a table with a series (1-nth number).
- Parameter Sniffing - Improved, now caches multiple plans and switches between them.
- Ignore Nulls - `FIRST_VALUE()`, `LAST_VALUE()`, `LAG()`, and `LEAD()`.

### SQL Server 2019

### Features

- Scalar UDF Inlining - 
- `APPROX_COUNT_DISTINCT()`
- TempDB - Improved using memory-optimization
- Verbose Truncation Warning - Big int to smaller int

### SQL Server 2017

#### Features

- `CONCAT_WS()` - Concatenation with separator
- `TRANSLATE()` - Replace character by character regardless of order in string
- `TRIM()` - Remove left and right whitespace
- `STRING_ARG()` - Concatenate column strings together
- Database Tuning - Improve over all performance
- `SELECT INTO` - Now allows non-default file groups

### SQL Server 2016

#### Features

- Live Query Statistics - See execution plan before the query finishes! No more waiting for long query plans.
- Dynamic Data Masking - Mask data for specific users.
- `DROP IF EXISTS` - As it's spoken
- `FORMATMESSAGE()` - Supply custom strings, part of dynamic SQL
- `DATEDIFF_BIG()` - More precise date part
- `String_Split()`
- `AT TIME ZONE` - Converts between timezones
- `CURRENT_TRANSACTION_ID()` - Returns the current transaction ID.
- JSON support

## Look Into This

- Data Files and Filegroups
- In-Memory OLTP
- TempDB
