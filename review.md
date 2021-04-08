# Brief review for SQL (particularly the T-SQL and PL/SQL flavors)

## Introduction

In past 2 years, particularly during the months I was unemployed, I have realized that the hiring process for data analyst role is becoming increasingly technical. There are mainly two ways to test the applicant's technical ability that I have encountered:

1. **Assignment/Testing**: Applicant is assigned a problem, which can either be done at home and submitted before a given deadline, or done on-site. The former is more common. Applicant can choose whichever tools they are comfortable with, as long as they can produce result. Methodology and domain knowledge are often valued more.

2. **Technical questions** given during interviews, which cover various topics: probability, statistics, SQL, Python, R, etc. You may be required to write your code on paper.

This review is mainly beneficial for the 2nd method of testing, which is using technical questions, particularly SQL questions. I admit that I lack a formal education in database management/ETL, and thus heavily, and also willingly depend on Google and code completion. As such writing SQL on paper (not to mention in front of other people) can be scary. 

---
## Some notes on joining tables

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
select t1.*, t2.RFR as RFR_LM
from table t1, table t2
where t1.Due_Date = dateadd(month, -1, t2.DUE_DATE)
```

In this query I use the **Due_Date** column as the key to join. Current date is matched with the same date from last month (see the `DATEADD` function) to return the roll-forawrd rate from last month. A new column called **RFR_LM** (roll forward rate last month) is created, which can then be compared with the already-existed **RFR** column.

## Some problems related to window functions

### Count the number of time a value changes

| allocation_month | agency | count_change |
|------------------|--------|--------------|
| 2021-01          | A      | 1            |
| 2021-02          | B      | 2            |
| 2021-03          | C      | 3            |
| 2021-04          | C      | 3            |
| 2021-05          | C      | 3            |
| 2021-06          | B      | 4            |
| 2021-07          | C      | 5            |
| 2021-08          | D      | 6            |
| 2021-09          | D      | 6            |

Supposely we are given the first 2 columns, which show the log of agencies that a contract is allocated to for debt collection, and has to create the third one called **no_of_change** which indicates the number of time the values changes over time (**day**).

This task requires the use of `DENSE_RANK()`, `ROW_NUMBER()` and `MIN()` as follows:

1. Let's create a table with the 2 first columns:

```sql
drop table if exists #value_change
create Table #value_change
( 
allocation_month varchar(8) null ,
agency varchar(1) null
)

-- Inserting Records
insert into #value_change values('2021-01','A')
insert into #value_change values('2021-02','B')
insert into #value_change values('2021-03','C')
insert into #value_change values('2021-04','C')
insert into #value_change values('2021-05','C')
insert into #value_change values('2021-06','B')
insert into #value_change values('2021-07','C')
insert into #value_change values('2021-08','D')
insert into #value_change values('2021-09','D')
```

2. There are several things we have to be aware of in this problem: 
    
    - First ocurrance of any agency is counted as 1.
    - We have to differentiate streaks of occurance of an agency. For example, C was allocated consecutively 2 times, the first time from 2021-02 to 2021-04 and the second time in 2021-06. During the first time, the count remains the same at 3, and in the second time, the count changes, even though the contract goes to an agency that was allocated to before.
    - From the above observation, we can try isolate each continuous streak of an agency with order, then use `DENSE_RANK()` to rank them. Note that we can't use `RANK()` as the ranking wouldn't be consecutive.

3. First, let's try using `ROW_NUMBER()` with and without partitions of __agency__ to see what's happening:

```sql
select *, rn1-rn2 as gr1
from (
	select *,
		row_number() over(order by allocation_month) as rn1,
		row_number() over(partition by agency order by allocation_month) as rn2
	from #value_change
	) x
