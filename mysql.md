
# 索引数据结构：二叉树、红黑树、hash、B+树
### 为什么不用二叉树
- 因为如果是数据是顺序递增插入的话，用二叉树就会变成个链表。
### 为什么不用红黑树
- 红黑树其实也是个二叉树，如果数据量上百千万，树的高度会太高
### 为什么不用B树，而用B+树
- B树就是一个节点放很多个索引和data(索引所在行的磁盘文件地址)，索引元素之间会存储下一层的节点的地址。
- B+树其实是在B树的基础上做了优化，第一个优化就是非叶子节点不存储data，只存储冗余索引，叶子节点包含所有索引；第二个优化就是叶子节点之间有双向的指针，可以用于范围查询。
- B+树查询性能好就是因为，从B+树根节点开始，会先low一个节点到内存，这个是比较耗时的，然后在内存里面做数据比对找到要找的数据的下一层的节点地址，这个不是特别耗时。而红黑树一个索引就一个节点，把节点的数据low到内存是特别耗时的。mysql三层的树高就可以存储两千多万的数据。而且mysql可能会把根节点常驻到内存里面，
- 为什么要把data都放到叶子节点，就是因为树的高度是由非叶子节点能放多少个索引决定的，一个节点存越多索引，树的高度就越低，而且节点越少，也可以减少把节点low到内存的时间。
- 为什么要加个双向指针？方便范围查询

```text
比如索引是bigint，占8byte，一个地址是6Byte，那么一个非叶子节点可以存储1170个元素，而叶子节点存的data一般不超过1k，那一个叶子节点就可以存储16个数据，那三层的树，就可以存储1170*1170*16=两千多万
```
### 聚簇索引、稀疏索引是什么
- myisam的索引结构：表的底层存储是frm（表结构信息）、myd（data）、myi（index）三个文件，myd存的是数据，myi存的是索引的B+树。索引和数据是分开存储，这种叫非聚簇索引。索引的B+树的叶子节点的data存的是数据的某一个行的地址。
- innodb的索引结构：表的底层存储是frm（表结构信息）、ibd（data+index）两个文件。ibd只会有一个聚簇索引，就是数据和索引在一起的B+树，数据存在B+树的叶子节点。但是普通索引其实还是非聚簇索引，叶子节点的data存的是主键值，存主键的好处是保证数据一致性和节省存储空间。

### 为什么推荐使用整型自增索引做主键
- 因为如果是字符串，索引查找的就得对字符串索引的每个字符一一比较，非常耗时。
- 如果不是自增的话，往B+树节点的中间位置插入一个数的时候，可能会导致节点分裂、或者做个树平衡。如果是自增的话，就会再往后面开一个节点，不影响原来的节点。

### 联合索引底层数据结构，mysql最左前缀优化原理
- 先按照第一个字段排好序，再按照第二个字段排好序，再按照第三个字段排好序
- mysql最左前缀优化原理就是因为跳过第一个字段，第二个字段是无序的，所以没办法用，只能整个表查询。

# explain详解与使用
- partitions：分区
- filtered：rows*filtered/100估算出将要和explain中前一个表进行连接的行数（前一个表指id值比当前表id值小的表）
- id：有几个select就有几个id，id越大执行优先级越高，id相同则从上往下执行，id为null则最后执行
- select type：表示对应行是简单还是复杂的查询。
  - simple：简单查询，不包含子查询和union
  - primary：最外层的select；
  - subquery：select中的子查询；
  - derived：from中的子查询；mysql会把结果放在一个临时表中，也叫派生表
  - union：union后面的select
- type：mysql如何查找表中的行，从好到差：system>const>req_ref>ref>range>index>all，一般达到range以上，最好达到ref
  - null：在优化阶段分解查询语句，在执行阶段不用遍历表或索引就能直接确定。（比如select min(id) from 表） 
  - const： where后面的字段，用到了primary或者唯一索引，只有一条匹配数据。system是const的特例，表里只有一条数据。
  - req_ref: 被关联表的on的过滤字段是primary或者唯一索引（A left join B，B是被关联表，A是主表）
  - ref：就是用到了普通索引或者唯一性索引的部分前缀，筛选出来的条件会有多条。（可以是简单查询的where条件或者是关联查询的on条件）
  - range：用到了索引，但是是范围查找的。用到了in，between，>,<
  - index：扫描整个二级索引拿到结果。（比如select id,name from 表，name有二级索引）
  - all：扫描整个聚簇索引
- possible key：可能使用哪些索引来查询。
- key：最终用到的索引
- key_len:在索引里使用的字节数。可以算出索引用了哪些列。（char(n)=n字节长度，varchar（utf-8 n）=3n+2字节长度 （2用来存字符串长度），int=4字节长度） 
- ref：在key列记录的索引中，表查找值所用到的列或常量。比如：const（常量）、字段名
- rows：估计要读取或检测的行数
- extra：
  - using index：使用覆盖索引，就是包括where或者select后面的字段都可以直接从索引中获取
  - using where：用了where，并且where或者select后面的列未被索引覆盖
  - using index condition：查询的列不完全被索引覆盖，比如where后面是范围查询某个索引字段
  - using temporary：需要创建一张临时表来处理查询，比如遇到没有索引的数据（select distinct name from 表）
  - using filesort: 用外部排序而不是索引排序，数据小从内存排序，否则从磁盘排序。

# 常见索引优化原则
- 全值匹配
- 最左前缀法则
- 不在索引列上做函数、计算、类型转换处理
- 所有a列是范围，b列会失效
- 尽量使用覆盖所有，就是查询列在索引列中，减少select *
- 使用!=会导致全表扫描
- is null 或者is not null一般情况走不了索引
- like "%aa"，会导致索引失效
- 少用or或者in，不一定会走索引
- 范围查询不一定会走索引
# mysql的内部组件结构
<img width="711"  src="https://user-images.githubusercontent.com/27798171/229345624-de795209-3fb2-46d1-9247-05083894c766.png">

- 分为server层和引擎层，客户通首先通过server层的连接器进行连接，如果开启了server层的缓存，会先从缓存获取，这个缓存的key就是sql语句，如果缓存没有，就依次通过server层的词法分析器，优化器，执行器。执行器会去调用引擎接口查询数据。
- server层的缓存：很鸡肋，因为一旦修改了数据，缓存就会被清空，再查询的时候会去磁盘查完数据再放到缓存里面，所以只能适用于配置表或者字段表这种不经常变的数据。

```
my.cnf
query_cache_type = 0关闭 1开启 2遇到select SQL_CACHE * from才走缓存；
show status like “%Qcache%”  //查看运行的缓存信息
```
- 词法分析器：分6个步骤完成对sql语句的分析：
  - 词法分析
  - 语法分析
  - 语义分析
  - 构造执行树
  - 生成执行计划
  - 计划的执行 

<img width="1209"  src="https://user-images.githubusercontent.com/27798171/229351698-baabe3b7-db11-4306-aad1-44313839af34.png">

- binlog：是server层实现的二进制日志。会追加记录curd操作，记录格式分为statement-记录执行的sql、row-记录操作后的行数据（推荐）、mixed混合模式
```
flush logs 建个新的binlog日志，新的日志会放到这里面，老的不动
reset master 清空所有binlog日志

# 查询某个binlog日志
/mysql/bin/mysqlbinlog --no-defaults 某个binlog文件

# 恢复某个binlog的数据
/mysql/bin/mysqlbinlog --no-defaults 某个binlog文件 | mysql -uroot -p 密码

# 恢复某个binlog指定位置的数据
/mysql/bin/mysqlbinlog  --start-position=1609 --stop-position=1822 | mysql -uroot -p 密码

# 恢复指定时间内binlog指定位置的数据
/mysql/bin/mysqlbinlog --no-defaults 某个binlog文件  --start-date="2012-10-15 16:30:00" --stop-date="2012-10-15 17:00:00" | mysql -uroot -p 密码
```
# 索引走不走？
假设index(a,b,c)

