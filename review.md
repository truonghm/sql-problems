# Brief review for SQL (particularly the T-SQL and PL/SQL flavors)

## Introduction

In past 2 years, particularly during the months I have been unemployed, I have realized that the hiring process for data analyst role is becoming increasingly technical. There are mainly two ways to test the applicant technical ability that I have encountered:

1. **Assignment/Testing**: Applicant is assigned a data analysis problem, which can either be done at home and submit before given deadline, or done on-site. The former is more common. Applicant can choose whichever technologies they are comfortable with, as long as they can produce result. Methodology and domain knowledge are often valued more.

2. **Technical questions** given during interviews, which cover various topics: probability, statistics, SQL, Python, R, etc. You can often code on paper.

This review is mainly about the 2nd method: technical questions, particularly SQL questions. I admit that I lack a formal education in database management/ETL, and thus heavily, and also willingly depend on Google and code completion instead of memorization. As such SQL questions can be scary. 

I'm writing this review with the goal of becoming more proficient in SQL, at a level where I understand how SQL and databases work under the hood and begin to approach optimization, instead of just focusing on making queries work without error.

https://stackoverflow.com/questions/354070/sql-join-where-clause-vs-on-clause

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
However, we can also it like this:

```sql
select *
from table1
left join table2
on table1.key1 = table2.key2
    and table1.cond1 = 'x'
    and table2.cond2 > 100
```

The above query not only specifes which columns are use to match, but also which **rows** will be joined: in table1, only rows where cond1 = 'x' are joined, and in table2, only rows where cond2 > 100 are joined.

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
select t1.*, t2.[%RFR] as [%RFR_LM]
from table t1, table t2
where t1.Due_Date = convert(varchar, dateadd(month, -1, CONVERT(DATETIME,t2.DUE_DATE,112)), 112)
```

In this query I use the **Due_Date** column as the key to join. Current date is matched with the same date from last month (see the `DATEADD` function) to return the roll-forawrd rate from last month. A new column called **%RFR_LM** (roll forward rate last month) is created, which can then be compared with the already-existed **%RFR** column.

### UPDATE & JOIN


