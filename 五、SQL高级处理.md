# 1、窗口函数
## 概念及使用方法
**窗口函数**，即**OLAP函数**。OLAP 是OnLine AnalyticalProcessing 的简称，意思是对数据库数据进行实时分析处理。

<font color=red>窗口函数</font>通用形式:

> <窗口函数> OVER (PARTITION BY <列名>
                      ORDER BY <排序用列名>)

* PARTITON BY是用来分组，即选择要看哪个窗口，类似于GROUP BY，但不具备汇总功能，不会改变原始数据行数，<font color=blue>该部分可省略</font>
* ORDER BY 用来排序，即决定窗口内是按哪种规则(字段)来排序

```sql
SELECT product_name
       ,product_type
       ,sale_price
       ,RANK() OVER (PARTITION BY product_type
                          ORDER BY sale_price) AS rank_num
  FROM product
```
|product_name|	product_type|	sale_price|	ranking|
|:---|:---|:---|:---|
|圆珠笔	|办公用|	100	|   1|
|打孔器	|办公用|	500	|   2|
|叉子	|厨房用|	500	|   1|
|擦菜板	|厨房用|	880	|   2|
|菜刀	|厨房用|	3000|	3|
|高压锅	|厨房用|	6800|	4|
|T恤	|衣服	|   1000|	1|
|运动T恤|	衣服|	4000|	2|

PARTITION BY 能够设定窗口对象范围。一个商品种类就是一个小的"窗口"
ORDER BY 每个窗口中按指定列排序

# 2、函数种类
* 将SUM、MAX、MIN等聚合函数用在窗口函数中
* RANK、DENSE_RANK等排序用的专用串口函数
## 2.1 专用窗口函数

* RANK函数

计算排序，存在相同记录则跳过之后位次
如：1，1，1，4

* DENSE_RANK函数

计算排序，存在相同位次，不跳过后续

如：1，1，1，2

* ROW_NUMBER函数

赋予唯一的连续位次。

## 2.2 聚合函数在窗口函数上的使用

结果是一个累计的聚合函数值
> SUM() 当前所在行及之前所有的行的合计  
AVG() 当前所在行及之前所有的行的均值

# 3、计算移动平均

指定详细的汇总范围——**框架(frame)**

语法
> <窗口函数> OVER (ORDER BY <排序列> ROWS n PRECEDING)
> <窗口函数> OVER (ORDER BY <排序列> ROWS BETWEEN n PRECEDING AND n FOLLOWING)

n PRECEDING -- 将框架指定为 “截止到之前 n 行”，加上自身行
n FOLLOWING -- 将框架指定为 “截至到之后 n 行”，加上自身行
BETWEEN n PRECEDING AND n FOLLOWING -- 将框架指定为‘之前n行’+‘自身行’+‘之后n行’

### <font color=red>窗口函数适用范围和注意事项</font>

* 原则上，窗口函数只能在SELECT子句中使用。
* 窗口函数OVER 中的ORDER BY 子句并不会影响最终结果的排序。其只是用来决定窗口函数按何种顺序计算。

# 4、GROUPING运算符

## ROLLUP - 计算合计及小计

`GROUP BY` 只能得到每个分类的小计，计算分类的合计用 `ROLLUP` 关键字

>GROUP BY <聚合依据列> WITH ROLLUP  

ROLLUP 可以对多列进行汇总求小计和合计。

# 延申--窗口函数其他应用

计算**组内平均值、组内求和及组内百分比和累计百分比**  
【组内求和与平均】
窗口函数中的ORDER BY 依据改为分组条件，将移动求和改为组内汇总，组内平均写法一样

```sql
 SELECT product_name
			 ,product_type
			 ,sale_price
			 ,SUM(sale_price) OVER (PARTITION BY product_type
												 ORDER BY sale_price) AS sum_org
			 ,SUM(sale_price) OVER (PARTITION BY product_type
												 ORDER BY product_type) AS sum_tol
  FROM product;
  ```
【组内百分比与累计百分比】

组内占比为 单价/组内求和，即sale_price/sum_tol  
累计占比为 累计求和/组内求和，即sum_org/sum_tol  
求百分比用上述两个结算结果的商乘以100，再拼接"%"

实现如下：
```sql
 SELECT product_name
				,product_type
				,sale_price
				,sum_org
				,CONCAT(ROUND((sale_price/sum_tol)*100, 2), "%") AS sal_percent
				,sum_tol
				,CONCAT(ROUND((sum_org/sum_tol)*100, 2), "%") AS org_percent
	FROM ( SELECT product_name
			 ,product_type
			 ,sale_price
			 ,SUM(sale_price) OVER (PARTITION BY product_type
												 ORDER BY sale_price) AS sum_org
			 ,SUM(sale_price) OVER (PARTITION BY product_type
												 ORDER BY product_type) AS sum_tol
  FROM product) AS p
```	
结果：

|product_name |roduct_type |ale_price|sum_org |sal_percent| sum_tol|org_percent|
|:---|:---|:---|:---|:---|:---|:---|
|圆珠笔	      |   办公用品|100	     | 100	  |     16.67%|	600	   | 16.67%|
|打孔器	      |   办公用品|500	     | 600	  |     83.33%|	600	   | 100.00%|
|叉子	      |   厨房用具|500	     | 500	  |     4.47%|	11180	|4.47%|
|擦菜板	      |   厨房用具|880	     | 1380	  |     7.87%|	11180	|12.34%|
|菜刀	      |   厨房用具|3000	     | 4380	  |     26.83%|	11180	|39.18%|
|高压锅	      |   厨房用具|6800	     | 11180  |     60.82%|	11180	|100.00%|
|T恤	      |   衣服	   |1000     | 1000	  |     20.00%|	5000	|20.00%|
|运动T恤      |   	衣服	|4000	 |  5000  |     80.00%|	5000	|100.00%|


# 练习题

## 1
请说出针对本章中使用的 product（商品）表执行如下 SELECT 语句所能得到的结果。  
```sql
SELECT  product_id
       ,product_name
       ,sale_price
       ,MAX(sale_price) OVER (ORDER BY product_id) AS Current_max_price
  FROM product
```
截至本行数据，统计sale_price的最大值，若换为MIN()为截至本行统计数据的最小值

## 2  
继续使用product表，计算出按照登记日期（regist_date）升序进行排列的各日期的销售单价（sale_price）的总额。排序是需要将登记日期为NULL 的“运动 T 恤”记录排在第 1 位（也就是将其看作比其他日期都早）

思路：使用窗口函数以日期作为分组条件，排序条件为升序(默认)，对销售单价求和
```sql
 SELECT regist_date
				,sale_price
				,SUM(sale_price) OVER (PARTITION BY regist_date
															ORDER BY regist_date) AS date_price
	 FROM product;
```
查询结果
|regist_date|sale_price	|date_price|
|:---|:---|:---|
|Null	       |    4000	|4000|
|2008-04-28|880	        |880|
|2009-01-15|6800	    |6800|
|2009-09-11|500	        |500|
|2009-09-20|1000	    |4500|
|2009-09-20|3000	    |4500|
|2009-09-20|500	        |4500|
|2009-11-11|100	        |100|

## 3
思考题

① 窗口函数不指定PARTITION BY的效果是什么？

不指定PARTITION BY，便不会分组，窗口函数针对SELECT部分可以取到的所有数据，我是这样理解的。

② 为什么说窗口函数只能在SELECT子句中使用？实际上，在ORDER BY 子句使用系统并不会报错。

因为ORDER BY 的依据是SELECT中的内容，在SELECT之后执行，本质上还是以SELECT为基础。