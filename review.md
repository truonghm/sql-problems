# Brief review for SQL (particularly the T-SQL and PL/SQL flavors)

## Introduction

In past 2 years, particularly during the months I have been unemployed, I have realized that the hiring process for data analyst role is becoming increasingly technical. There are mainly two ways to test the applicant technical ability that I have encountered:

1. **Assignment/Testing**: Applicant is assigned a data analysis problem, which can either be done at home and submit before given deadline, or done on-site. The former is more common. Applicant can choose whichever technologies they are comfortable with, as long as they can produce result. Methodology and domain knowledge are often valued more.

2. **Technical questions** given during interviews, which cover various topics: probability, statistics, SQL, Python, R, etc. You can often code on paper.

This review is mainly about the 2nd method: technical questions, particularly SQL questions. I admit that I lack a formal education in database management/ETL, and thus heavily, and also willingly depend on Google and code completion instead of memorization. As such SQL questions can be scary. 

I'm writing this review with the goal of becoming more proficient in SQL, at a level where I understand how SQL and databases work under the hood and begin to approach optimization, instead of just focusing on making queries work without error.

https://stackoverflow.com/questions/354070/sql-join-where-clause-vs-on-clause

## Interesting notes about joining tables

the usual way of joining tables is:

```sql
select table1.col1, table2.col2
from table1
left join table2
on table1.key1 = table2.key2
```

Notice that the `on` clause specifies which columns on both table to match with, hence the `=` operator. This is the standward way to join.
However, we can also it like this:

```sql
select table1.col1, table2.col2
from table1
left join table2
on table1.key1 = table2.key2
and table1.cond1 = 'x'
and table2.cond2 > 100
```

The above query not only specifes which columns are use to match, but also which **rows** will be joined: in table1, only rows where cond1 = 'x' are joined, and in table2, only rows where cond2 > 100 are joined.

Note that this is different from using the `WHERE` clause:

```sql


```

