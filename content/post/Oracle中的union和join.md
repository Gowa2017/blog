---
title: Oracle中的union和join
toc: true
date: 2017-01-03 22:39:53
tags: [Oracle, 数据库]
categories: 
    - 数据库
---
有时候，我们需要把很多表的查询结果给合并在一起显示或者导出，有时候呢我们又需要几张表联合一些条件进行查询，所以我们常会用到**join**和**union**语句。
<!--more-->
# UNION
[官方说明](http://docs.oracle.com/cd/B19306_01/server.102/b14200/queries004.htm)
包含**UNION [ALL], INTERSECT, MINUS**三个操作符，具有相同的优先级（可以用`(...)`进行改变优先级），含有多个的时候，自左至右进行计算。
在每个组成UNION查询的子查询中，其**select list(选择列表)**必须具有相同的数量和数据组类型。
如果子查询选择的是字符数据，其返回值的数据类型由下决定：
- 所有的子查询选择的`char`值具有相同的`length`，返回值就是`char(length)`。如果查询的值类型分别是`char(length1), char(length2)...`，其返回值是`varchar2(max(length1,length2,...))`
- 如果子查询选择的值都是`varchar2`，那么返回值就是`varchar2`。
对于选择的是数值数据：
- 任何一个查询选择的值是`binary_double`，其返回值就是`binary_double`。
- 任何一个查询选择的值是`binary_float`，其返回值就是`binary_float`。
- 所有查询选择的值是`number`，其返回值才是`number`。

使用这几个`集操作符`的时候，Oracle不会进行隐含的数据类型组之间的转换。所以，如果查询包含`number`和`char`类型的话，Oracle返回一个错误。
比如:
```sql
	select '3' from dual
		intersect
	select 3f from dual;
```
会返回一个错误。但：
```sql
	select 3 from dual
		intersect
	select 3f from dual;
```
则会被在类型组内隐含进行转换成：
```sql
	select to_binary_float(3) from dual
		intersect
	select 3f from dual;
```
## 限制
集操作符服从以下限制：
- 对列类型为`BLOB, CLOB, BFILE, VARRAY`或者`嵌套表`无效。
- `UNION, INTERSECT, MINUS`对`LONG`列无效。
- 在集操作符前的选择列表包含表达式的话，那么必须得对列设置别名以便后面在*order by clause*内使用。
- 不能用`for_update_clause`共用
- 在这些操作符的子查询内不能使用`order_by_clause`
- You cannot use these operators in SELECT statements containing TABLE collection expressions.
## Example
查询中`to_char(null)`用在当表中没有某列的时候来匹配数据类型。
```sql
SELECT location_id, department_name "Department", 
   TO_CHAR(NULL) "Warehouse"  FROM departments
   UNION
   SELECT location_id, TO_CHAR(NULL) "Department", warehouse_name 
   FROM warehouses;

```
`union`操作符联合结果中不重复的结果，`union all`联合所有的结果。
```sql
	SELECT product_id FROM order_items
	UNION
	SELECT product_id FROM inventories;

	SELECT location_id  FROM locations 
	UNION ALL 
	SELECT location_id  FROM departments;
```
`INTERSECT`相交操作符联合子查询中都有的行。
```sql
	SELECT product_id FROM inventories
	INTERSECT
	SELECT product_id FROM order_items;
```
`MINUS`相减操作符联合第一个查询的行并且没有在第二个查询中出现的行（同时会去重）。
```sql
	SELECT product_id FROM inventories
	MINUS
	SELECT product_id FROM order_items;
```
# JOIN
[官方说明](http://docs.oracle.com/cd/B19306_01/server.102/b14200/queries006.htm)
`JOIN`用来从两或多个表、视图、物化视图中结合数据。在`FROM`后面的表都会进行一个`JOIN`操作，这样我们就可以用`SELECT`语句查询这个`JOIN`中的任意列。当然，如果这些表中有相同名的列，就要用`tbl.col`这样的形式来来完整引用列了。
## Join Conditions
大多数join查询有一个**Join Condition**，可能出现在`FROM`或`WHERE`语句中，其会比较从不同表中的两列。对*Join Contidion*为*TRUE*的行，就把两个表中那一行组合成一行。需要注意的是不能出现在**select list**中。
对与join三个或以上的表，Oracle首先Join根据*Join Condition*Join前两个表，然后再把表这个结果和新表根据*Join Conditon*进行Join，直到把所有表都Join完。Oracle的Optimizer了决定Join的顺序。
## Equijoins
一个`equijoin`就是*Join Condition*包含一个等号，对指定的列具有相等的值的行进行Join。
## Self Joins
表本身进行Join。在`FROM`后出现两次，并且跟随别名。
## Cartesian Products（笛卡尔乘积）
如果一个Join查询不包含*Join Conditon*，那么Oracle返回的就是一个他们的**Cartesian product**。这个结果一般没有什么用，所以Join的时候最后都指定*Join Conditon*。
## Inner Joins(simple join)
只Join满足的行。
## Outer Joins
`Outer Join`扩展了`simple join`的结果。一个`outer join`返回所有满足*Join Conditon*的行，同时从一个不满足条件的表返回一些或所有行。
### LEFT [outer] JOIN
想要Join表 A,B，同时返回A的所有行。在`FROM`后面使用`left [outer] join`语句，或者在`WHERE`语句中的*Join Conditon*对B的所有列使用**outer join operator**（+）。对A在B中没有匹配行，在B的列中就会返回NULL。
举个例子
有一个属地代码表md_area(areano, name)。
有一个用户表users(mdn,areano,....)。
我现在要统计users表中各属地的用户数，还要根据代码显示出属地名称，以便更加直观的进行统计。
```sql
	select area, areaname, ct from
		(select areano as area, count(*) as ct from users group by areano) t1
		left join 
		(select areano, name as areaname from md_area t2) on t2.areano = t1.area;
```
### Right [outer] Join
接上例，其等于:
```sql
	select area, areaname, ct from
		(select areano, name as areaname from md_area t2) 
		right join 
		(select areano as area, count(*) as ct from users group by areano) t1
		on t2.areano = t1.area;
```


## Antijoins (反连接）
## Semijoins （半连接）

