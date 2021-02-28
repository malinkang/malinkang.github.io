---
title: sqlite常用语法总结
date: 2014-03-26 17:23:24
tags:
draft: false
---

<!--more-->
# 1.创建表 

SQLite数据类型

integer：整数值

real：浮点数

text:文本

blob：二进制大对象

NULL


## 唯一性约束`UNIQUE`

```sql
CREATE TABLE Contacts (id INTEGER PRIMARY KEY,name TEXT NOT NULL,phone TEXT NOT NULL DEFAULT 'UNKNOWN',UNIQUE(name,phone));

INSERT INTO Contacts (name,phone) values ('Jerry','UNKNOWN');
//再次插入会出现错误
INSERT INTO Contacts (name,phone) values ('Jerry','UNKNOWN');
Error: UNIQUE constraint failed: Contacts.name, Contacts.phone
//插入成功，因为name自身不是唯一的，但是name和phone合起来是唯一的
INSERT INTO Contacts (name,phone) values ('Jerry','555-1212');
```
NULL不等于任何值，甚至不等于其他的NULL所以不会出现NULL冲突。

## 主键约束

## `NOT NULL`约束

## `check`约束

check约束允许定义表达式来测试要插入或者更新的字段值。

```sql

CREATE TABLE Foo ( x INTEGER,y INTEGER CHECK (y>x), z INTEGER CHECK (z>abs(y)));
INSERT INTO Foo VALUES (-2,-1,2);
sqlite> INSERT INTO Foo VALUES (-2,-1,1);
//不满足条件出错
INSERT INTO Foo VALUES (-2,-1,1);
Error: CHECK constraint failed: Foo

```

SQLite提供了5种可能的冲突解决方案来解决冲突：`replace`、`ignore`、`fail`、`abort`、`rollback`。

`replace`：当违反了唯一性约束时，新记录会替代原来的记录
```sql
CREATE TABLE Contacts (id INTEGER PRIMARY KEY,name TEXT NOT NULL,phone TEXT UNIQUE ON CONFLICT REPLACE);
sqlite> INSERT INTO Contacts (name,phone) values ('Jerry','12345678');sqlite> INSERT INTO Contacts (name,phone) values ('Tom','12345678');
SELECT * FROM Contacts;


2|Tom|12345678
```

```sql

CREATE TABLE COMPANY(
   ID INT PRIMARY KEY     NOT NULL,
   NAME           TEXT    NOT NULL,
   AGE            INT     NOT NULL,
   ADDRESS        CHAR(50),
   SALARY         REAL    DEFAULT 50000.00,
   UNIQUE (NAME,AGE) ON CONFLICT REPLACE
);
INSERT INTO COMPANY VALUES (0,"MLK",22,"ABC",20000.0);
//替换上一条
INSERT INTO COMPANY VALUES (1,"MLK",22,"ABC",20000.0);
//替换上一条
INSERT INTO COMPANY VALUES (2,"MLK",22,"EFG",10000.0);
INSERT INTO COMPANY VALUES (3,"MLK",23,"EFG",10000.0);
SELECT * FROM COMPANY;
//查询结果
2|MLK|22|EFG|10000.0
3|MLK|23|EFG|10000.0

```

`ignore`：违反唯一性约束，将忽略新的记录。
```sql

CREATE TABLE Contacts (id INTEGER PRIMARY KEY,name TEXT NOT NULL,phone TEXT UNIQUE ON CONFLICT IGNORE);
INSERT INTO Contacts (name,phone) values ('Jerry','12345678');
INSERT INTO Contacts (name,phone) values ('Tom','12345678');
SELECT * FROM Contacts;
1|Jerry|12345678

```
`fail`：当约束发生时，SQLite终止命令，但是不恢复约束违反之前已经修改的记录。也就是说，在约束违法发生前的改变都保留。例如，如果update命令在第100行违法约束，那么前99行已经修改的记录不会回滚。对第100行和之外的改变不会发生，因为命令已经终止了。

abort：当约束违反发生时，SQLite恢复命令所做的所有改变并终止命令。abort是SQLite中所有操作的默认解决方法，也是SQL标准定义的行为。

rollback：当约束违反发生时，SQLite执行回滚，终止当前命令和整个事务。






##### LIKE操作符和GLOB操作符

