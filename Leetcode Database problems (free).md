Leetcode Database problems (free)

## Combine two tables (Easy)

Runtime: 970 ms (better than 64.42% of mssql submissions)

```sql
select p.FirstName, p.LastName, a.City, a.State
from Person p
left join Address a
on p.PersonId = a.PersonId
```

## Second highest salary (Easy)

Runtime: 812 ms (better than 77.10% of mssql submissions)

```sql
select max(salary) as SecondHighestSalary
from (SELECT
    ROW_NUMBER() OVER (ORDER BY Salary DESC) AS rownumber,
    Salary
  FROM (select distinct salary
        from Employee) o
) AS f
where rownumber = 2
```

## Nth highest salary (Medium)

Runtime: 1007 ms (better than 52.89% of mssql submissions)

```sql
CREATE FUNCTION getNthHighestSalary(@N INT) RETURNS INT AS
BEGIN
    RETURN (
        /* Write your T-SQL query statement below. */
        select max(salary) as SecondHighestSalary
        from (SELECT
            ROW_NUMBER() OVER (ORDER BY Salary DESC) AS rownumber,
            Salary
          FROM (select distinct salary
                from Employee) o
        ) AS f
        where rownumber = @N
    );
END
```

## Rank scores (Medium)

Runtime: 1333 ms (better than 22.59% of mssql submissions)

```sql
select score, DENSE_RANK() over(order by score desc) as rank
from Scores
```

## Department Top Three Salaries (Hard)

Runtime: 1273 ms (better than 39.42% of mssql submissions)

```sql
select Department, Employee, Salary
from (
    select d.Name as Department, e.Name as Employee, e.Salary,
            dense_rank() over(partition by e.DepartmentId order by e.Salary DESC) r
    from employee e
    inner join Department d
    on e.DepartmentId = d.Id) a
where r < 4
```

## Consecutive Numbers (Medium)

Runtime: 1177 ms (beats 63.47 % of mssql submissions)

```sql
select distinct Num as ConsecutiveNums
from (
    select Num, lead(Num, 1) over(order by Id) as l1, lead(Num, 2) over(order by Id) as l2
    from Logs) a
where Num = l1
and l1 = l2
```

## Trips and Users (Hard)

Runtime: 858 ms (beats 42.19 % of mssql submissions)

```sql
select Day, 
    case
    when count_cancel = 0 then 0
    when count_cancel = count_total then 1.00
    else cast(count_cancel/count_total as decimal(18,2)) end as [Cancellation Rate]
from (
select cast(Request_at as Date) as Day,
        cast(sum(case 
        when Status like 'cancelled_by_%' 
        then 1 else 0 end) as float) count_cancel,
        cast(count(*) as float) as count_total
from Trips t
inner join Users u1
on t.Client_Id = u1.Users_Id
and u1.Banned = 'No'
inner join Users u2
on t.Client_Id = u2.Users_Id
and u2.Banned = 'No'
group by Request_at
) a
where Day between '2013-10-01' and '2013-10-03'
```

## Employees Earning More Than Their Managers (Easy)

Runtime: 1327 ms (beats 39.27 % of mssql submissions)

```sql
select e1.Name as Employee
from Employee e1, Employee e2
where e1.ManagerId = e2.Id
and e1.Salary > e2.Salary
```

## Duplicate Emails (Easy)

Runtime: 982 ms (beats 63.67 % of mssql submissions)

```sql
select distinct Email
from Person
group by Email
having count(Email) > 1
```

## Customers Who Never Order (Easy)

Runtime: 831 ms (beats 72.08 % of mssql submissions)

```sql
select Name as Customers
from Customers c
left join Orders o
on c.Id = o.CustomerId
where o.Id is null
```

## Department Highest Salary (Medium)

Runtime: 1159 ms (beats 45.10 % of mssql submissions)

```sql
select Department, Employee, Salary
from (
    select d.Name as Department, e.Name as Employee, e.Salary,
            dense_rank() over(partition by e.DepartmentId order by e.Salary DESC) r
    from employee e
    inner join Department d
    on e.DepartmentId = d.Id) a
where r = 1
```

## Rising Temperature (Easy)
 
Runtime: 1398 ms (beats 32.56 % of mssql submissions)

```sql
select id
from (
select id,
    datediff(day, lag(recordDate, 1) over(order by recordDate), recordDate) as yesterday,
    Temperature - lag(Temperature, 1) over(order by recordDate) as dif
from Weather) a
where yesterday = 1
and dif > 0
```

## Big Countries (Easy)

Runtime: 1129 ms, faster than 84.93% of MS SQL Server online submissions

```sql
select [name], population, area
from world
where area > 3000000
or population > 25000000
```

## Classes More Than 5 Students (Easy)

```sql
select class
from courses
group by class
having count(distinct student) >= 5
```

## Not Boring Movies (Easy)

Solution 1 using modulus:

```sql
select id, movie, description, rating
from cinema
where id % 2 = 1
and description <> 'boring'
order by rating desc
```

Solution 2 using bitwise operation:

```sql
select *
from cinema
where id & 1 = 1
and description <> 'boring'
order by rating desc
```

## Human Traffic of Stadium (Hard)

Solution 1, nested select:

```sql
select id, visit_date, people
from (

select *,
    count(*) over(partition by gr) as cgr 
from (
select *,
    id - row_number() over(order by id) as gr
from stadium
where people >= 100
) a
) b
where cgr >= 3
```

Solution 2 using CTEs:

```sql
select id, visit_date, people
from (

select *,
    count(*) over(partition by gr) as cgr 
from (
select *,
    id - row_number() over(order by id) as gr
from stadium
where people >= 100
) a
) b
where cgr >= 3
```

## Exchange Seats (Easy)

```sql
select id, 
    case when id & 1 = 1 then 
        case when id <> (select max(id) from seat) then lead(student, 1) over(order by id)
        else student
        end
    else lag(student, 1) over(order by id) 
    end as student
from seat
```

## Swap Salary (Easy)

Requirement: using a single update statement.

```sql
update salary
set sex = case
when sex = 'm' then 'f'
when sex = 'f' then 'm'
end
```