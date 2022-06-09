##### 20210610
###### 题目
有如下一张表T0610

| ID   | NAME | NUM  |
| ---- | ---- | ---- |
| 1    | 关羽 | 1    |
| 2    | 关羽 | 2    |
| 3    | 关羽 | 6    |
| 4    | 关羽 | 2    |
| 5    | 关羽 | 3    |
| 6    | 曹操 | 2    |
| 7    | 曹操 | 3    |
| 8    | 曹操 | 3    |

求出NAME中每组累加/每组总数的比例大于0.6的ID和NAME

预期的结果应该为
| ID   | NAME | NUM  |
| ---- | ---- | ---- |
| 7    | 曹操 | 3    |
| 8    | 曹操 | 3    |
| 3    | 关羽 | 6    |
| 4    | 关羽 | 2    |
| 5    | 关羽 | 3    |

###### 解答：

```
SELECT B.ID, B.NAME, B.NUM 
FROM
(
  SELECT A.*,
  (SELECT SUM(T1.NUM) FROM T20210610 T1 WHERE T1.ID <= A.ID AND T1.NAME = A.NAME) / 
  (SELECT SUM(T2.NUM) FROM T20210610 T2 WHERE T2.NAME = A.NAME) Ratio
  FROM T20210610 A
) B WHERE B.Ratio >= 0.6;
```

##### 20210611

###### 题目
有如下一张表：

| ID   | NUM  |
| :--: | ---- |
| 1    | 5    |
| 2    | 11   |
| 3    | 0    |
| 4    | -2   |
| 5    | 2    |
| 6    | 9    |
| 7    | 1    |
| 8    | -4   |
| 9    | -7   |

要求：当Num中的数据同时大于上下两行数据，返回是

当Num中的数据小于上下两行数据中的任何一行，返回否

例如：11大于5,11大于0，所以返回是
5小于11所以返回否

预期的结果如下：

| ID   | NUM  | Result |
| ---- | ---- | ------ |
| 1    | 5    | 否     |
| 2    | 11   | 是     |
| 3    | 0    | 否     |
| 4    | -2   | 否     |
| 5    | 2    | 否     |
| 6    | 9    | 是     |
| 7    | 1    | 否     |
| 8    | -4   | 否     |
| 9    | -7   | 否     |

###### 答案：

##### 20211208

有如下两张表：

T1208A

| QTY  | HU   | Charg |
| ---- | ---- | ----- |
| 3    | x    | N1    |
| 2    | x    | N2    |

T1208B

| HU   | serial_number |
| ---- | ------------- |
| x    | A1            |
| x    | A8            |
| x    | A3            |
| x    | A5            |
| x    | A0            |

```sql
create table T1208A (HU varchar(20), QTY int, Charg varchar(20));
insert into T1208A values('X', 3, 'N1');
insert into T1208A values('X', 2, 'N2');

create table T1208B (HU varchar(20), serial_number varchar(20));
insert into T1208B values ('X', 'A1');
insert into T1208B values('X', 'A8');
insert into T1208B values('X', 'A3');
insert into T1208B values('X', 'A5');
insert into T1208B values('X', 'A0');
```

希望把表T1208A的Charg**随机**分配到表T1208B里面，但是数量得对上，比如表T1208A的N1对应的QTY是3，则表T1208B只随机分配3个serial_number过去。两个表的关联关系为HU字段，表T1208A的QTY合计一定是表T1208B的条数。

答案：

```sql
SELECT t2.HU,t2.SERIAL_NUMBER,t1.CHARG 
FROM 
(select * ,
SUM(QTY) OVER(PARTITION BY HU ORDER BY HU 
ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS SumQty from T1208A ) as t1 join
(select * , 
ROW_NUMBER() OVER(PARTITION by HU order by HU) as RN 
from T1208B ) as t2
on t1.HU=t2.HU 
AND t2.RN>t1.SumQty-t1.QTY 
AND t2.RN<=t1.SumQty
```

结果：

| HU   | SERIAL_NUMBER | CHARG |
| ---- | ------------- | ----- |
| x    | A1            | N1    |
| x    | A8            | N1    |
| x    | A3            | N1    |
| x    | A5            | N2    |
| x    | A0            | N2    |

##### 20211209

题目

表T1209中的字段时user_id，time（用户访问时间），求每个用户相邻两次浏览时间之差小于三分钟的次数

```sql
CREATE TABLE T1209  (
user_id INT,
times DATETIME
)

INSERT INTO T1209  VALUES (1,'2020-12-7 21:13:07');
INSERT INTO T1209  VALUES (1,'2020-12-7 21:15:26');
INSERT INTO T1209  VALUES (1,'2020-12-7 21:17:44');
INSERT INTO T1209  VALUES (2,'2020-12-13 21:14:06');
INSERT INTO T1209  VALUES (2,'2020-12-13 21:18:19');
INSERT INTO T1209  VALUES (2,'2020-12-13 21:20:36');
INSERT INTO T1209  VALUES (3,'2020-12-21 21:16:51');
INSERT INTO T1209  VALUES (4,'2020-12-16 22:22:08');
INSERT INTO T1209  VALUES (4,'2020-12-2 21:17:22');
INSERT INTO T1209  VALUES (4,'2020-12-30 15:15:44');
INSERT INTO T1209  VALUES (4,'2020-12-30 15:17:57');
```