```sql

CREATE TABLE Persons (Id Integer PRIMARY KEY AUTOINCREMENT,LastName varchar(20),FirstName varchar(20),Address varchar(30),City varchar(20));

INSERT INTO Persons VALUES (1,'Adams','John','Oxford Street','London');
INSERT INTO Persons VALUES (2,'Bush','George','Fifth Avenue','New York');
INSERT INTO Persons VALUES (3,'Carter','Thomas','Changan','Beijing');
SELECT * FROM Persons;
1|Adams|John|Oxford Street|London
2|Bush|George|Fifth Avenue|New York
3|Carter|Thomas|Changan|Beijing

//从Persons表中获取居住在以N开始的城市里的人

SELECT * FROM Persons WHERE City LIKE 'N%';
2|Bush|George|Fifth Avenue|New York
//从 "Persons" 表中选取居住在包含 "lon" 的城市里的人
SELECT * FROM Persons WHERE City LIKE '%lon%';
1|Adams|John|Oxford Street|London

//从 "Persons" 表中选取居住在以 "g" 结尾的城市里的人
SELECT * FROM Persons WHERE City LIKE '%g';
3|Carter|Thomas|Changan|Beijing

```
通配符
%   替代一个或多个字符
_   仅替代一个字符
[charlist]  字符列中的任何单一字符
[^charlist]
或者
[!charlist]
不在字符列中的任何单一字符

GLOB操作符在行为上与LIKE操作符相似。

```sql
SELECT * FROM Persons WHERE City GLOB 'B*';
3|Carter|Thomas|Changan|Beijing
4|Gates|Bill|Xuanwumen10|Beijing
```


##### IN 操作符

IN 操作符允许我们在 WHERE 子句中规定多个值。

```sql
CREATE TABLE Persons (Id Integer PRIMARY KEY AUTOINCREMENT,LastName varchar(20),FirstName varchar(20),Address varchar(30),City varchar(20));

INSERT INTO Persons VALUES (1,'Adams','John','Oxford Street','London');
INSERT INTO Persons VALUES (2,'Bush','George','Fifth Avenue','New York');
INSERT INTO Persons VALUES (3,'Carter','Thomas','Changan','Beijing');
SELECT * FROM Persons;
1|Adams|John|Oxford Street|London
2|Bush|George|Fifth Avenue|New York
3|Carter|Thomas|Changan|Beijing

//从列表中选取姓氏为Adams和Carter的人

SELECT * FROM Persons WHERE LastName IN ('Adams','Carter');
1|Adams|John|Oxford Street|London
3|Carter|Thomas|Changan|Beijing
```

##### BetWeen 操作符

操作符 BETWEEN ... AND会选取介于两个值之间的数据范围。

```sql

CREATE TABLE Persons (Id Integer PRIMARY KEY AUTOINCREMENT,LastName varchar(20),FirstName varchar(20),Address varchar(30),City varchar(20));

INSERT INTO Persons VALUES (1,'Adams','John','Oxford Street','London');
INSERT INTO Persons VALUES (2,'Bush','George','Fifth Avenue','New York');
INSERT INTO Persons VALUES (3,'Carter','Thomas','Changan','Beijing');
INSERT INTO Persons values (4,'Gates','Bill','Xuanwumen10','Beijing');

SELECT * FROM Persons;
1|Adams|John|Oxford Street|London
2|Bush|George|Fifth Avenue|New York
3|Carter|Thomas|Changan|Beijing
4|Gates|Bill|Xuanwumen10|Beijing

//以字母顺序显示介于 "Adams"（包括）和 "Carter"之间的人
SELECT * FROM Persons WHERE LastName BETWEEN 'Adams' AND 'Carter';
1|Adams|John|Oxford Street|London
2|Bush|George|Fifth Avenue|New York
3|Carter|Thomas|Changan|Beijing

//显示上述范围之外的人，使用NOT操作符
SELECT * FROM Persons WHERE LastName NOT BETWEEN 'Adams' AND 'Carter';
4|Gates|Bill|Xuanwumen10|Beijing
```

##### Alias 别名

```sql
CREATE TABLE Persons (Id Integer PRIMARY KEY AUTOINCREMENT,LastName varchar(20),FirstName varchar(20),Address varchar(30),City varchar(20));

INSERT INTO Persons VALUES (1,'Adams','John','Oxford Street','London');
INSERT INTO Persons VALUES (2,'Bush','George','Fifth Avenue','New York');
INSERT INTO Persons VALUES (3,'Carter','Thomas','Changan','Beijing');
INSERT INTO Persons values (4,'Gates','Bill','Xuanwumen10','Beijing');

SELECT * FROM Persons;
1|Adams|John|Oxford Street|London
2|Bush|George|Fifth Avenue|New York
3|Carter|Thomas|Changan|Beijing
4|Gates|Bill|Xuanwumen10|Beijing

SELECT LastName As Family,FirstName As Name FROM Persons;
Adams|John
Bush|George
Carter|Thomas
Gates|Bill

```

##### Null值

如果表中的某个列是可选的，那么我们可以在不向该列添加值的情况下插入新记录或更新已有的记录。这意味着该字段将以 NULL 值保存。

