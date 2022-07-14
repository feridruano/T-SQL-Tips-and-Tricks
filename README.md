# T-SQL Tips and Tricks
A repository of useful T-SQL Tips and Tricks. Includes basic generalized queries that I'd rather write down than remember while being searchable.

## Tricks

### Get First or Last Record Using NOT EXISTS()

#### Example 1: Get the first order sold by sales person.

```sql
SELECT *
FROM dbo.Orders
WHERE NOT EXISTS(SELECT *
                 FROM dbo.Orders T
                 WHERE T.SalesPerson = Orders.SalesPerson
                   AND T.OrderDate < Orders.OrderDate) -- First Record For Each Sales Person
```

#### Example 2: For each year, find the order # that cumulatively reached over $10 million revenue. 

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

#### Tips

- Supported on all SQL Server versions.
- Remember, <u>data</u> within the record pertains to the <u>First or Last Record</u>.
- Additional predicates (filters) can be included. However, those predicates must be placed on both the inner and outer `WHERE` clause inorder to maintain the ***same dataset*** and retrieve the **<u>correct</u>** first or last record.
- `JOIN`s are allowed but can increase dataset complexity. Although, they'll help you figure out the limitations and possibilities here.
- `NOT EXISTS()` is allowed in `ON` statements for `JOIN`s. For example, joining the first or last record only to another dataset.