```

Output:

| allocation_month | agency | rn1 | rn2 | gr1 |
|------------------|--------|-----|-----|-----|
| 2021-00          | A      | 1   | 1   | 0   |
| 2021-01          | B      | 2   | 1   | 1   |
| 2021-05          | B      | 6   | 2   | 4   |
| 2021-02          | C      | 3   | 1   | 2   |
| 2021-03          | C      | 4   | 2   | 2   |
| 2021-04          | C      | 5   | 3   | 2   |
| 2021-06          | C      | 7   | 4   | 3   |
| 2021-07          | D      | 8   | 1   | 7   |
| 2021-08          | D      | 9   | 2   | 7   |

We can see that if we take the difference between __rn1__ and __rn2__ (which we would name __gr1__), the result would include groups of agencies based on consecutive allocation, however the order is wrong compared with __allocation_month__. For example, in __allocation_month__ = '2021-06', the difference is 3, which is less than the immediate value before it (which is 4). 

4. To handle this, we can take the earliest __allocation_month__ of each group in __gr1__ by using `MIN()`, so that we can get the correct order. Also, for now let's not worry about formatting too much; we will simply nest the queries and show every newly calculated columns so that we can see the logic:

```sql
select *, min(allocation_month) over(partition by agency, gr1) as gr2
from (
	select *, rn1-rn2 as gr1
	from (
		select *,
			row_number() over(order by allocation_month) as rn1,
			row_number() over(partition by agency order by allocation_month) as rn2
		from #value_change
		) x
	) y
order by allocation_month
```

Output:

| allocation_month | agency | rn1 | rn2 | gr1 | gr2     |
|------------------|--------|-----|-----|-----|---------|
| 2021-00          | A      | 1   | 1   | 0   | 2021-00 |
| 2021-01          | B      | 2   | 1   | 1   | 2021-01 |
| 2021-05          | B      | 6   | 2   | 4   | 2021-05 |
| 2021-02          | C      | 3   | 1   | 2   | 2021-02 |
| 2021-03          | C      | 4   | 2   | 2   | 2021-02 |
| 2021-04          | C      | 5   | 3   | 2   | 2021-02 |
| 2021-06          | C      | 7   | 4   | 3   | 2021-06 |
| 2021-07          | D      | 8   | 1   | 7   | 2021-07 |
| 2021-08          | D      | 9   | 2   | 7   | 2021-07 |

5. Now that we have been able to group the consecutive streaks of agencies in the correct order, we can use `DENSE_RANK()` on the __gr2__ column to count the number of time the allocated agency is changed:

```sql
select *, dense_rank() over(order by gr2) as gr3
from ( 
	select *, min(allocation_month) over(partition by agency, gr1) as gr2
	from (
		select *, rn1-rn2 as gr1
		from (
			select *,
				row_number() over(order by allocation_month) as rn1,
				row_number() over(partition by agency order by allocation_month) as rn2
			from #value_change
			) x
		) y
	) z
order by allocation_month
```

Output:

| allocation_month | agency | rn1 | rn2 | gr1 | gr2     | gr3 |
|------------------|--------|-----|-----|-----|---------|-----|
| 2021-00          | A      | 1   | 1   | 0   | 2021-00 | 1   |
| 2021-01          | B      | 2   | 1   | 1   | 2021-01 | 2   |
| 2021-02          | C      | 3   | 1   | 2   | 2021-02 | 3   |
| 2021-03          | C      | 4   | 2   | 2   | 2021-02 | 3   |
| 2021-04          | C      | 5   | 3   | 2   | 2021-02 | 3   |
| 2021-05          | B      | 6   | 2   | 4   | 2021-05 | 4   |
| 2021-06          | C      | 7   | 4   | 3   | 2021-06 | 5   |
| 2021-07          | D      | 8   | 1   | 7   | 2021-07 | 6   |
| 2021-08          | D      | 9   | 2   | 7   | 2021-07 | 6   |

Now that we've gotten the final result in __gr3__ column, let's rewrite the query so that it looks neater:

```sql
select 
	allocation_month,
	agency, 
	dense_rank() over(order by min_allocation_month) as count_change
from ( 
	select 
		allocation_month,
		agency, 
		min(allocation_month) over(partition by agency, gr) as min_allocation_month
	from (
		select *,
			row_number() over(order by allocation_month) - row_number() over(partition by agency order by allocation_month) as gr
	from #value_change
		) x
	) y
order by allocation_month
```

### Cumulative sums

Supposedly we have a table called __transactions__ with transaction dates along with the total transaction value of that date meaning that the transaction dates are unique).

| transaction_date | daily_revenue |
|------------------|---------------|
| 2021-04-01       | 1000          |
| 2021-04-02       | 2000          |
| 2021-04-03       | 1500          |
| 2021-04-04       | 500           |
| 2021-04-05       | 800           |
| 2021-04-06       | 1300          |
| 2021-04-07       | -2000         |
| 2021-04-08       | 100           |
| 2021-04-09       | 5000          |

The task given is to create a 3rd column with cumulative revenue of April 2021. This can be accomplished in 2 ways:

__Solution 1__: Using windows function:

```sql
SELECT 
    transaction_date, 
    SUM(daily_revenue) OVER (ORDER BY transaction_date ASC) as mtd_revenue