无法比较 NULL 和 0；它们是不等价的。

```sql
//选取在 "Address" 列中带有 NULL 值的记录
SELECT LastName,FirstName,Address FROM Persons Where Address IS NULL;
//选取在 "Address" 列中不带有 NULL 值的记录
 SELECT LastName,FirstName,Address FROM Persons Where Address IS NOT NULL;
```




##### UNION和UNION ALL操作符

UINON操作符用于合并两个或多个SELECT语句的结果集。

请注意，UNION 内部的 SELECT 语句必须拥有相同数量的列。列也必须拥有相似的数据类型。同时，每条 SELECT 语句中的列的顺序必须相同。


```sql
CREATE TABLE Employees_China (E_ID Integer,E_Name varchar(20));
INSERT INTO Employees_China VALUES (01,'Zhang,Hua');
INSERT INTO Employees_China VALUES (02,'Wang,Wei');
INSERT INTO Employees_China VALUES (03,'Carter,Thomas');
INSERT INTO Employees_China VALUES (04,'Yang,Ming');
SELECT E_Name FROM Employees_China;
E_ID        E_Name
----------  ----------
1           Zhang,Hua
2           Wang,Wei
3           Carter,Tho
4           Yang,Ming


CREATE TABLE Employees_USA (E_ID Integer,E_Name varchar(20));
INSERT INTO Employees_USA VALUES (01,'Adams,John');
INSERT INTO Employees_USA VALUES (02,'Bush,George');
INSERT INTO Employees_USA VALUES (02,'Carter,Thomas');
INSERT INTO Employees_USA VALUES (04,'Gates,Bill');

SELECT * FROM Employees_USA;
E_ID        E_Name
----------  ----------
1           Adams,John
2           Bush,Georg
3           Carter,Tho
4           Gates,Bill
//使用UNION命令 UNION 命令只会选取不同的值。
SELECT E_Name FROM Employees_China UNION SELECT E_Name From Employees_USA;
E_Name
----------
Adams,John
Bush,Georg
Carter,Tho
Gates,Bill
Wang,Wei
Yang,Ming
Zhang,Hua
//使用 UNION ALL 命令

SELECT E_Name FROM Employees_China UNION ALL SELECT E_Name From Employees_USA;

E_Name
----------
Zhang,Hua
Wang,Wei
Carter,Tho
Yang,Ming
Adams,John
Bush,Georg
Carter,Tho
Gates,Bill
```
##### GROUP BY

```sql
CREATE TABLE Orders (O_Id Integer PRIMARY KEY AUTOINCREMENT,OrderDate varchar(20),OrderPrice Integer,Customer varchar(20));
INSERT INTO Orders VALUES (1,'2008/12/29','1000','Bush');
INSERT INTO Orders VALUES (2,'2008/11/23','1600','Carter');
INSERT INTO Orders VALUES (3,'2008/10/05','700','Bush');
INSERT INTO Orders VALUES (4,'2008/9/28','300','Bush');
INSERT INTO Orders VALUES (5,'2008/8/6','2000','Adams');
INSERT INTO Orders VALUES (6,'2008/7/21','100','Carter');
SELECT * FROM Orders;
1|2008/12/29|1000|Bush
2|2008/11/23|1600|Carter
3|2008/10/05|700|Bush
4|2008/9/28|300|Bush
5|2008/8/6|2000|Adams
6|2008/7/21|100|Carter

SELECT Customer,SUM(OrderPrice) FROM Orders GROUP BY Customer;
Adams|2000
Bush|2000
Carter|1700
```



##### Limit 

创建表

```sql
CREATE TABLE COMPANY (ID INTEGER PRIMARY KEY AUTOINCREMENT,NAME TEXT,AGE INTEGER,ADDRESS TEXT,SALARY REAL);
INSERT INTO COMPANY VALUES (1,'Paul',32,'California',2000.0);
INSERT INTO COMPANY VALUES (2,'Allen',25,'Texas',1500.0);
INSERT INTO COMPANY VALUES (3,'Teddy',23,'Norway',2000.0);
INSERT INTO COMPANY VALUES (4,'Mark',25,'Rich-Mond',6500.0);
INSERT INTO COMPANY VALUES (5,'David',27,'Texas',8500.0);
INSERT INTO COMPANY VALUES (6,'Kim',22,'South-Hall',4500.0);
INSERT INTO COMPANY VALUES (7,'James',24,'Houston',3500.0);

```

```sql
SELECT * FROM COMPANY;
```


1|Paul|32|California|2000.0
2|Allen|25|Texas|1500.0
3|Teddy|23|Norway|2000.0
4|Mark|25|Rich-Mond|6500.0
5|David|27|Texas|8500.0
6|Kim|22|South-Hall|4500.0
7|James|24|Houston|3500.0