- where a=3 and b>4 and c=5 【肯定使用到了a和b索引。 】
- where a=3 and b>=4 and c=5 【mysql索引内部优化：使用到了a和b索引，有可能用到c索引】
- where a=3 and b like “kk%” and c=5  【肯定使用了a和b和c索引。kk%相当于常量，不管表大表小】
- where a=3 and b like “k%kk%” and c=5 【肯定使用了a和b和c索引。k%kk%相当于常量，不管表大表小】
- where a=3 and b like “%kk” and c=5 【可能使用了a索引，%kk相当于范围】
- where a>= 5 and b=4 and c=5 【mysql索引内部优化：如果联合索引第一个字段用了范围，大概率不走索引】
- where a> “aa” 【mysql索引内部优化：大概率不走索引】
- where a> “zz” 【mysql索引内部优化：大概率走索引】
- where a in (4,5,6) and b=4 and c=5 【mysql索引内部优化：数据量大走索引a，b，c；数据量小不走索引】
- where (a=4 or a=5) and b=4 and c=5 【mysql索引内部优化：数据量大走索引a，b，c；数据量小不走索引】


# 索引下推优化详解
假设index(a,b,c), where a like “kk%” and b=4 and c=5, 可能使用到了a，b，c索引，为什么a like "kk%"相当于常量？

- 索引下推是5.6版本之后引入的，5.6之前在遇到a=kk%的时候，先根据a把结果集过滤出来，然后根据叶子结点主键id回表去聚簇索引拿出来所有的结果集，再根据b和c条件去过滤
- 5.6以后每过滤出来一条a like “kk%”的数据，同时还会去比较一下b和c是否满足条件，如果符合才把id拿出来
- 那为什么范围查询不做索引下推？可能是因为like确定出来的结果大多时候会比范围查询少
# trace工具
```
SET SESSION OPTIMIZER_TRACE="enabled=on",END_MARKERS_IN_JSON=on;

先写个自己的sql，后面带这个sql，就能看到trade执行计划：
select * from information_schema.optimizer_trace

第一阶段：sql准备阶段，格式化sql
第二阶段：sql优化
第三阶段：预估表的访问成本
 "rows_estimation": [
              {
                "table": "`emp`",
                "range_analysis": {
                  "table_scan": {
                  	/* 全表扫描预估扫描651300条记录，花费是227957*/
                    "rows": 651300,
                    "cost": 227957
                  },
                  "potential_range_indexes": [
                  	/* 可能用到的索引 */
                    {
                 	  /* 主键索引 =》 不适用*/
                      "index": "PRIMARY",
                      "usable": false,
                      "cause": "not_applicable"
                    },
                    {
                    	/* 联合索引idx_name_age_job =》 适用*/
                      "index": "idx_name_age_job",
                      "usable": true,
                      "key_parts": [
                        "name",
                        "age",
                        "job",
                        "id"
                      ] /* key_parts */
                    }
                  ] /* potential_range_indexes */,
                  "best_covering_index_scan": {
                  	/* 最优选择，使用联合索引idx_name_age_job */
                    "index": "idx_name_age_job",
                    "cost": 73074,
                    "chosen": true
                  } /* best_covering_index_scan */,
                  "setup_range_conditions": [
                  ] /* setup_range_conditions */,
                  "group_index_range": {
                    "chosen": false,
                    "cause": "not_group_by_or_distinct"
                  } /* group_index_range */,
                  "skip_scan_range": {
                    "potential_skip_scan_indexes": [
                      {
                        "index": "idx_name_age_job",
                        "usable": false,
                        "cause": "prefix_not_const_equality"
                      }
                    ] /* potential_skip_scan_indexes */
                  } /* skip_scan_range */,
                  "analyzing_range_alternatives": {
                    "range_scan_alternatives": [
                      {
                        "index": "idx_name_age_job",
                        "ranges": [
                          "hu <= name <= hu AND 20 < age"
                        ] /* ranges */,
                        "index_dives_for_eq_ranges": true,
                        "rowid_ordered": false,
                        "using_mrr": false,
                        "index_only": true,
                        "rows": 1,
                        "cost": 1.11,
                        "chosen": true
                      }
                    ] /* range_scan_alternatives */,
                    "analyzing_roworder_intersect": {
                      "usable": false,
                      "cause": "too_few_roworder_scans"
                    } /* analyzing_roworder_intersect */
                  } /* analyzing_range_alternatives */,
                  "chosen_range_access_summary": {
                    "range_access_plan": {
                      "type": "range_scan",
                      "index": "idx_name_age_job",
                      "rows": 1,
                      "ranges": [
                        "hu <= name <= hu AND 20 < age"
                      ] /* ranges */
                    } /* range_access_plan */,
                    "rows_for_plan": 1,
                    "cost_for_plan": 1.11,
                    "chosen": true
                  } /* chosen_range_access_summary */
                } /* range_analysis */
              }
            ] /* rows_estimation */


```
# 索引优化order by与group by
假设index(a,b,c)

- where a=3 and c=4 order by b 【肯定使用到了a和b索引。 】
- where a=3 order by b,c 【肯定使用到了a、b、c索引。 】
- where a=3 order by c,b 【肯定使用到了a索引。 】
- where a=3 order by b asc, c dsec 【肯定使用到了a索引。mysql8版本和以上有降序索引支持查询 】
- where a in （3，4） order by b,c 【b和c肯定不会走索引，mysql索引内部优化：a不一定走索引 】
- select * from 表 where a > 3 order by a 【mysql索引内部优化：大概率不走索引，因为涉及到回表查询别的列表。 】
- select * from 表 order by a 【ysql索引内部优化：大概率不走索引，因为涉及到回表查询别的列表。】
- select a, b, c from 表 where a > 3 order by a 【mysql索引内部优化：覆盖索引，肯定走a，b，c索引 ，不需要回表】

总结
- MysQL支持两种方式的排序ilesor和index， Using index是指MysQL扫描索引本身完成排序。index效率高，filesor效率低。
- order by满足两种情況会使用Using index。
  - order by语句使用索引最左前列。
  - 使用where子句与order by子句条件列组合满足素引最左前列。
- 尽量在素引1列上完成排序，遵循索引建立（索引创建的顺序）时的最左前缀法则。
- 如果order by的条件不在索引列上，就会产生Using filesort.
- 能用覆盖索引尽量用覆盖索引
- group by与order by很类似，其实质是先排序后分组，遵照索引创建顺序的最左前级法则。对于group by的优化如果不需要排序的可以加上order by nul禁止排序。注意，where高于having，能写在where中的限定条件就不要去having限定了。

# using filesort 文件排序详解
filesor文件排序方式
- 单路排序：是-次性取出满足条件行的所有字段，然后在sort buffer中进行排序;
  - 比如 没有索引，select a,b,c,d,e where a=3 order by b,c;会根据where条件查询满足的行，每行的a,b,c,d,e一起进入到排序缓存中根据b,c排序。
  - 用trace工具可以看到sortmode信息里显示<sort_key, additional_fields>或者<sort_key, packed_additional_fields>
- 双路排序（又叫回表排序模式）：是首先根据相应的条件取出相应的排序字段和可以直接定位行数据的行ID，然后在 sort buffer 中进行排序，排序完后需要再次取回其它需要的字段。
  - 比如 没有索引，select a,b,c,d,e where a=3 order by b,c;会根据where条件查询满足的行，每行的id，b,c一起进入到排序缓存中根据b,c排序,再根据id回表聚簇索引拿到d、e字段
  - 用tace土具可以看到sort mode信息里显示<sort_key,rowid>
