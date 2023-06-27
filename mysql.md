
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
- innodb的索引结构：表的底层存储是frm（表结构信息）、ibd（data+index）三个文件。ibd只会有一个聚簇索引，就是数据和索引在一起的B+树，数据存在B+树的叶子节点。但是普通索引其实还是非聚簇索引，叶子节点的data存的是主键值，存主键的好处是保证数据一致性和节省存储空间。

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
- 可重复读：解决了脏读、不可重读。但是会导致幻读，比如
- 可串行化：解决了脏读、不可重读、幻读。避免幻读的方法：(1)查询语句会对涉及的行加写锁。(2)如果执行的是一个范围查询，那么范围内的所有行和行记录所在的间隙区间范围都会被加锁。
# mysql锁
### mysql 锁类型
- 性能上区分：乐观锁、悲观锁
- 操作类型上区分：读锁、写锁（都属于悲观锁）。读锁也叫共享锁（S锁），针对同一份数据，支持多个读操作，不支持写操作。写锁也叫排他锁（X锁），针对同一份数据，不支持别人读或写操作。
  - myIsame在执行查询语句前，会自动给涉及的表加读锁，在执行update、delete、insert前，会自动给涉及的表加写锁。
  - innodb在执行查询语句前，因为有mvcc机制，不会加锁。在执行update、delete、insert前，会自动给涉及的行加锁。
- 粒度上区分：表锁和行锁。表锁一般用于数据迁移。
- 间隙锁：间隙锁是在可重复读隔离级别下才会生效，检索条件必须走索引，不然更新语句会导致表锁。行数加间隙锁就是临键锁
<img width="892" alt="截屏2023-05-03 下午8 40 32" src="https://user-images.githubusercontent.com/27798171/235918758-18b65cd2-70b2-4efe-9cda-a535dbefaafa.png">


```sql

============读锁===============
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

# innodb引擎bufferPool缓存机制详解
<img  src="https://user-images.githubusercontent.com/27798171/235950668-06d722b4-f2e3-4e0d-8480-3aeddec4e703.png">

为什么Mysql不能直接更新磁盘上的数据而设置这么一套复杂的机制来执行SQL了？
- 因为来一个请求就直接对磁盘文件进行随机读写，然后更新磁盘文件里的数据性能可能相当差。因为磁盘随机读写的性能是非常差的，所以直接更新磁盘文件是不能让数据库抗住很高并发的。
- Mysql这套机制看起来复杂，但它可以保证每个更新请求都是更新内存BufferPool，然后顺序写日志文件，同时还能保证各种异常情况下的数据一致性。更新内存的速度很快，然后顺序写磁盘上的日志文件的速度也很快，远高于随机读写磁盘文件。


