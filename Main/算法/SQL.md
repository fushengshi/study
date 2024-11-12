# SQL

## 开窗函数

开窗函数有两类：一类是聚合开窗函数，一类是排序开窗函数。

```sql
函数() over (partition by 列名 order by 列名 desc) as 别名
```

- `row_number()`

  对相等的值不进行区分，其实就是行号，相等的值对应的排名不同，序号从1到n连续。

- `rank()`

  相等的值排名相同，但若有相等的值，则序号从1到n不连续。

  如果有两个人都排在第一名，第3个人是第3名。

- `dense_rank()`

  对相等的值排名相同，但序号从1到n连续。

  如果有两个人都排在第一名，第3个人是第2名。

例如：

```
Employee =
| id | name  | salary | departmentId |
| -- | ----- | ------ | ------------ |
| 1  | Joe   | 85000  | 1            |
| 2  | Henry | 80000  | 2            |
| 3  | Sam   | 60000  | 2            |
| 4  | Max   | 90000  | 1            |
| 5  | Janet | 69000  | 1            |
| 6  | Randy | 85000  | 1            |

Department =
| id | name  |
| -- | ----- |
| 1  | IT    |
| 2  | Sales |
```

执行开窗函数，按照`departmentId`分组排序：

```sql
with e as (
    select name, salary, departmentId,
    	   dense_rank() over (partition by departmentId order by salary desc) as no
    from Employee
)
select * from e 
```

输出结果：

```
| name  | salary | departmentId | no   |
| ----- | ------ | ------------ | ---- |
| Max   | 90000  | 1            | 1    |
| Joe   | 85000  | 1            | 2    |
| Randy | 85000  | 1            | 2    |
| Will  | 70000  | 1            | 3    |
| Janet | 69000  | 1            | 4    |
| Henry | 80000  | 2            | 1    |
| Sam   | 60000  | 2            | 2    |
```



### [部门工资前三高的所有员工](https://leetcode.cn/problems/department-top-three-salaries/)

```sql
with e as (
    select name, salary, departmentId,
    	   dense_rank() over (partition by departmentId order by salary desc) as no
    from Employee
)

select d.name as Department, e.name as Employee, e.salary as Salary
from e left join Department d on e.departmentId = d.id
where e.no <= 3
```



### [第二高的薪水](https://leetcode.cn/problems/second-highest-salary/)

```sql
with e as (
    select salary,
           dense_rank() over (order by salary desc) as no
    from Employee
)

select if(max(e.no) < 2, null, e.salary) as SecondHighestSalary
from e where e.no = 2
```



### [删除重复的电子邮箱](https://leetcode.cn/problems/delete-duplicate-emails/)

```sql
with p as ( 
    select id,
           row_number() over (partition by email order by id) as no
    from Person
)

delete from Person where id in (
    select id from p where no > 1
)
```



### [连续出现的数字](https://leetcode.cn/problems/consecutive-numbers/)

不连续 id 的同一个 num 不被视作连续出现。

```sql
with t as (
    select id, num,
           row_number() over (partition by num order by id) as no
    from Logs
)

select distinct num as ConsecutiveNums
from t
group by num, id - no + 1
having count(*) >= 3
```

如果一个num连续出现时，那么它的`真实id - 出现的次数`一定是个定值。 

1. 假设一个num出现后，它的 `真实id` 为 `i` ，同时假设它是第 `k` 次出现的， 差值为 `i-k`。
2. 当它连续出现第 `n` 次时，它的 `真实id` 一定为 `i+n`，它出现的次数为 `k+n`，差值为 `i+n-(k+n)=i-k`。
3. 如果它不连续出现，假设`m`个其他num出现之后，它又出现了，则它的真实序列为 `i+n+m`，而出现的次数为 `k+n+1`，差值为 `i-k+m-1` 。

```
BIGINT UNSIGNED value is out of range in '(`t`.`id` - `t`.`no`)'
```

id 为 0 时会由于 id 和 `row_number()` 函数生成的行号相减产生了负数值，导致出现了 BIGINT UNSIGNED 数据类型的取值范围溢出的问题。故需要对 id 加偏移量使得减法计算后结果非负。





### [2016年的投资](https://leetcode.cn/problems/investments-in-2016/)

```sql
with t as ( 
    select TIV_2016,
           count(pid) over (partition by TIV_2015) as rn1,
           count(pid) over (partition by concat(LAT, LON)) as rn2
    from Insurance
)

select round(sum(TIV_2016), 2) as TIV_2016 from t where rn1 > 1 and rn2 = 1
```



## 分组

### [列出指定时间段内所有的下单产品](https://leetcode.cn/problems/list-the-products-ordered-in-a-period/)

```sql
select
    product_name, sum(unit) as unit
from Orders o 
left join Products p on o.product_id = p.product_id
where order_date like '2020-02%'
group by product_name having sum(unit) >= 100
```



### 查询指定时间范围内：每时每分中请求的次数

```sql
SELECT
  DATE_FORMAT(CREATE_DATETIME, '%H:%i') tim,
  count(*) coun
FROM
  TABLE_NAME
WHERE
  CREATE_DATETIME >= '2022-04-25 00:00:00'
  AND CREATE_DATETIME <= '2022-04-25 12:59:59'
GROUP BY
  DATE_FORMAT(CREATE_DATETIME, '%H:%i')
ORDER BY
  tim DESC
```

在使用 GROUP BY 和 HAVING 时，必须遵循一定的顺序：WHERE > GROUP BY > HAVING > ORDER BY。

WHERE 子句用于在分组前对数据进行筛选，GROUP BY 用于对数据进行分组，HAVING 用于对分组后的结果进行筛选，ORDER BY 用于对查询结果进行排序。虽然不是必须同时使用 HAVING，但如果需要筛选聚合函数的结果，则必须使用 HAVING。































