- MySQL 通过比较系统变量max_lenath_for_sort_data(默认1024字节）的大小和需要查询的字段总大小来判断使用哪种排序模式。如果字段的总长度小于max_lenath_for_sort_data，那么使用单路排序模式；如果字段的总长度大于max_lenath_for_sort_data，那么使用双路排序模式。

# 分页查询优化详解
- limit m,n (从第m行的后面开始，往后取n个数字)，会导致从1一直扫描到m+n行数据，然后排期前m行，所以想要查一张大表比较靠后的数据，效率会很低。
  - 优化方案1：根据自增且连续的主键排序的分页查询，改成【where id > m limit n】 。缺点：要求连续太苛刻
  - 优化方案2：根据非主键字段排序的分页查询，改成【order by 索引字段 limit m,n】。缺点：最好select 后面的字段可以覆盖索引，不然mysql大概率不走索引，而是全表查询
  - 优化方案3：基于方案2可以先把id查出来，再来回表，控制走索引。改成【select * from table t1 inner join (select id from table order by 索引字段 limit m,n) t2 on t1.id = t2.id 】。缺点：只适合n比较小的情况
  -

# 表join关联原理详解与优化
### 嵌套循环连接Nested-Loop JOIN（NLJ）算法
假设A是大表，有1万行，有e索引。B是小表，有1百行，有e索引。执行explain select * from A inner join B on A.e = B.e

- 驱动表：B是驱动表（先执行的表就是驱动表），A是被驱动表。
- 执行原理：mysql一般优先从小表B中取出满足条件的一行数据，然后取出e到大表A中查找满足条件的数据，然后把刚从B中取出的那行数据和从A表中查出的数据做合并追加到结果列表。然后继续重复这三步骤。
- 扫描次数：整个过程会扫描1次小表B的聚簇索引，扫描出100行。大表A的e索引会扫描100次，扫描出100行
- 如果被驱动表的关联字段没有索引，用NJL算法性能会比较低，mysql会选择BNL算法
### 基于块的嵌套循环连接算法Block Nested-Loop JOIN（BNL）算法
假设A是大表，有1万行， e没有索引。B是小表，有1百行，e没有索引。执行explain select * from A inner join B on A.e = B.e

- 驱动表：B是驱动表（先执行的表就是驱动表），A是被驱动表。
- 执行原理：把小表B的所有数据字段放到join_buffer中，把大表A中的每一行数据取出来跟join_buffer中的数据都比较一下找到B中满足条件的数据，然后把从B中满足条件的数据和A刚拿出来的那一行数据做合并追加到结果列表。然后继续重复这三步骤。
- 扫描次数：整个过程会扫描1次小表B的聚簇索引，扫描出100行。扫描1次大表A的聚簇索引，扫描出10000行。join_buffer里面的数据是无序的，大表A中每次取出一行来和join_buffer判断的时候，内存都得做100次判断，所以内存的判断次数是100*10000=1百万。如果改成用NLJ算法，需要磁盘扫描1百万次，而BNL只需要磁盘扫描100+10000次+内存判断1百万次，内存判断速度很快。
- 如果B表也是大表，join_buffer放不下怎么办？（方法一）join_buffer 默认256kb，可以由参数join buffer size调整大小 （方法二）分段放到join buffer，分段多少个就得扫描几次大表A

### 关联sql优化
- 尽量给被驱动表的关联字段加索引，走NLJ算法
- 小表驱动大表
- in： 会先走in 后面的sql 语句，再走in前面的sql语句，所以尽量in后面的sql语句是小表
- exsit：会先走exist前面的sql语句，再走exist后面的sql语句，所以尽量exist前面的语句是小表，exist查询往往可以用join代替
# 表count查询优化
count(*)≈count(1)>count(索引字段)>count(id)>count(无索引字段)
- count(1)不需要取出字段，就用常量1统计有多少个
- count(字段)还得取出字段，注意！！不会统计字段为null的数据
- count(*)不需要取出字段，按行累加，效率很高
- count(id)最终mysql如果有辅助索引，一定会优先走辅助索引，因为辅助索引比主键索引存储数据更少，检索性能更高
- 如果表很大，不管用哪种count，性能都会差
# mysql 数据类型选择分析

<img  alt="11" src="https://user-images.githubusercontent.com/27798171/233767423-89a22ab3-479d-4d0f-8b24-db793826e802.png">
<img  alt="22" src="https://user-images.githubusercontent.com/27798171/233767747-d8c61e44-03e1-4c91-8e2b-3d4e2c4a535d.png">

# mysql事务
为了解决多个事务并发问题，数据库设计了事务隔离机制、锁机制、mvcc多版本并发控制隔离机制。注意myIsame不支持事务、不支持行锁，innodb支持事务、行锁。
### 事务的ACID属性
- 原子性：对数据的修改要么都执行，要么都不执行。
- 一致性：在数据开始和完成时，数据都必须保持一致状态。
- 隔离性：事务在不受外部并发操作影响的独立环境执行。
- 持久性：事务完成之后，它对数据的修改是永久性的。
### 并发事务处理带来的问题
- 脏写/更新丢失：两个事务都读到库存为10，一个事务扣减2改成8，一个事务扣减3改成7，导致最后的更新覆盖了其他事务做的更新。
- 脏读：事务A读取了事务B已经修改但尚未提交的数据。
- 不可重读（默认）：事务A在相同查询语句在不同时刻查出来的结果不一样（更新）或者少了（被删了）
- 幻读：事务A在相同查询语句在不同时刻查出来的结果多了别的事务插入的数据。（针对插入）。【可重复读环境隔离级别：对于事务A先查询了id>1的数据，事务b插入了id=2，事务A想插入id=2的时候，发现已经存在了，A还可以更新id=2的数据】
### mysql事务隔离级别
- 读未提交：会导致脏读、不可重读、幻读。
- 读已提交：解决了脏读。
- 可重复读：解决了脏读、不可重读。但是会导致幻读
- 可串行化：解决了脏读、不可重读、幻读。避免幻读的方法：(1)查询语句会对涉及的行加写锁。(2)如果执行的是一个范围查询，那么范围内的所有行和行记录所在的间隙区间范围都会被加锁。
# mysql锁
### mysql 锁类型
- 性能上区分：乐观锁、悲观锁
- 操作类型上区分：读锁、写锁、意向锁（都属于悲观锁）。读锁也叫共享锁（S锁），针对同一份数据，支持多个读操作，不支持写操作。写锁也叫排他锁（X锁），针对同一份数据，不支持别人读或写操作。意向锁：给数据行加读锁或者写锁的时候，会给表加个标识，代表有行锁了。方便表锁的时候不用扫描全表。
  - myIsame在执行查询语句前，会自动给涉及的表加读锁，在执行update、delete、insert前，会自动给涉及的表加写锁。
  - innodb在执行查询语句前，因为有mvcc机制，不会加锁。在执行update、delete、insert前，会自动给涉及的行加锁。
- 粒度上区分：表锁、行锁、页锁。表锁一般用于数据迁移。
  - 表锁：对整个表加共享锁之前，需要先获取到意向共享锁。对整个表加排他锁之前，需要先获取到意向排他锁。
  - 页锁：只有BDB存储引擎支持页锁
  - 行锁：myIsame不支持行锁，innodb支持行锁。InnoDB的行锁实际上是针对索引加的锁(在索引对应的索引项上做标记)，不是针对整个行记录加的锁。并且该索引不能失效，否则会从行锁升级为表锁。(可重复读级别会升级为表锁，读已提交级别不会升级为表锁)
- 间隙锁：间隙锁是在可重复读隔离级别下才会生效，检索条件必须走索引，不然更新语句会导致表锁。行数加间隙锁就是临键锁
<img width="892" alt="截屏2023-05-03 下午8 40 32" src="https://user-images.githubusercontent.com/27798171/235918758-18b65cd2-70b2-4efe-9cda-a535dbefaafa.png">


```sql

=============悲观锁、读锁===============
select * from 表名 where id=100 lock in share mode
============悲观锁、写锁===============
# 查询数据的时候就上锁，不允许别人写或读操作
select * from 表名 where id = 1 for update
============乐观锁===============
# 更新时判断原来的值
update 表名 set version = 2 where id = 1 and version = 1
============表锁===============
# 手动增加表锁
lock table 表名1 READ, 表名2 WRITE

# 查看表上加过的锁
show open tables;

# 删除表锁
unlock tables;

```
### 行锁分析
- 分析系统上的行锁的争夺情况: show status like "innodb_row_lock%":
- 对各个状态量的说明如下：
  - Innodb_row_lock_current_waits: 当前正在等待锁定的数量
  - Innodb_row_lock_time：从系统启动到现在锁定总时间长度（等待总时长）
  - Innodb_row_lock_time_avg: 每次等待所花平均时间（等待平均时长）
  - Innodb_row_lock_time_max：从系统启动到现在等待最长的一次所花时间
  - Innodb_row_lock_waits:系统启动后到现在总共等待的次数（等待总次数）

### 查看INFORMATION_ SCHEMA系统库锁相关数据表
```sql
-- 查看事务
select * from INFORMATION_SCHEMA.INNODB_TRX:
-- 查看锁
select * from INFORMATION_SCHEMA.INNODB_LOCKS；
-- 查看锁等待
select * from INFORMATION_SCHEMA.INNODB_LOCK_WAITS;
-- 释放锁,trx_mysql_thread_id可以从INNODB_TRX中找到
kill trx_mysql_thread_id
```

### mysql锁优化建议
- 尽可能让所有数据检索都通过索引来完成，避免无索引行锁升级为表锁
- 合理设计索引，尽量缩小锁的范围
- 尽可能减少检索条件范围，避免间隙锁
- 尽量控制事务大小，减少锁定资源量和时间长度，涉及事务加锁的sql尽量放在事务最后执行
- 尽可能低级别事务隔离
# mvcc多版本并发控制机制
- 在读已提交和可重复读环境隔离级别中，对一行数据的读和写两个操作默认是不会通过加锁来保证隔离性的，而是靠mvcc机制，避免了频繁加锁互斥。
- MVCC机制的实现就是通过read-view机制与undo版本链比对机制，使得不同的事务会根据数据版本链对比规则读取同一条数据在版本链上的不同版本数据。
### undo日志版本链
一行数据被事务修改后，会保留修改前的数据的undo回滚日志，并且用两个隐藏字段trx_id事务id和roll_pointer回滚指针把这些undo日志串联起来形成一个历史记录版本链。
<img width="721"  src="https://user-images.githubusercontent.com/27798171/235947010-90fdce4f-c7f8-4bac-a96e-6d1d4aa3ee2d.png">

### read-view 一致性视图
- 在可重复读隔离级别，当事务开启，执行任何查询sql时会生成当前事务的一致性视图read-view，该视图在事务结束之前都不会变化(如果是读已提交隔离级别在每次执行查询sql时都会重新生成）
- 这个read-view视图由执行查询时所有未提交事务id数组（数组里最小的id为min_id)和包创建的最大事务id (max_id) 组成，事务里的任何sql查询结果需要从对应undo日志版本链里的最新数据开始逐条跟read-view做比对从而得到最终的快照结果。

### 版本链比对规则
- 如果row 的trx_id 落在(trx_id<min_id)，表示这个版本是已提交的事务生成的，这个数据是可见的；
- 如果row 的trx_id 落在(trx_id>max_id )，表示这个版本是由将来启动的事务生成的，是肯定不可见的；
- 如果row 的trx_id 落在(min_id <=trx_id<=max_id)，那就包括两种情况
  - a.若row 的trx id 在视图数组中，表示这个版本是由还没提交的事务生成的，不可见，若row 的trx_id 就是当前自己的事务是可见的；
  - b.若row 的trx id 不在视图数组中，表示这个版本是已经提交了的事务生成的，可见。
- 对于删除的情况可以认为是update的特殊情况，会将版本链上最新的数据复制一份，然后将trx_id修改成删除操作的trx_id，同时在该条记录的头信息(record header)里的(deleted_fag）标记位写上true，来表示当前记录已经被删除，在查询时按照上面的规则查到对应的记录如果delete_flag标记位为true，意味着记录已被删除，则不返回数据。
- 注意：begin/start transaction 命令并不是一个事务的起点，在执行到它们之后的第一个修改操作linnoDB表的语句，事务才真正启动，才会向mysal申请事务id，mysql内部是严格按照事务的启动顺序来分配事务id的。
# mysql的内部组件结构
<img  src="https://github.com/niqinhua/java-interview-draft/assets/27798171/de25a238-44ec-42d4-b36f-470712b6a06f">

### MySQL 可以分为 Server 层和存储引擎层两部分
- service层：主要包括连接器、查询缓存、分析器、优化器、执行器等，涵盖 MySQL 的大多数核心服务功能，以及所有的内置函数（如日期、时间、数学和加密函数等），所有跨存储引擎的功能都在这一层实现，比如存储过程、触发器、视图等。
- 存储引擎层：存储引擎层负责数据的存储和提取。其架构模式是插件式的，支持 InnoDB、MyISAM、Memory 等多个存储引擎。现在最常用的存储引擎是 InnoDB，它从 MySQL 5.5.5 版本开始成为了默认存储引擎
### 连接器
- 连接器负责跟客户端建立连接、获取权限、维持和管理连接。客户端的连接命令一般是这么写的：mysql -h 数据库地址 -u 用户 -p 密码 -P 3306
- 在完成经典的 TCP 握手后，连接器就要开始认证你的身份并查询权限。连接以后修改了权限，只有再新建的连接才会使用新的权限设置。
### 查询缓存
- MySQL 拿到一个查询请求后，会先到查询缓存看看，之前是不是执行过这条语句。之前执行过的语句及其结果可能会以 key-value 对的形式，被直接缓存在内存中。key 是查询的语句，value 是查询的结果。
- 如果语句不在查询缓存中，就会继续后面的执行阶段。执行完成后，执行结果会被存入查询缓存中。
- 大多数情况查询缓存就是个鸡肋，为什么呢？只要有对一个表的更新，这个表上所有的查询缓存都会被清空。一般建议大家在静态表里使用查询缓存，什么叫静态表呢？就是一般我们极少更新的表。
- 开启和关闭缓存配置的地方：my.cnf文件的 query_cache_type=0 （0代表关闭查询缓存OFF，1代表开启ON，2（DEMAND）代表当sql语句中有SQL_CACHE关键词时才缓存）
词时才缓存）
- 显式指定从缓存查询：select SQL_CACHE * from test where ID=5；
- mysql 8.0已经移除了查询缓存功能
### 分析器
- 首先，MySQL 需要知道你要做什么，因此需要对 SQL 语句做解析。分析器先会做“词法分析”。你输入的是由多个字符串和空格组成的一条 SQL 语句，MySQL 需要识别出里面的字符串分别是什么，代表什么。
- 做完了这些识别以后，就要做“语法分析”。根据词法分析的结果，语法分析器会根据语法规则，判断你输入的这个 SQL 语句是否满足 MySQL 语法。
- 下图是分析器对sql的分析过程步骤：
<img  src="https://github.com/niqinhua/java-interview-draft/assets/27798171/1859ecd3-5500-4226-a040-a7cf49ed5042">
- SQL语句经过分析器分析之后，会生成一个这样的语法树
<img  src="https://github.com/niqinhua/java-interview-draft/assets/27798171/65441955-fe93-4cce-a66d-304dff9d0ba3">

### 优化器
- 优化器是在表里面有多个索引的时候，决定使用哪个索引；或者在一个语句有多表关联（join）的时候，决定各个表的连接顺序；以及一些mysql自己内部的优化机制

### 执行器
开始执行的时候，要先判断一下你对这个表 T 有没有执行查询的权限，如果没有，就会返回没有权限的错误，如下所示 (在工程实现上，如果命中查询缓存，会在查询缓存返回结果的时候，做权限验证)。如果有权限，就打开表继续执行。打开表的时候，执行器就会根据表的引擎定义，去使用这个引擎提供的接口。

#### innodb引擎bufferPool缓存机制详解

<img  src="https://user-images.githubusercontent.com/27798171/235950668-06d722b4-f2e3-4e0d-8480-3aeddec4e703.png">

更新一个sql语句的流程
- 1.从idb磁盘文件加载id为1的记录的整页数据到buffer pool缓存池
- 2.写入更新数据的旧值undo日志方便回滚 
- 3.更新buffer pool 的内存数据
- 4.写redo日志到buffer中
- 5.顺序写redo日志到磁盘中，准备提交事务
- 6.service层写binlog日志到磁盘
- 7. 写入commimit标记到redo日志磁盘文件中，提交事务完成。该标记为了保证事务提交后redo和binlog数据一致
- 8. 会有个IO线程在系统空闲时把buffer pool中的数据随机写入磁盘，以page为单位写入

为什么Mysql不能直接更新磁盘上的数据而设置这么一套复杂的机制来执行SQL了？
- 因为来一个请求就直接对磁盘文件进行随机读写，然后更新磁盘文件里的数据性能可能相当差。因为磁盘随机读写的性能是非常差的，所以直接更新磁盘文件是不能让数据库抗住很高并发的。
- Mysql这套机制看起来复杂，但它可以保证每个更新请求都是更新内存BufferPool，然后顺序写日志文件，同时还能保证各种异常情况下的数据一致性。更新内存的速度很快，然后顺序写磁盘上的日志文件的速度也很快，远高于随机读写磁盘文件。

#### redo log重做日志
- redo log 从头开始写，写完一个文件继续写另一个文件，写到最后一个文件末尾就又回到第一个文件开头循环写
- redo log 的写入策略：innodb_flush_log_at_trx_commit=1
  - 设置为0：表示每次事务提交时都只是把 redo log 留在 redo log buffer 中，数据库宕机可能会丢失数据
  - 设置为1(默认值)：表示每次事务提交时都将 redo log 直接持久化到磁盘，数据最安全，不会因为数据库宕机丢失数据，但是效率稍微差一点，线上系统推荐这个设置。
  - 设置为2：表示每次事务提交时都只是把 redo log 写到操作系统的缓存page cache里，这种情况如果数据库宕机是不会丢失数据的，但是操作系统如果宕机了，page cache里的数据还没来得及写入磁盘文件的话就会丢失数据。
  - InnoDB 有一个后台线程，每隔 1 秒，就会把 redo log buffer 中的日志，调用 操作系统函数 write 写到文件系统的 page cache，然后调用操作系统函数 fsync 持久化到磁盘文件。

<img  src="https://github.com/niqinhua/java-interview-draft/assets/27798171/58dc2468-2ccf-412d-861e-d02a0ad07936">

- 相关redo log文件的配置
  - innodb_log_buffer_size：设置redo log buffer大小参数，默认16M ，最大值是4096M，最小值为1M。
  - innodb_log_file_size：设置单个redo log文件大小，默认值为48M。最大值为512G，注意最大值指的是整个 redo log系列文件之和，即(innodb_log_files_in_group * innodb_log_file_size)不能大于最大值512G。
  - innodb_log_files_in_group：设置redo log文件的个数，命名方式如: ib_logfile0, iblogfile1... iblogfileN。默认2个，最大100个

#### binlog二进制归档日志
- binlog二进制日志记录保存了所有执行过的修改操作语句，不保存查询操作。如果 MySQL 服务意外停止，可通过二进制日志文件排查，用户操作或表结构操作，从而来恢复数据库数据。
启动binlog记录功能，会影响服务器性能，但如果需要恢复数据或主从复制功能，则好处则大于对服务器的影响。
- MySQL5.7 版本中，binlog默认是关闭的，8.0版本默认是打开的。
- binlog 的日志格式：用参数 binlog_format 可以设置binlog日志的记录格式，mysql支持三种格式类型：
  - STATEMENT：基于SQL语句的复制，每一条会修改数据的sql都会记录到master机器的bin-log中，这种 方式日志量小，节约IO开销，提高性能，但是对于一些执行过程中才能确定结果的函数，比如UUID()、SYSDATE()等函数如果随sql同步到slave机器去执行，则结果跟master机器执行的不一样。
  - ROW：基于行的复制，日志中会记录成每一行数据被修改的形式，然后在slave端再对相同的数据进行修改记录下每一行数据修改的细节，可以解决函数、存储过程等在slave机器的复制问题，但这种方式日志量较大，性能不如Statement。举个例子，假设update语句更新10行数据，Statement方式就记录这条update语句，Row方式会记录被修改的10行数据。
  - MIXED：混合模式复制，实际就是前两种模式的结合，在Mixed模式下，MySQL会根据执行的每一条具体的sql语句来区分对待记录的日志形式，也就是在Statement和Row之间选择一种，如果sql里有函数或一些在执行时才知道结果的情况，会选择Row，其它情况选择Statement，推荐使用这一种。
- binlog写入磁盘机制： sync_binlog=0
  - 为0的时候，表示每次提交事务都只 write 到page cache，由系统自行判断什么时候执行 fsync 写入磁盘。虽然性能得到提升，但是机器宕机，page cache里面的 binlog 会丢失。
  -  也可以设置为1，表示每次提交事务都会执行 fsync 写入磁盘，这种方式最安全。
  -  还有一种折中方式，可以设置为N(N>1)，表示每次提交事务都write 到page cache，但累积N个事务后才 fsync 写入磁盘，这种如果机器宕机会丢失N个事务的binlog。
- 发生以下任何事件时, binlog日志文件会重新生成：（1）服务器启动或重新启动（2）服务器刷新日志，执行命令flush logs（3）日志文件大小达到 max_binlog_size 值，默认值为 1GB
- binlog 相关配置my.cnf
```
# log‐bin设置binlog的存放位置，可以是绝对路径，也可以是相对路径，这里写的相对路径，则binlog文件默认会放在data数据目录下
log‐bin=mysql‐binlog

# Server Id是数据库服务器id，随便写一个数都可以，这个id用来在mysql集群环境中标记唯一mysql服务器，集群环境中每台mysql服务器的id不能一样，不加启动会报错
server‐id=1

# 其他配置
binlog_format = row # 日志文件格式
expire_logs_days = 15 # 执行自动删除binlog日志文件的天数， 默认为0， 表示不自动删除
max_binlog_size = 200M # 单个binlog日志文件的大小限制，默认为 1GB
```
#### undo log回滚日志
- InnoDB对undo log文件的管理采用段的方式，也就是回滚段（rollback segment） 。每个回滚段记录了 1024 个 undo log segment ，每个事务只会使用一个undo log segment。在MySQL5.5的时候，只有一个回滚段，那么最大同时支持的事务数量为1024个。在MySQL 5.6开始，InnoDB支持最大 128个回滚段，故其支持同时在线的事务限制提高到了 128*1024 。

- undo log日志什么时候删除？
  - 新增类型的，在事务提交之后就可以清除掉了。
  - 修改类型的，事务提交之后不能立即清除掉，这些日志会用于mvcc。只有当没有事务用到该版本信息时才可以清除。

#### 错误日志
它记录了数据库启动和停止，以及运行过程中发生任何严重错误时的相关信息。当数据库出现任何故障导致无法正常使用时，建议首先查看此日志。
在MySQL数据库中，错误日志功能是默认开启的，而且无法被关闭
#### 通用查询日志
- 通用查询日志记录用户的所有操作，包括启动和关闭MySQL服务、所有用户的连接开始时间和截止时间、发给 MySQL 数据库服务器的所有 SQL 指令等，如select、show等，无论SQL的语法正确还是错误、也无论SQL执行成功还是失败，MySQL都会将其记录下来。通用查询日志用来还原操作时的具体场景，可以帮助我们准确定位一些疑难问题，比如重复支付等问题。
- general_log：是否开启日志参数，默认为OFF，处于关闭状态，因为开启会消耗系统资源并且占用磁盘空间。一般不建议开启，只在需要调试查询问题时开启。

# mysql8.0新特性
### 支持索引一个字段升序，一个字段降序
create table t1(c1 int,c2 int,index idx_c1_c2(c1,c2 desc));
### group by 不再隐式排序
 - mysql 5.7 使用groud by的时候会对group by的字段进行正序排序；
 - mysql 8.0 对于group by 字段不再隐式排序，如需要排序，必须显式加上order by 子句。
### 增加隐藏索引
- 使用 invisible 关键字在创建表或者进行表变更中设置索引为隐藏索引。索引隐藏只是不可见，但是数据库后台还是会维护隐藏索引的，在查询时优化器不使用该索引，即使用force index，优化器也不会使用该索引，同时优化器也不会报索引不存在的错误，因为索引仍然真实存在，必要时，也可以把隐藏索引快速恢复成可见。注意，主键不能设置为 invisible。
- 软删除就可以使用隐藏索引，比如我们觉得某个索引没用了，删除后发现这个索引在某些时候还是有用的，于是又得把这个索引加回来，如果表数据量很大的话，这种操作耗费时间是很多的，成本很高，这时，我们可以将索引先设置为隐藏索引，等到真的确认索引没用了再删除。
```sql
-- 创建隐藏索引
 create table t2(c1 int, c2 int, index idx_c1(c1), index idx_c2(c2) invisible);

-- 在查询时优化器不使用该索引，即使用force index，优化器也不会使用该索引
 explain select * from t2 where c2=1; ‐‐隐藏索引c2不会被使用,key=NULL

-- 查询表的所有索引（包括隐藏索引）
 show index from t2

-- 在回话级别看到隐藏索引并使用隐藏索引
set session optimizer_switch="use_invisible_indexes=on"
explain select * from t2 where c2=1; ‐‐隐藏索引c2会被使用

-- 把隐藏索引改为可见
alter table t2 alter index idx_c2 visible;

-- 把可见索引改为隐藏索引
alter table t2 alter index idx_c2 invisible;
```
### 函数索引
- 之前我们知道，如果在查询中加入了函数，索引不生效，所以MySQL 8引入了函数索引，MySQL 8.0.13开始支持在索引中使用函数(表达式)的值。
- 函数索引基于虚拟列功能实现，在MySQL中相当于新增了一个列，这个列会根据你的函数来进行计算结果，然后使用函数索引的时候就会用这个计算后的列作为索引。

```sql
create table t3(c1 varchar(10),c2 varchar(10));
create index idx_c1 on t3(c1); ‐‐创建普通索引
-- 使用函数索引举例
create index func_idx on t3((UPPER(c2))); ‐‐创建一个大写的函数索引
explain select * from t3 where upper(c2)='ZHUGE'; ‐‐使用了函数索引 func_idx
```

### innodb存储引擎select for update跳过锁等待
- 在5.7及之前的版本，select...for update，如果获取不到锁，会一直等待，直到innodb_lock_wait_timeout（行锁锁定时间默认50秒）超时。
- 在8.0版本，通过添加nowait，skip locked语法，能够立即返回。如果查询的行已经加锁，那么nowait会立即报错返回，而skip locked也会立即返回，只是返回的结果中不包含被锁定的行。
- 应用场景比如查询余票记录，如果某些记录已经被锁定，用skip locked可以跳过被锁定的记录，只返回没有锁定的记录，提高系统性能。

### 新增innodb_dedicated_server自适应参数
- 能够让InnoDB根据服务器上检测到的内存大小自动配置innodb_buffer_pool_size(buffer pool缓存大小，一般为物理内存的60%-70%。)，innodb_log_file_size（单个redo log文件大小，默认值为48M）等参数，会尽可能多的占用系统可占用资源提升性能。解决非专业人员安装数据库后默认初始化数据库参数默认值偏低的问题，前提是服务器是专用来给MySQL数据库的，如果还有其他软件或者资源或者多实例MySQL使用，不建议开启该参数，不然会影响其它程序。
```sql
show variables like '%innodb_dedicated_server%'; ‐‐默认是OFF关闭，修改为ON打开
```
### 死锁检查控制
MySQL 8.0 （MySQL 5.7.15）增加了一个新的动态变量 innodb_deadlock_detect，用于控制系统是否执行 InnoDB 死锁检查，默认是打开的。死锁检测会耗费数据库性能的，对于高并发的系统，我们可以关闭死锁检测功能，提高系统性能。但是我们要确保系统极少情况会发生死锁，同时要将锁等待超时参数调小一点，以防出现死锁等待过久的情况。
```sql
 show variables like '%innodb_deadlock_detect%'; ‐‐默认是打开的
```
### undo文件不再使用系统表空间
默认创建2个UNDO表空间，不再使用系统表空间。

<img  src="https://github.com/niqinhua/java-interview-draft/assets/27798171/da0f8120-725d-4f5c-a760-887d764d2377">

### binlog日志过期时间精确到秒
之前是天，并且参数名称发生变化. 在8.0版本之前，binlog日志过期时间设置都是设置expire_logs_days参数，而在8.0版本中，
MySQL默认使用binlog_expire_logs_seconds参数。

### 窗口函数(Window Functions)：也称分析函数
- 窗口函数与 SUM()、COUNT() 这种分组聚合函数类似，在聚合函数后面加上over()就变成窗口函数了，在括号里可以加上partition by等分组关键字指定如何分组，窗口函数即便分组也不会将多行查询结果合并为一行，而是将结果放回多行当中，即窗口函数不需要再使用 GROUP BY。
- 专用窗口函数
  - 序号函数：ROW_NUMBER()、RANK()、DENSE_RANK()
  - 分布函数：PERCENT_RANK()、CUME_DIST()
  - 前后函数：LAG()、LEAD()
  - 头尾函数：FIRST_VALUE()、LAST_VALUE()
  - 其它函数：NTH_VALUE()、NTILE()

```sql
 # 创建一张账户余额表
 CREATE TABLE `account_channel` (
 `id` int NOT NULL AUTO_INCREMENT,
 `name` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci DEFAULT NULL COMMENT '姓名',
 `channel` varchar(20) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci DEFAULT NULL COMMENT '账户渠道',
 `balance` int DEFAULT NULL COMMENT '余额',
 PRIMARY KEY (`id`)
 ) ENGINE=InnoDB

 # 插入一些示例数据
 INSERT INTO `test`.`account_channel` (`id`, `name`, `channel`, `balance`) VALUES ('1', 'zhuge', 'wx','100');
 INSERT INTO `test`.`account_channel` (`id`, `name`, `channel`, `balance`) VALUES ('2', 'zhuge', 'alipay','200');
 INSERT INTO `test`.`account_channel` (`id`, `name`, `channel`, `balance`) VALUES ('3', 'zhuge', 'yinhang','300');
 INSERT INTO `test`.`account_channel` (`id`, `name`, `channel`, `balance`) VALUES ('4', 'lilei', 'wx','200');
 INSERT INTO `test`.`account_channel` (`id`, `name`, `channel`, `balance`) VALUES ('5', 'lilei', 'alipay','100');
 INSERT INTO `test`.`account_channel` (`id`, `name`, `channel`, `balance`) VALUES ('6', 'hanmeimei', 'wx','500');

 mysql> select * from account_channel;
 +‐‐‐‐+‐‐‐‐‐‐‐‐‐‐‐+‐‐‐‐‐‐‐‐‐+‐‐‐‐‐‐‐‐‐+
 | id | name | channel | balance |
 +‐‐‐‐+‐‐‐‐‐‐‐‐‐‐‐+‐‐‐‐‐‐‐‐‐+‐‐‐‐‐‐‐‐‐+
 | 1 | zhuge | wx | 100 |
 | 2 | zhuge | alipay | 200 |
 | 3 | zhuge | yinhang | 300 |
 | 4 | lilei | wx | 200 |
 | 5 | lilei | alipay | 100 |
 | 6 | hanmeimei | wx | 500 |
 +‐‐‐‐+‐‐‐‐‐‐‐‐‐‐‐+‐‐‐‐‐‐‐‐‐+‐‐‐‐‐‐‐‐‐+
 6 rows in set (0.00 sec)

 mysql> select name,sum(balance) from account_channel group by name;
 +‐‐‐‐‐‐‐‐‐‐‐+‐‐‐‐‐‐‐‐‐‐‐‐‐‐+
 | name | sum(balance) |
 +‐‐‐‐‐‐‐‐‐‐‐+‐‐‐‐‐‐‐‐‐‐‐‐‐‐+
 | zhuge | 600 |
 | lilei | 300 |
 | hanmeimei | 500 |
 +‐‐‐‐‐‐‐‐‐‐‐+‐‐‐‐‐‐‐‐‐‐‐‐‐‐+
 3 rows in set (0.00 sec)

 # 在聚合函数后面加上over()就变成分析函数了，后面可以不用再加group by制定分组，因为在over里已经用partition关键字指明了如何分组计算，这种可以保留原有表数据的结构，不会像分组聚合函数那样每组只返回一条数据
 mysql> select name,channel,balance,sum(balance) over(partition by name) as sum_balance from account_channel;
 +‐‐‐‐‐‐‐‐‐‐‐+‐‐‐‐‐‐‐‐‐+‐‐‐‐‐‐‐‐‐+‐‐‐‐‐‐‐‐‐‐‐‐‐+
 | name | channel | balance | sum_balance |
 +‐‐‐‐‐‐‐‐‐‐‐+‐‐‐‐‐‐‐‐‐+‐‐‐‐‐‐‐‐‐+‐‐‐‐‐‐‐‐‐‐‐‐‐+
 | hanmeimei | wx | 500 | 500 |
 | lilei | wx | 200 | 300 |
 | lilei | alipay | 100 | 300 |
 | zhuge | wx | 100 | 600 |
 | zhuge | alipay | 200 | 600 |
 | zhuge | yinhang | 300 | 600 |
 +‐‐‐‐‐‐‐‐‐‐‐+‐‐‐‐‐‐‐‐‐+‐‐‐‐‐‐‐‐‐+‐‐‐‐‐‐‐‐‐‐‐‐‐+
 6 rows in set (0.00 sec)

# 每个分组的sum_balance为前面行和当前行的累加
 mysql> select name,channel,balance,sum(balance) over(partition by name order by balance) as sum_balance from account_channel;
 +‐‐‐‐‐‐‐‐‐‐‐+‐‐‐‐‐‐‐‐‐+‐‐‐‐‐‐‐‐‐+‐‐‐‐‐‐‐‐‐‐‐‐‐+
 | name | channel | balance | sum_balance |
 +‐‐‐‐‐‐‐‐‐‐‐+‐‐‐‐‐‐‐‐‐+‐‐‐‐‐‐‐‐‐+‐‐‐‐‐‐‐‐‐‐‐‐‐+
 | hanmeimei | wx | 500 | 500 |
 | lilei | alipay | 100 | 100 |
 | lilei | wx | 200 | 300 |
 | zhuge | wx | 100 | 100 |
 | zhuge | alipay | 200 | 300 |
 | zhuge | yinhang | 300 | 600 |
 +‐‐‐‐‐‐‐‐‐‐‐+‐‐‐‐‐‐‐‐‐+‐‐‐‐‐‐‐‐‐+‐‐‐‐‐‐‐‐‐‐‐‐‐+
 6 rows in set (0.00 sec)


 # over()里如果不加条件，则默认使用整个表的数据做运算
 mysql> select name,channel,balance,sum(balance) over() as sum_balance from account_channel;
 +‐‐‐‐‐‐‐‐‐‐‐+‐‐‐‐‐‐‐‐‐+‐‐‐‐‐‐‐‐‐+‐‐‐‐‐‐‐‐‐‐‐‐‐+
 | name | channel | balance | sum_balance |
 +‐‐‐‐‐‐‐‐‐‐‐+‐‐‐‐‐‐‐‐‐+‐‐‐‐‐‐‐‐‐+‐‐‐‐‐‐‐‐‐‐‐‐‐+
 | zhuge | wx | 100 | 1400 |
 | zhuge | alipay | 200 | 1400 |
 | zhuge | yinhang | 300 | 1400 |
 | lilei | wx | 200 | 1400 |
 | lilei | alipay | 100 | 1400 |
 | hanmeimei | wx | 500 | 1400 |
 +‐‐‐‐‐‐‐‐‐‐‐+‐‐‐‐‐‐‐‐‐+‐‐‐‐‐‐‐‐‐+‐‐‐‐‐‐‐‐‐‐‐‐‐+
 6 rows in set (0.00 sec)

### 按照balance字段排序，展示序号
 mysql> select name,channel,balance,row_number() over(order by balance) as row_number1 from account_channel;
 +‐‐‐‐‐‐‐‐‐‐‐+‐‐‐‐‐‐‐‐‐+‐‐‐‐‐‐‐‐‐+‐‐‐‐‐‐‐‐‐‐‐‐‐+
 | name | channel | balance | row_number1 |
 +‐‐‐‐‐‐‐‐‐‐‐+‐‐‐‐‐‐‐‐‐+‐‐‐‐‐‐‐‐‐+‐‐‐‐‐‐‐‐‐‐‐‐‐+
 | zhuge | wx | 100 | 1 |
 | lilei | alipay | 100 | 2 |
 | zhuge | alipay | 200 | 3 |
 | lilei | wx | 200 | 4 |
| zhuge | yinhang | 300 | 5 |
| hanmeimei | wx | 500 | 6 |
+‐‐‐‐‐‐‐‐‐‐‐+‐‐‐‐‐‐‐‐‐+‐‐‐‐‐‐‐‐‐+‐‐‐‐‐‐‐‐‐‐‐‐‐+
6 rows in set (0.00 sec)

# 按照balance字段排序，first_value()选出排第一的余额
mysql> select name,channel,balance,first_value(balance) over(order by balance) as first1 from account_chann

+‐‐‐‐‐‐‐‐‐‐‐+‐‐‐‐‐‐‐‐‐+‐‐‐‐‐‐‐‐‐+‐‐‐‐‐‐‐‐+
| name | channel | balance | first1 |
+‐‐‐‐‐‐‐‐‐‐‐+‐‐‐‐‐‐‐‐‐+‐‐‐‐‐‐‐‐‐+‐‐‐‐‐‐‐‐+
| zhuge | wx | 100 | 100 |
| lilei | alipay | 100 | 100 |
| zhuge | alipay | 200 | 100 |
| lilei | wx | 200 | 100 |
| zhuge | yinhang | 300 | 100 |
| hanmeimei | wx | 500 | 100 |
+‐‐‐‐‐‐‐‐‐‐‐+‐‐‐‐‐‐‐‐‐+‐‐‐‐‐‐‐‐‐+‐‐‐‐‐‐‐‐+
6 rows in set (0.01 sec)
```
### 默认字符集由latin1变为utf8mb4
在8.0版本之前，默认字符集为latin1，utf8指向的是utf8mb3，8.0版本默认字符集为utf8mb4，utf8默认指向的也是utf8mb4。\

### MyISAM系统表全部换成InnoDB表
将系统表(mysql)和数据字典表全部改为InnoDB存储引擎，默认的MySQL实例将不包含MyISAM表，除非手动创建MyISAM表。

### 元数据存储变动
MySQL 8.0删除了之前版本的元数据文件，例如表结构.frm等文件，全部集中放入mysql.ibd文件里。

<img  src="https://github.com/niqinhua/java-interview-draft/assets/27798171/070fbf49-0108-438f-9284-3e128c0b9afa">

### 自增变量持久化
在8.0之前的版本，自增主键AUTO_INCREMENT的值如果大于max(primary key)+1，在MySQL重启后，会重置AUTO_INCREMENT=max(primary key)+1，这种现象在某些情况下会导致业务主键冲突或者其他难以发现的问题。自增主键重启重置的问题很早就被发现(https://bugs.mysql.com/bug.php?id=199)，一直到8.0才被解决，8.0版本将会对AUTO_INCREMENT值进行持久化，MySQL重启后，该值将不会改变。

```sql
 # ====MySQL 5.7演示====
 mysql> create table t(id int auto_increment primary key,c1 varchar(20));
 Query OK, 0 rows affected (0.03 sec)

 mysql> insert into t(c1) values('zhuge1'),('zhuge2'),('zhuge3');
 Query OK, 3 rows affected (0.00 sec)
 Records: 3 Duplicates: 0 Warnings: 0

 mysql> select * from t;
 +‐‐‐‐+‐‐‐‐‐‐‐‐+
 | id | c1 |
 +‐‐‐‐+‐‐‐‐‐‐‐‐+
 | 1 | zhuge1 |
 | 2 | zhuge2 |
 | 3 | zhuge3 |
 +‐‐‐‐+‐‐‐‐‐‐‐‐+
 3 rows in set (0.00 sec)

 mysql> delete from t where id = 3;
 Query OK, 1 row affected (0.01 sec)

 mysql> select * from t;
 +‐‐‐‐+‐‐‐‐‐‐‐‐+
 | id | c1 |
 +‐‐‐‐+‐‐‐‐‐‐‐‐+
 | 1 | zhuge1 |
 | 2 | zhuge2 |
 +‐‐‐‐+‐‐‐‐‐‐‐‐+
 2 rows in set (0.00 sec)

 mysql> exit;
 Bye

 # 重启MySQL服务，并重新连接MySQL, 自增主键AUTO_INCREMENT=4的值如果大于max(primary key)+1，即2+1，在MySQL重启后，会重置AUTO_INCREMENT=max(primary key)+1，即2+1=3
 mysql> insert into t(c1) values('zhuge4');
 Query OK, 1 row affected (0.01 sec)

 mysql> select * from t;
 +‐‐‐‐+‐‐‐‐‐‐‐‐+
 | id | c1 |
 +‐‐‐‐+‐‐‐‐‐‐‐‐+
 | 1 | zhuge1 |
 | 2 | zhuge2 |
 | 3 | zhuge4 |
 +‐‐‐‐+‐‐‐‐‐‐‐‐+
 3 rows in set (0.00 sec)

 mysql> update t set id = 5 where c1 = 'zhuge1';
 Query OK, 1 row affected (0.01 sec)
 Rows matched: 1 Changed: 1 Warnings: 0

 mysql> select * from t;
 +‐‐‐‐+‐‐‐‐‐‐‐‐+
 | id | c1 |
 +‐‐‐‐+‐‐‐‐‐‐‐‐+
 | 2 | zhuge2 |
 | 3 | zhuge4 |
 | 5 | zhuge1 |
 +‐‐‐‐+‐‐‐‐‐‐‐‐+
 3 rows in set (0.00 sec)

 mysql> insert into t(c1) values('zhuge5');
 Query OK, 1 row affected (0.01 sec)

 mysql> select * from t;
 +‐‐‐‐+‐‐‐‐‐‐‐‐+
 | id | c1 |
 +‐‐‐‐+‐‐‐‐‐‐‐‐+
 | 2 | zhuge2 |
 | 3 | zhuge4 |
 | 4 | zhuge5 |
 | 5 | zhuge1 |
 +‐‐‐‐+‐‐‐‐‐‐‐‐+
 4 rows in set (0.00 sec)

 mysql> insert into t(c1) values('zhuge6');
 ERROR 1062 (23000): Duplicate entry '5' for key 'PRIMARY'



 # ====MySQL 8.0演示====
 mysql> create table t(id int auto_increment primary key,c1 varchar(20));
 Query OK, 0 rows affected (0.02 sec)

 mysql> insert into t(c1) values('zhuge1'),('zhuge2'),('zhuge3');
 Query OK, 3 rows affected (0.00 sec)
 Records: 3 Duplicates: 0 Warnings: 0

 mysql> select * from t;
 +‐‐‐‐+‐‐‐‐‐‐‐‐+
 | id | c1 |
 +‐‐‐‐+‐‐‐‐‐‐‐‐+
 | 1 | zhuge1 |
 | 2 | zhuge2 |
 | 3 | zhuge3 |
 +‐‐‐‐+‐‐‐‐‐‐‐‐+
 3 rows in set (0.00 sec)

 mysql> delete from t where id = 3;
 Query OK, 1 row affected (0.01 sec)

 mysql> select * from t;
 +‐‐‐‐+‐‐‐‐‐‐‐‐+
 | id | c1 |
 +‐‐‐‐+‐‐‐‐‐‐‐‐+
 | 1 | zhuge1 |
 | 2 | zhuge2 |
 +‐‐‐‐+‐‐‐‐‐‐‐‐+
 2 rows in set (0.00 sec)

 mysql> exit;
 Bye
 [root@localhost ~]# service mysqld restart
 Shutting down MySQL.... SUCCESS!
 Starting MySQL... SUCCESS!

 # 重新连接MySQL
 mysql> insert into t(c1) values('zhuge4');
 Query OK, 1 row affected (0.00 sec)

 mysql> select * from t; ‐‐生成的id为4，不是3
 +‐‐‐‐+‐‐‐‐‐‐‐‐+
 | id | c1 |
 +‐‐‐‐+‐‐‐‐‐‐‐‐+
 | 1 | zhuge1 |
 | 2 | zhuge2 |
 | 4 | zhuge4 |
 +‐‐‐‐+‐‐‐‐‐‐‐‐+
 3 rows in set (0.00 sec)

 mysql> update t set id = 5 where c1 = 'zhuge1';
 Query OK, 1 row affected (0.01 sec)
 Rows matched: 1 Changed: 1 Warnings: 0

 mysql> select * from t;
 +‐‐‐‐+‐‐‐‐‐‐‐‐+
 | id | c1 |
 +‐‐‐‐+‐‐‐‐‐‐‐‐+
 | 2 | zhuge2 |
 | 4 | zhuge4 |
 | 5 | zhuge1 |
 +‐‐‐‐+‐‐‐‐‐‐‐‐+
 3 rows in set (0.00 sec)

 mysql> insert into t(c1) values('zhuge5');
 Query OK, 1 row affected (0.00 sec)

 mysql> select * from t;
 +‐‐‐‐+‐‐‐‐‐‐‐‐+
 | id | c1 |
 +‐‐‐‐+‐‐‐‐‐‐‐‐+
 | 2 | zhuge2 |
 | 4 | zhuge4 |
 | 5 | zhuge1 |
 | 6 | zhuge5 |
 +‐‐‐‐+‐‐‐‐‐‐‐‐+
 4 rows in set (0.00 sec)
```

### DDL原子化
- InnoDB表的DDL支持事务完整性，要么成功要么回滚。
- MySQL 8.0 开始支持原子 DDL 操作，其中与表相关的原子 DDL 只支持 InnoDB 存储引擎。
- 一个原子 DDL 操作内容包括：更新数据字典，存储引擎层的操作，在 binlog 中记录 DDL 操作。
- 支持与表相关的 DDL：数据库、表空间、表、索引的 CREATE、ALTER、DROP 以及 TRUNCATE TABLE。
- 支持的其它 DDL ：存储程序、触发器、视图、UDF 的 CREATE、DROP 以及ALTER语句。
- 支持账户管理相关的 DDL：用户和角色的 CREATE、ALTER、DROP 以及适用的 RENAME等等。

```sql
# MySQL 5.7
 mysql> show tables;
 +‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐+
 | Tables_in_test |
 +‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐+
 | account |
 | actor |
 | employee |
 | film |
 | film_actor |
 | leaf_id |
 | t1 |
 | test_innodb |
 | test_myisam |
 | test_order_id |
 +‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐+
 10 rows in set (0.01 sec)

 mysql> drop table t1,t2; //删除表报错不会回滚，t1表会被删除
 ERROR 1051 (42S02): Unknown table 'test.t2'
 mysql> show tables;
 +‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐+
 | Tables_in_test |
 +‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐+
 | account |
 | actor |
 | employee |
 | film |
 | film_actor |
 | leaf_id |
 | test_innodb |
 | test_myisam |
 | test_order_id |
 +‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐+
 9 rows in set (0.00 sec)


 # MySQL 8.0
 mysql> show tables;
 +‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐+
 | Tables_in_test |
 +‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐+
 | account |
 | actor |
 | employee |
 | film |
 | film_actor |
 | leaf_id |
 | t1 |
 | test_innodb |
 | test_myisam |
 | test_order_id |
 +‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐+
 10 rows in set (0.00 sec)

 mysql> drop table t1,t2; //删除表报错会回滚，t1表依然还在
 ERROR 1051 (42S02): Unknown table 'test.t2'
 mysql> show tables;
 +‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐+
 | Tables_in_test |
 +‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐+
 | account |
 | actor |
 | employee |
 | film |
 | film_actor |
 | leaf_id |
 | t1 |
 | test_innodb |
 | test_myisam |
 | test_order_id |
 +‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐+
 10 rows in set (0.00 sec)
```

### 参数修改持久化
MySQL 8.0版本支持在线修改全局参数并持久化，通过加上PERSIST关键字，可以将修改的参数持久化到新的配置文件（mysqld-auto.cnf）中，重启MySQL时，可以从该配置文件获取到最新的配置参数。set global 设置的变量参数在mysql重启后会失效。
```sql
 set persist innodb_lock_wait_timeout=25;
 系统会在数据目录下生成一个包含json格式的mysqld‐auto.cnf 的文件，当my.cnf 和mysqld‐auto.cnf 同时存在时，后者具有更高优先级。
```
