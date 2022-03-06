## B+树和B树的区别 ##
1. B+树
	- 非叶子节点不存储data,只存储索引(冗余),可以放更多的索引
	- 叶子节点包含所有索引字段
	- 叶子节点用指针连接,双向连接,提高区间访问的性能  
	![](https://github.com/chenkun141/markdownImg/blob/master/Mysql/B%2BTree.png)
2. B树
	- 也只节点具有相同的深度,叶节点的指针为空
	- 所有索引元素不重复
	- 节点中的数据索引从左到右递增排列  
	![](https://github.com/chenkun141/markdownImg/blob/master/Mysql/B-Tree.png) 

## InnoDB和MyISAM区别 ##
1. InnoDB 
	- 聚集索引(主键索引) 叶子节点包含完整的数据记录(.IBD文件)
	- 二级索引(非主键索引): 叶子节点存放主键id,不存放完整数据,主要是为了节约存储空间,保证数据一致性,二级索引查询完之后,根据id回表查询聚集索引获得完整的数据记录
	- 建议表必须建主键,推荐使用整形自增

2. MyISAM
	- 非聚集索引: 索引文件(.MYI文件)和数据文件(.MYD文件)分开存储  
	![](https://github.com/chenkun141/markdownImg/blob/master/Mysql/MyISAM.png)

## hash索引 ##
1. 对索引的的key进行一次hash计算就可以定位出数据存储的位置
2. 很多Hash索引要比B+ 树索引更高效
3. **仅能满足"=","in"**,不支持范围查询
4. hash冲突问题  
![](https://github.com/chenkun141/markdownImg/blob/master/Mysql/hash.png)

## 联合索引最左前缀原理 ##
联合索引在建的时候根据索引字段从左到右顺序排好序的数据

## Explain Extra(扩展列) ##
1. Using index：使用覆盖索引
2. Using where：使用 where 语句来处理结果，并且查询的列未被索引覆盖
3. Using index condition：查询的列不完全被索引覆盖，where条件中是一个前导列的范围
4. Using temporary：mysql需要创建一张临时表来处理查询。出现这种情况一般是要进行优化的，首先是想到用索
引来优化
5. Using filesort：将用外部排序而不是索引排序，数据较小时从内存排序，否则需要在磁盘完成排序。这种情况下一
般也是要考虑使用索引来优化的
6. Select tables optimized away：使用某些聚合函数（比如 max、min）来访问存在索引的某个字段是

## 覆盖索引 ##
mysql执行计划explain结果里的key有使用索引，如果select后面查询的字段都可以从这个索引的树中获取，这种情况一般可以说是用到了覆盖索引，extra里一般都有using index；覆盖索引一般针对的是辅助索引，整个查询结果只通过辅助索引就能拿到结果，不需要通过辅助索引树找到主键，再通过主键去主键索引树里获取其它字段值

> 强制走索引
SELECT * FROM employees force index(idx_name_age_position) WHERE name > 'LiLei' AND age = 22 AND position ='manager';

### Using Filesort文件排序原理 ###
1. filesort文件排序方式  

	- 单路排序：是一次性取出满足条件行的所有字段，然后在sort buffer中进行排序；用trace工具可以看到sort_mode信息里显示< sort_key, additional_fields >或者< sort_key,packed_additional_fields >

	- 双路排序（又叫回表排序模式）：是首先根据相应的条件取出相应的排序字段和可以直接定位行数据的行 ID，然后在 sort buffer 中进行排序，排序完后需要再次取回其它需要的字段；用trace工具可以看到sort_mode信息里显示< sort_key, rowid >

2. MySQL 通过比较系统变量 max_length_for_sort_data(默认1024字节) 的大小和需要查询的字段总大小来判断使用哪种排序模式 
	- 如果 字段的总长度小于max_length_for_sort_data ，那么使用 单路排序模式；
	- 如果 字段的总长度大于max_length_for_sort_data ，那么使用 双路排序模式。

3. 过程详解

	- 单路排序的详细过程：
		1. 从索引name找到第一个满足 name = ‘zhuge’ 条件的主键 id
		2. 根据主键 id 取出整行，取出所有字段的值，存入 sort_buffer 中
		3. 从索引name找到下一个满足 name = ‘zhuge’ 条件的主键 id
		4. 重复步骤 2、3 直到不满足 name = ‘zhuge’
		5. 对 sort_buffer 中的数据按照字段 position 进行排序
		6. 返回结果给客户端

	- 双路排序的详细过程：
		1. 从索引 name 找到第一个满足 name = ‘zhuge’ 的主键id
		2. 根据主键 id 取出整行，把排序字段 position 和主键 id 这两个字段放到 sort buffer 中
		3. 从索引 name 取下一个满足 name = ‘zhuge’ 记录的主键 id
		4. 重复 3、4 直到不满足 name = ‘zhuge’
		5. 对 sort_buffer 中的字段 position 和主键 id 按照字段 position 进行排序
		6. 遍历排序好的 id 和字段 position，按照 id 的值回到原表中取出 所有字段的值返回给客户端

	其实对比两个排序模式，单路排序会把所有需要查询的字段都放到 sort buffer 中，而双路排序只会把主键和需要排序的字段放到 sort buffer 中进行排序，然后再通过主键回到原表查询需要的字段。  
	如果 MySQL 排序内存 sort_buffer 配置的比较小并且没有条件继续增加了，可以适当把 max_length_for_sort_data 配置小点，让优化器选择使用双路排序算法，可以在sort_buffer 中一次排序更多的行，只是需要再根据主键回到原表取数据。  
	如果 MySQL 排序内存有条件可以配置比较大，可以适当增大 max_length_for_sort_data 的值，让优化器优先选择全字段排序(单路排序)，把需要的字段放到 sort_buffer 中，这样排序后就会直接从内存里返回查询结果了。  
	所以，MySQL通过 max_length_for_sort_data 这个参数来控制排序，在不同场景使用不同的排序模式，从而提升排序效率。  
	注意，如果全部使用sort_buffer内存排序一般情况下效率会高于磁盘文件排序，但不能因为这个就随便增大sort_buffer(默认1M)，mysql很多参数设置都是做过优化的，不要轻易调整。  

## 索引下推 ##
对于辅助的联合索引,正常情况按照最左前缀原则,SELECT * FROM employees WHERE name like 'LiLei%'AND age = 22 AND position ='manager' 这种情况只会走name字段索引，因为根据name字段过滤完，得到的索引行里的age和position是无序的，无法很好的利用索引。    

在MySQL5.6之前的版本，这个查询只能在联合索引里匹配到名字是 'LiLei' 开头的索引，然后拿这些索引对应的主键逐个回表，到主键索引上找出相应的记录，再比对age和position这两个字段的值是否符合。
  
MySQL 5.6引入了索引下推优化，可以在索引遍历过程中，对索引中包含的所有字段先做判断，过滤掉不符合条件的记录之后再回表，可以有效的减少回表次数。使用了索引下推优化后，上面那个查询在联合索引里匹配到名字是 'LiLei' 开头的索引之后，同时还会在索引里过滤age和position这两个字段，拿着过滤完剩下的索引对应的主键id再回表查整行数据。
  
索引下推会减少回表次数，对于innodb引擎的表索引下推只能用于二级索引，innodb的主键索引（聚簇索引）树叶子节点上保存的是全行数据，所以这个时候索引下推并不会起到减少查询全行数据的效果。  
 

## 为什么范围查找Mysql没有用索引下推优化？ ##
估计应该是Mysql认为范围查找过滤的结果集过大，like KK% 在绝大多数情况来看，过滤后的结果集比较小，所以这里Mysql选择给 like KK% 用了索引下推优化，当然这也不是绝对的，有时like KK% 也不一定就会走索引下推。

## SQL的执行过程 ##
客户端 -> 连接器(管理连接与权限校验) -> 词法分析器(词法分析,语法分析) -> 优化器(执行计划生成索引选择) -> 执行器(调用引擎接口,获取查询结果) -> 引擎层(读写磁盘,数据结构化存储的实现)

## cost成本计算,看是否需要走索引 ##
mysql最终是否选择走索引或者一张表涉及多个索引，mysql最
终如何选择索引，我们可以用**trace工具**来一查究竟，开启trace工具会影响mysql性能，所以只能临时分析sql使用，用
完之后立即关闭.  
**trace工具用法：**

		
	mysql> set session optimizer_trace="enabled=on",end_markers_in_json=on; ‐‐开启trace
	mysql> select * from employees where name > 'a' order by position;
	mysql> SELECT * FROM information_schema.OPTIMIZER_TRACE;
	
	查看trace字段：
	{
	"steps": [
	{
	"join_preparation": { ‐‐第一阶段：SQL准备阶段，格式化sql
	"select#": 1,
	"steps": [
	{
	"expanded_query": "/* select#1 */ select `employees`.`id` AS `id`,`employees`.`name` AS `name`,`empl
	oyees`.`age` AS `age`,`employees`.`position` AS `position`,`employees`.`hire_time` AS `hire_time` from
	`employees` where (`employees`.`name` > 'a') order by `employees`.`position`"
	}
	] /* steps */
	} /* join_preparation */
	},
	{
	"join_optimization": { ‐‐第二阶段：SQL优化阶段
	"select#": 1,
	"steps": [
	{
	"condition_processing": { ‐‐条件处理
	"condition": "WHERE",
	"original_condition": "(`employees`.`name` > 'a')",
	"steps": [
	{
	"transformation": "equality_propagation",
	"resulting_condition": "(`employees`.`name` > 'a')"
	},
	{
	"transformation": "constant_propagation",
	"resulting_condition": "(`employees`.`name` > 'a')"
	},
	{
	"transformation": "trivial_condition_removal",
	"resulting_condition": "(`employees`.`name` > 'a')"
	}
	] /* steps */
	} /* condition_processing */
	},
	{
	"substitute_generated_columns": {
	} /* substitute_generated_columns */
	},
	{
	"table_dependencies": [ ‐‐表依赖详情
	{
	"table": "`employees`",
	"row_may_be_null": false,
	"map_bit": 0,
	"depends_on_map_bits": [
	] /* depends_on_map_bits */
	}
	] /* table_dependencies */
	},
	{
	"ref_optimizer_key_uses": [
	] /* ref_optimizer_key_uses */
	},
	{
	"rows_estimation": [ ‐‐预估表的访问成本
	{
	"table": "`employees`",
	"range_analysis": {
	"table_scan": { ‐‐全表扫描情况
	"rows": 10123, ‐‐扫描行数
	"cost": 2054.7 ‐‐查询成本
	} /* table_scan */,
	"potential_range_indexes": [ ‐‐查询可能使用的索引
	{
	"index": "PRIMARY", ‐‐主键索引
	"usable": false,
	"cause": "not_applicable"
	},
	{
	"index": "idx_name_age_position", ‐‐辅助索引
	"usable": true,
	"key_parts": [
	"name",
	"age",
	"position",
	"id"
	] /* key_parts */
	}
	] /* potential_range_indexes */,
	"setup_range_conditions": [
	] /* setup_range_conditions */,
	"group_index_range": {
	"chosen": false,
	"cause": "not_group_by_or_distinct"
	} /* group_index_range */,
	"analyzing_range_alternatives": { ‐‐分析各个索引使用成本
	"range_scan_alternatives": [
	{
	"index": "idx_name_age_position",
	"ranges": [
	"a < name" ‐‐索引使用范围
	] /* ranges */,
	"index_dives_for_eq_ranges": true,
	"rowid_ordered": false, ‐‐使用该索引获取的记录是否按照主键排序
	"using_mrr": false,
	"index_only": false, ‐‐是否使用覆盖索引
	"rows": 5061, ‐‐索引扫描行数
	"cost": 6074.2, ‐‐索引使用成本
	"chosen": false, ‐‐是否选择该索引
	"cause": "cost"
	}
	] /* range_scan_alternatives */,
	"analyzing_roworder_intersect": {
	"usable": false,
	"cause": "too_few_roworder_scans"
	} /* analyzing_roworder_intersect */
	} /* analyzing_range_alternatives */
	} /* range_analysis */
	}
	] /* rows_estimation */
	},
	{
	"considered_execution_plans": [
	{
	"plan_prefix": [
	] /* plan_prefix */,
	"table": "`employees`",
	"best_access_path": { ‐‐最优访问路径
	"considered_access_paths": [ ‐‐最终选择的访问路径
	{
	"rows_to_scan": 10123,
	"access_type": "scan", ‐‐访问类型：为scan，全表扫描
	"resulting_rows": 10123,
	"cost": 2052.6,
	"chosen": true, ‐‐确定选择
	"use_tmp_table": true
	}
	] /* considered_access_paths */
	} /* best_access_path */,
	"condition_filtering_pct": 100,
	"rows_for_plan": 10123,
	"cost_for_plan": 2052.6,
	"sort_cost": 10123,
	"new_cost_for_plan": 12176,
	"chosen": true
	}
	] /* considered_execution_plans */
	},
	{
	"attaching_conditions_to_tables": {
	"original_condition": "(`employees`.`name` > 'a')",
	"attached_conditions_computation": [
	] /* attached_conditions_computation */,
	"attached_conditions_summary": [
	{
	"table": "`employees`",
	"attached": "(`employees`.`name` > 'a')"
	}
	] /* attached_conditions_summary */
	} /* attaching_conditions_to_tables */
	},
	{
	"clause_processing": {
	"clause": "ORDER BY",
	"original_clause": "`employees`.`position`",
	"items": [
	{
	"item": "`employees`.`position`"
	}
	] /* items */,
	"resulting_clause_is_simple": true,
	"resulting_clause": "`employees`.`position`"
	} /* clause_processing */
	},
	{
	"reconsidering_access_paths_for_index_ordering": {
	"clause": "ORDER BY",
	"steps": [
	] /* steps */,
	"index_order_summary": {
	"table": "`employees`",
	"index_provides_order": false,
	"order_direction": "undefined",
	"index": "unknown",
	"plan_changed": false
	} /* index_order_summary */
	} /* reconsidering_access_paths_for_index_ordering */
	},
	{
	"refine_plan": [
	{
	"table": "`employees`"
	}
	] /* refine_plan */
	}
	] /* steps */
	} /* join_optimization */
	},
	{
	"join_execution": { ‐‐第三阶段：SQL执行阶段
	"select#": 1,
	"steps": [
	] /* steps */
	} /* join_execution */
	}
	] /* steps */
	}
	
	结论：全表扫描的成本低于索引扫描，所以mysql最终选择全表扫描
	
	mysql> select * from employees where name > 'zzz' order by position;
	mysql> SELECT * FROM information_schema.OPTIMIZER_TRACE;
	
	查看trace字段可知索引扫描的成本低于全表扫描，所以mysql最终选择索引扫描
	
	mysql> set session optimizer_trace="enabled=off"; ‐‐关闭trace


## 索引设计原则 ##
1. 代码先行,索引后上
2. 联合索引尽量覆盖条件
3. 不要在小基数字段上建立索引
4. 长字符串我们可以采用前缀索引
5. where与order by冲突时优先where
6. 基于慢sql查询做优化

## Mysql的表关联常见两种算法 ##
1. 嵌套循环连接 Nested-Loop Join(NLJ) 算法
2. 基于块的嵌套循环连接 Block Nested-Loop Join(BNL)算法

## 分页查询优化 ##
1. 根据自增且连续的主键排序的分页查询

		select * from employees where id > 90000 limit 5;

2. 根据非主键字段排序的分页查询

		select * from employees e inner join (select id from employees order by name limit 90000,5) ed on e.id = ed.id;

## count(*)查询优化 ##
1. 字段有索引：count(*)≈count(1)>count(字段)>count(主键 id) //字段有索引，count(字段)统计走二级索引，二级索引存储数据比主键索引少，所以count(字段)>count(主键 id)
2. 字段无索引：count(*)≈count(1)>count(主键 id)>count(字段) //字段没有索引count(字段)统计走不了索引，count(主键 id)还可以走主键索引，所以count(主键 id)>count(字段)

> count(1)跟count(字段)执行过程类似，不过count(1)不需要取出字段统计，就用常量1做统计，count(字段)还需要取出字段，所以理论上count(1)比count(字段)会快一点。  
count(*) 是例外，mysql并不会把全部字段取出来，而是专门做了优化，不取值，按行累加，效率很高，所以不需要用count(列名)或count(常量)来替代 count(*)。  
为什么对于count(id)，mysql最终选择辅助索引而不是主键聚集索引？因为二级索引相对主键索引存储数据更少，检索性能应该更高，mysql内部做了点优化(应该是在5.7版本才优化)。

## Mysql事务 ##
脏读:事务A读取到事务B已经修改但尚未提交的数据.
不可重复读: 事务A内部的相同查询语句在不同时刻读出结果不一致,不符合隔离性.
幻读定义: 事务A读取到事务B提交的新增数据,不符合隔离性.

MVCC解决脏读和不可重复读
间隙锁可以解决幻读
临键说:行锁与间隙锁的组合.


## MVCC(Multi-Version Concurrency Control) ##
MVCC机制的实现是通过read-view机制与undo版本链比对机制,使得不同的事务会根据数据版本链对比规则读取同一条数据在版本链上的不同版本数据.  
1. undo日志版本链与read view 机制详解  
undo日志版本链是指一行数据被多个事务依次修改过后,在每个事务修改完后,Mysql会保留修改前的数据undo回滚日志,并且用两个隐藏字段trx_id和roll_pointer吧这些undo日志串联起来形成一个历史记录版本链.  
在**可重复度隔离级别**,当事务开启,执行任何查询sql是会生成当前事务的**一致性视图read-view**,该视图在事务结束之前都不会变化(**如果是读已提交隔离级别在每次执行查询sql时都会重新生成**),这个视图由执行查询时所有未提交事务id数组(数组里最小的id为min_id)和已创建的最大事务id(max_id)组成,事务里的任何sql查询结果需要从对应版本链里的最新数据开始逐条跟read-view做比对从而得到最终的快照结果.  