FROM
    transactions 
ORDER BY 
    date ASC
```

Note that is is the preferred solution, as it's more efficient and shorter.

__Solution 2__: Using join:

```sql
SELECT 
    a.transaction_date, 
    SUM(b.daily_revenue) as mtd_revenue
FROM
    transactions a
JOIN transactions b 
ON a.date >= b.date 
GROUP BY 
    a.transaction_date 
ORDER BY 
    a.transaction_date
```

Note that in the above solution, the condition for joining in the `ON` clause uses _equal or greater than_ comparison, as not only _equal_ comparison is legal when joining tables.

## Other interesting problems

### Find best previous performing month and fetch the same day as today from that month

Recently I was assigned the following task: The performance report at my company (that I created and maintain) used to show comparison of current month and the previous month. However, as certain months are affected by holidays, it makes more sense to compare the current month with the best performing month in the past. The task can be broken down into 2 parts:

1. Find the best previous performing month, e.g. December 2020, and compare with the current month, e.g. April 2021
2. Besides total monthly numbers, we also show daily numbers. This mean that we need to fetch data for December 7, 2021, assuming that the current reporting date is April 7, 2021, and compare those 2 dates.

Assuming the __report__ table is as below:

| contract_id | report_date | daily_payment | accumulated_payment |
|-------------|-------------|---------------|---------------------|
| 1           | 20201001    | 1000          | 1000                |
| 2           | 20201001    | 2000          | 2000                |
| ...         | ...         |               | ...                 |
| 1           | 20201002    | NULL          | 1000                |
| 2           | 20201002    | NULL          | 2000                |
| ...         | ...         |               | ...                 |
| 1           | 20201003    | 2103          | 3103                |
| 2           | 20201003    | 2301          | 4301                |
| ...         | ...         |               | ...                 |
| 1           | 20201004    | 2212          | 5315                |
| 2           | 20201004    | 3465          | 7766                |

Notice that the table has a few quirks:

-  The __report_date__ column contains strings representing date in "YYYYMMDD" format, instead of actual dates.
-  There are a fixed number of __contract_id__ repeating through each day, and if there's no transaction for a particular contract on a particular day, the payment amount would be 0.

And here's the solution:

```sql
DECLARE @max_report_date varchar(8) = (select max(report_date) from report)
DECLARE @max_report_date_bom varchar(8) = left(@max_report_date, 6) + '01'
DECLARE @max_value_date varchar(8) = (
			select top 1 report_date
			from (
				select 
					report_date,
					isnull(sum(accumulated_payment),0) as payment_amt
				from report
				where Report_date in (
								select left(report_date, 4) + substring(report_date, 5, 2) + right('0'+ cast(max(cast(right(report_date, 2) as int)) as varchar(max)), 2)
								from report
								where left(report_date, 6) != left(@max_report_date, 6)
								group by left(report_date, 4), substring(report_date, 5, 2)
										)
				and mtd_pmt is not null
				group by report_date
				) x
			order by payment_amt DESC)
DECLARE @max_value_date_bom varchar(8) = left(@max_value_date, 6) + '01'
DECLARE @MONTH_DIFF int = DATEDIFF(MONTH, convert(date, @max_value_date_bom, 112), convert(date, @max_report_date_bom, 112))
DECLARE @compare_date varchar(8) = (select convert(varchar, dateadd(m, @MONTH_DIFF*(-1), convert(date, @max_report_date, 112)), 112))

select @compare_date
```

The steps in the above query can be explained as follows:

1. Find the latest reporting date in the __report__ table (`max_report_date`)
2. Get the date at the beginning of the reporting month (BOM) (`max_report_date_bom`)
3. Find the month with the highest end-of-month total revenue; the result would be the last day of that month (because the last day would have the end-of-month total revenue)
4. Find the BOM date of the best performing previous month
5. Calculate the difference in months between the BOM of the current reporting month and the BOM of the best performing previous month
6. Substract the difference in months as calculated above from the last reporting date to get the corresponding date from the best performing previous month

## Miscellaneous stuff

### `<>` vs. `!=`

In SQL Server, we can use `<>` and `!=` interchangably. However, `<>` is defined in the [ANSI 99 SQL standard](http://web.cecs.pdx.edu/~len/sql1999.pdf) and `!=` is not. So not all DB engines may support it and if we want to generate portable code it's recommended to use `<>`.

---

To be continued.
