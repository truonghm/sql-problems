# Brief review for SQL (particularly the T-SQL and PL/SQL flavors)

## Introduction

In past 2 years, particularly during the months I was unemployed, I have realized that the hiring process for data analyst role is becoming increasingly technical. There are mainly two ways to test the applicant technical ability that I have encountered:

1. **Assignment/Testing**: Applicant is assigned a data analysis problem, which can either be done at home and submitted before a given deadline, or done on-site. The former is more common. Applicant can choose whichever technologies they are comfortable with, as long as they can produce result. Methodology and domain knowledge are often valued more.

2. **Technical questions** given during interviews, which cover various topics: probability, statistics, SQL, Python, R, etc. You may be required to write your code on paper.

This review is mainly about the 2nd method: technical questions, particularly SQL questions. I admit that I lack a formal education in database management/ETL, and thus heavily, and also willingly depend on Google and code completion instead of memorization. As such SQL questions can be scary. 

I'm writing this review with the goal of becoming more proficient in SQL, at a level where I understand how SQL and databases work under the hood and begin to approach optimization, instead of just focusing on making queries work without error.

---
## Interesting notes about joining tables

### Specifying conditions when joining
the usual way of joining tables is:

```sql
select *
from table1
left join table2
on table1.key1 = table2.key2
```

Notice that the `on` clause specifies which columns on both table to match with, hence the `=` operator. This is the standward way to join.
However, we can also do it like this:

```sql
select *
from table1
left join table2
on table1.key1 = table2.key2
    and table1.cond1 = 'x'
    and table2.cond2 > 100
```

The above query specifies not only which columns are used as key to join, but also which **rows** will be joined: in table1, only rows where cond1 = 'x' are joined, and in table2, only rows where cond2 > 100 are joined.

Note that this is different from using the `WHERE` clause:

```sql
select *
from table1
left join table2
on table1.key1 = table2.key2
where table1.cond1 = 'x'
    and table2.cond2 > 100
```

The first query returns every rows from table1, but only rows that satisfy cond1 and cond2 are joined with table2.

The second query joins table1 with table2 on every rows of table1, but only return rows that match the conditions in the `WHERE` clause.


### Self-join

Self-join is basic but often forgotten (by me!). It's more compact and faster than joining 2 select clauses from the same table:

```sql
select column_name(s)
from table1 T1, table1 T2
where condition
```

This is particularly useful when you want to compare e.g. rates from a previous period:

```sql
select t1.*, t2.[RFR] as [RFR_LM]
from table t1, table t2
where t1.Due_Date = convert(varchar, dateadd(month, -1, CONVERT(DATETIME,t2.DUE_DATE,112)), 112)
```

In this query I use the **Due_Date** column as the key to join. Current date is matched with the same date from last month (see the `DATEADD` function) to return the roll-forawrd rate from last month. A new column called **RFR_LM** (roll forward rate last month) is created, which can then be compared with the already-existed **RFR** column.

## Some problems related to window functions

### Count the number of time a value changes

| allocation_month | agency | no_of_change |
|------------------|--------|--------------|
| 2021-01          | A      | 1            |
| 2021-02          | B      | 2            |
| 2021-03          | C      | 3            |
| 2021-04          | C      | 3            |
| 2021-05          | B      | 4            |
| 2021-06          | D      | 5            |
| 2021-07          | D      | 5            |

Supposely we are given the first 2 columns, which show the log of agencies that a contract is allocated to for debt collection, and has to create the third one called **no_of_change** which indicates the number of time the values changes over time (**day**).

This task requires the use of `DENSE_RANK()`, `ROW_NUMBER()` and `MIN()` as follows:

- First, let's create a table with the 2 first columns:

```sql
create Table #value_change
( 
allocation_month varchar(8) null ,
agency varchar(1) null
)

-- Inserting Records
insert into #value_change values('2021-00','A')
insert into #value_change values('2021-01','B')
insert into #value_change values('2021-02','C')
insert into #value_change values('2021-03','C')
insert into #value_change values('2021-04','B')
insert into #value_change values('2021-05','D')
insert into #value_change values('2021-06','D')
```

- There are several things we have to be aware of in this problem: 
    
    1. First ocurrance of any agency is counted as 1.
    2. We have to differentiate streaks of occurance of an agency. For example, C was allocated consecutively 2 times, the first time from 2021-02 to 2021-04 and the second time in 2021-06. During the first time, the count remains the same at 3, and in the second time, the count changes, even though the contract goes to an agency that was allocated to before.




## CTEs

From Microsoft's documentation, CTEs can be used to:

> - Create a recursive query. For more information, see [Recursive Queries Using Common Table Expressions](https://docs.microsoft.com/en-us/previous-versions/sql/sql-server-2008-r2/ms186243(v=sql.105)).
> - Substitute for a view when the general use of a view is not required; that is, you do not have to store the definition in metadata.
> - Enable grouping by a column that is derived from a scalar subselect, or a function that is either not deterministic or has external access.
> - Reference the resulting table multiple times in the same statement.


