# PostgreSQL-OLAPfunctions-idea
在《SQL基础教程》窗口函数一章中，对计算移动平均方法里关键字的解释补充。


# Requirements
* PostgreSQL == 12.1

# Text
0. 以下内容请参考《SQL基础教程(MICK著)》第8-1章。


1. 预备知识

1).数据准备(后续更新)
```PostgreSQL
 product_id | product_name | product_type | sale_price | purchase_price | regist_date
------------+--------------+--------------+------------+----------------+-------------
 0001       | T恤衫        | 衣服         |       1000 |            500 | 2009-09-20
 0002       | 打孔器       | 办公用品     |        500 |            320 | 2009-09-11
 0003       | 运动T恤      | 衣服         |       4000 |           2800 |
 0004       | 菜刀         | 厨房用具     |       3000 |           2800 | 2009-09-20
 0005       | 高压锅       | 厨房用具     |       6800 |           5000 | 2009-01-15
 0006       | 叉子         | 厨房用具     |        500 |                | 2009-09-20
 0007       | 擦菜板       | 厨房用具     |        880 |            790 | 2008-04-28
 0008       | 圆珠笔       | 办公用品     |        100 |                | 2009-11-11
(8 行记录)
```

2).PostgreSQL窗口函数语法为：
```PostgreSQL
-- primary by <列清单> 可省略
<窗口函数> over (primary by <列清单> order by <排序用列清单>)
```

3).计算移动平均(参照P266)
  
  在计算移动平均中，把指定“最靠近的3行”作为汇总对象的代码清单(当前记录及其前2行)：
```PostgreSQL
select product_id,product_name,sale_price,
       avg(sale_price) over (order by product_id
                             rows 2 preceding) as moving_avg
  from product;
```
运行结果为：
```PostgreSQL
 product_id | product_name | sale_price |      moving_avg
------------+--------------+------------+-----------------------
 0001       | T恤衫        |       1000 | 1000.0000000000000000
 0002       | 打孔器       |        500 |  750.0000000000000000
 0003       | 运动T恤      |       4000 | 1833.3333333333333333
 0004       | 菜刀         |       3000 | 2500.0000000000000000
 0005       | 高压锅       |       6800 | 4600.0000000000000000
 0006       | 叉子         |        500 | 3433.3333333333333333
 0007       | 擦菜板       |        880 | 2726.6666666666666667
 0008       | 圆珠笔       |        100 |  493.3333333333333333
 ```
 
 
2. 问题来源
前文代码运行正常，但是在《SQL基础教程》P267中，将关键字‘following’替换成‘preceding’后(即改为当前记录及其后2行)，并不能出现P268的查询结果。
```PostgreSQL
select product_id,product_name,sale_price,
       avg(sale_price) over (order by product_id
                             rows 2 following) as moving_avg
  from product;
```
运行后结果为：
```PostgreSQL
ERROR:  frame starting from following row cannot end with current row
```

当笔者跳过这一步，将框架指定改为当前记录及其前后1行时，程序正常运行：
```PostgreSQL
select product_id,product_name,sale_price,
       avg(sale_price) over (order by product_id
                             rows between 1 preceding and 1 following) as moving_avg
  from product;
```
```PostgreSQL
 product_id | product_name | sale_price |      moving_avg
------------+--------------+------------+-----------------------
 0001       | T恤衫        |       1000 |  750.0000000000000000
 0002       | 打孔器       |        500 | 1833.3333333333333333
 0003       | 运动T恤      |       4000 | 2500.0000000000000000
 0004       | 菜刀         |       3000 | 4600.0000000000000000
 0005       | 高压锅       |       6800 | 3433.3333333333333333
 0006       | 叉子         |        500 | 2726.6666666666666667
 0007       | 擦菜板       |        880 |  493.3333333333333333
 0008       | 圆珠笔       |        100 |  490.0000000000000000
(8 行记录)
```

3.问题解决
  通过ERROR提示与between...and...结构的反思，推测出问题根源来自当前行(current row)。
  
1).假设计算移动平均中第一种方法(仅使用关键字preceding)省略了between...and...结构，则笔者尝试将源代码补全为：
```PostgreSQL
select product_id,product_name,sale_price,
       avg(sale_price) over (order by product_id
                             rows between 2 preceding and current row) as moving_avg
  from product;
```
运行结果：
```PostgreSQL
 product_id | product_name | sale_price |      moving_avg
------------+--------------+------------+-----------------------
 0001       | T恤衫        |       1000 | 1000.0000000000000000
 0002       | 打孔器       |        500 |  750.0000000000000000
 0003       | 运动T恤      |       4000 | 1833.3333333333333333
 0004       | 菜刀         |       3000 | 2500.0000000000000000
 0005       | 高压锅       |       6800 | 4600.0000000000000000
 0006       | 叉子         |        500 | 3433.3333333333333333
 0007       | 擦菜板       |        880 | 2726.6666666666666667
 0008       | 圆珠笔       |        100 |  493.3333333333333333
(8 行记录)
```
因此可以推断，教程书上省略了这一步，只写出省略之后的代码。

2).再将计算移动平均中第二种方法(仅使用关键字following)按照上述解法补全代码：
```PostgreSQL
select product_id,product_name,sale_price,
       avg(sale_price) over (order by product_id
                             rows between current row and 2 following) as moving_avg
  from product;
```
 发现没有出现ERROR情况：
```PostgreSQL
 product_id | product_name | sale_price |      moving_avg
------------+--------------+------------+-----------------------
 0001       | T恤衫        |       1000 | 1833.3333333333333333
 0002       | 打孔器       |        500 | 2500.0000000000000000
 0003       | 运动T恤      |       4000 | 4600.0000000000000000
 0004       | 菜刀         |       3000 | 3433.3333333333333333
 0005       | 高压锅       |       6800 | 2726.6666666666666667
 0006       | 叉子         |        500 |  493.3333333333333333
 0007       | 擦菜板       |        880 |  490.0000000000000000
 0008       | 圆珠笔       |        100 |  100.0000000000000000
(8 行记录)
```

 问题解决。

4.原问题上的思考。
Q：between A and B 结构中的A，B是否存在顺序
 
A：存在。
例：将‘2 preceding’与‘current row’互换位置
```PostgreSQL
select product_id,product_name,sale_price,
       avg(sale_price) over (order by product_id
                             rows between current row and 2 preceding) as moving_avg
  from product;
  ```
结果为：
```PostgreSQL
ERROR:  frame starting from current row cannot have preceding rows
```
同理：第二种与第三种互换后出错。

5.总结
在计算移动平均时
1).仅使用关键字preceding时，完整语法为：rows between n preceding and current row
                              可简化为：rows n preceding
   但是在使用following时不能省略
2).between A and B 结构中A，B顺序不能对换。前后顺序为：n preceding, current row, n following


 


# Author
* Luther Q(QiuLirong)，tczhangzhi(ZhangZhi)