```sql
SELECT * FROM COMPANY LIMIT 2,3;
```

3|Teddy|23|Norway|2000.0
4|Mark|25|Rich-Mond|6500.0
5|David|27|Texas|8500.0


```sql
SELECT * FROM COMPANY LIMIT 2;
```
1|Paul|32|California|2000.0
2|Allen|25|Texas|1500.0

```sql
SELECT * FROM COMPANY LIMIT 2 OFFSET 0;
```
1|Paul|32|California|2000.0
2|Allen|25|Texas|1500.0

```sql
SELECT * FROM COMPANY LIMIT 2 OFFSET 2;
```
3|Teddy|23|Norway|2000.0
4|Mark|25|Rich-Mond|6500.0

##### Joins 


```sql
CREATE TABLE COMPANY ( ID INTEGER PRIMARY KEY AUTOINCREMENT,NAME TEXT,AGE INTEGER,ADDRESS TEXT,SALARY READ);
INSERT INTO COMPANY VALUES (1,'Paul',32,'California',20000.0);
INSERT INTO COMPANY VALUES (2,'Allen',25,'Texas',15000.0);
INSERT INTO COMPANY VALUES (3,'Teddy',23,'Norway',20000.0);
INSERT INTO COMPANY VALUES (4,'Mark',25,'Rich-Mond',65000.0);
INSERT INTO COMPANY VALUES (5,'David',27,'Texas',85000.0);
INSERT INTO COMPANY VALUES (6,'Kim',22,'South-Hall',45000.0);
INSERT INTO COMPANY VALUES (7,'James',24,'Houston',10000.0);

CREATE TABLE DEPARTMENT( ID INTEGER PRIMARY KEY, DEPT TEXT ,EMP_ID TEXT);
INSERT INTO DEPARTMENT VALUES (1, 'IT Billing', 1 );
INSERT INTO DEPARTMENT  VALUES (2, 'Engineering', 2 );
INSERT INTO DEPARTMENT VALUES (3, 'Finance', 7 );

```
交叉连接（CROSS JOIN）把第一个表的每一行与第二个表的每一行进行匹配。如果两个输入表分别有 x 和 y 列，则结果表有 x+y 列。

```sql
SELECT EMP_ID,NAME,DEPT FROM COMPANY CROSS JOIN DEPARTMENT;
```
1|Paul|IT Billing
2|Paul|Engineering
7|Paul|Finance
1|Allen|IT Billing
2|Allen|Engineering
7|Allen|Finance
1|Teddy|IT Billing
2|Teddy|Engineering
7|Teddy|Finance
1|Mark|IT Billing
2|Mark|Engineering
7|Mark|Finance
1|David|IT Billing
2|David|Engineering
7|David|Finance
1|Kim|IT Billing
2|Kim|Engineering
7|Kim|Finance
1|James|IT Billing
2|James|Engineering
7|James|Finance

内连接（INNER JOIN）根据连接谓词结合两个表（table1 和 table2）的列值来创建一个新的结果表。查询会把 table1 中的每一行与 table2 中的每一行进行比较，找到所有满足连接谓词的行的匹配对。当满足连接谓词时，A 和 B 行的每个匹配对的列值会合并成一个结果行。
内连接（INNER JOIN）是最常见的连接类型，是默认的连接类型。INNER 关键字是可选的。
```sql
SELECT EMP_ID, NAME, DEPT FROM COMPANY INNER JOIN DEPARTMENT ON COMPANY.ID = DEPARTMENT.EMP_ID;
```
1|Paul|IT Billing
2|Allen|Engineering
7|James|Finance

外连接（OUTER JOIN）是内连接（INNER JOIN）的扩展。虽然 SQL 标准定义了三种类型的外连接：LEFT、RIGHT、FULL，但 SQLite 只支持 左外连接（LEFT OUTER JOIN）。
外连接（OUTER JOIN）声明条件的方法与内连接（INNER JOIN）是相同的，使用 ON、USING 或 NATURAL 关键字来表达。最初的结果表以相同的方式进行计算。一旦主连接计算完成，外连接（OUTER JOIN）将从一个或两个表中任何未连接的行合并进来，外连接的列使用 NULL 值，将它们附加到结果表中。
下面是左外连接（LEFT OUTER JOIN）的语法：
```sql
SELECT EMP_ID, NAME, DEPT FROM COMPANY LEFT OUTER JOIN DEPARTMENT ON COMPANY.ID = DEPARTMENT.EMP_ID;
```
1|Paul|IT Billing
2|Allen|Engineering
|Teddy|
|Mark|
|David|
|Kim|
7|James|Finance

* [SQLite](http://www.runoob.com/sqlite/sqlite-tutorial.html)
