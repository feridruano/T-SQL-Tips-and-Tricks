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

- At the moment, I need a better lyman's explanation of how `NOT EXISTS()` is used here because I forget how it works trying to figure it out each time.
- Only the record with the earliest `OrderDate` passes the `NOT EXISTS()`.

### Tips

- Supported on all SQL Server versions.
- Remember, <u>data</u> within the record pertains to the <u>First or Last Record</u>.
- Additional predicates (filters) can be included. However, those predicates must be placed on both the inner and outer `WHERE` clause inorder to maintain the **<u>same dataset</u>** and retrieve the **<u>correct</u>** first or last record.
- `JOIN`s are allowed but can increase dataset complexity and **<u>may</u>** need to be apart of both the outer query and inner `NOT EXISTS()` query. Practicing with `JOIN`s is useful in understanding the limitations and possibilities of `NOT EXISTS()`.
- `NOT EXISTS()` is allowed in `ON` statements for `JOIN`s. For example, joining the first or last record of one dataset to another dataset.

</br>

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

</br>

## Concatenating Multiple Strings Within a Single Column (Row Concatenation)

```sql
SELECT 
  SampleUserNumber, 
  STUFF((SELECT N' ' + SampleTextLine FROM #SampleTable
       WHERE SampleUserNumber = t.SampleUserNumber
       ORDER BY SampleLineNumber
       FOR XML PATH, TYPE).value(N'.[1]',N'nvarchar(max)'),1,1,'')
FROM #SampleTable AS t
GROUP BY SampleUserNumber
ORDER BY SampleUserNumber
```

To insert a separator between each concatenation you can specify a one in the `N' '` of `SELECT N' ' + SampleTextLine`. The default is a empty space.

**Examples:**  

`SELECT N'/' + SampleTextLine` results in `user1/user2/user3`.

`SELECT N',' + SampleTextLine` results in `user1,user2,user3`.

Essentially, the subquery creates a temporary dataset with a singular column where every value is appended with a ***given separator*** and ordered by a ***stated order***. The `XML PATH` loops through the modified and ordered column values top-down while concatenating each value into an **XML String**. The `.value()` converts the **XML String** to an **NVARCHAR(MAX) String**. Finally, `STUFF()` removes the first character which is a separator.

**Link:** [StackExchange](https://dba.stackexchange.com/questions/207371/please-explain-what-does-for-xml-path-type-value-nvarcharmax)

### Tips

- Supported on all SQL Server versions. For ***SQL Server 2017*** and above use `STRING_AGG()` instead.
- T-SQL requires`.value()` to be **<u>lowercase</u>**, Otherwise an error is thrown.
- `.value()` is **<u>not</u>** the same as `VALUES`.
- For distinct value concatenation, add a `GROUP BY` within the subquery or create an nested subquery to prepare data.

</br>

## SQL Server Version Differences

### SQL Server 2022

- `GREATEST()` and `LEAST()`
- `STIRNG_SPLIT()` - **Ordinal** is now deterministic.
- `GENERATE_SERIES()` - Create a table with a series (1-nth number).
- `IGNORE NULLS` - `FIRST_VALUE()`, `LAST_VALUE()`, `LAG()`, and `LEAD()`.
- `WINDOW` Clause - Simplify multiple of the same `OVER()` window function.
- Parameter Sniffing - Improved, now caches multiple plans and switches between them.

### SQL Server 2019

### Features

- Scalar UDF Inlining
- `APPROX_COUNT_DISTINCT()`
- TempDB - Improved using memory-optimization.
- Verbose Truncation Warning - Big int to smaller int warning.

### SQL Server 2017

#### Features

- `CONCAT_WS()` - Concatenation with separator
- `TRANSLATE()` - Replace character by character regardless of order in string
- `TRIM()` - Remove left and right whitespace
- `STRING_ARG()` - Concatenate column strings together
- `SELECT INTO` - Now allows non-default file groups
- Database Tuning - Improve over all performance

### SQL Server 2016

#### Features

- Live Query Statistics - See execution plan before the query finishes! No more waiting for long query plans.
- Dynamic Data Masking - Mask data for specific users.
- `DROP IF EXISTS`
- `FORMATMESSAGE()` - Supply custom strings, part of dynamic SQL.
- `DATEDIFF_BIG()` - More precise date part.
- `STRING_SPLIT()` - Split strings by separator. Non-deterministic ordinal.
- `AT TIME ZONE` - Converts between timezones.
- `CURRENT_TRANSACTION_ID()` - Returns the current transaction ID.
- JSON Support

</br>

## Look Into This

- Data Files and Filegroups
- In-Memory OLTP
- TempDB
