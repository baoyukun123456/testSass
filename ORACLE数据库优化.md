## <a>一个事务本质上有四个特点ACID：</a> ##
	Atomicity     原子性
	Consistency   一致性
	Isolation     隔离性
	Durability    耐久性
## <a>悲观锁和乐观锁的区别</a> ##
### &nbsp;&nbsp;&nbsp;&nbsp;悲观锁（Pessimistic Lock）： ###
&ensp;&ensp;&ensp;&ensp;每次拿数据的时候都会担心会被别人修改（疑心重很悲观），所以每次在拿数据的时候都会上锁。确保自己使用的过程中不会被别人访问，自己使用完后再解锁。期间需要访问该数据的都会等待。
### &nbsp;&nbsp;&nbsp;&nbsp;乐观锁（Optimistic Lock）： ###
&ensp;&ensp;&ensp;&ensp;每次拿数据的时候都完全不担心会被别人修改（心态好很乐观），所以每次在拿数据的时候都不会上锁。但是在更新数据的时候去判断该期间是否被别人修改过（使用版本号等机制），期间该数据可以随便被其他人读取。两种锁各有优缺点，不能单纯的定义哪个好于哪个。乐观锁比较适合数据修改比较少，读取比较频繁的场景，即使出现了少量的冲突，这样也省去了大量的锁的开销，故而提高了系统的吞吐量。但是如果经常发生冲突（写数据比较多的情况下），上层应用不不断的retry，这样反而降低了性能，对于这种情况使用悲观锁就更合适
## <a>Left join、right join、inner join区别</a> ##
	
	left join   :左连接，返回左表中所有的记录以及右表中连接字段相等的记录。
	right join  :右连接，返回右表中所有的记录以及左表中连接字段相等的记录。
	inner join  :内连接，又叫等值连接，只返回两个表中连接字段相等的行。
	full join   :外连接，返回两个表中的行：left join + right join
	cross join  :结果是笛卡尔积，就是第一个表的行数乘以第二个表的行数。
## <a>SQL优化</a> ##
 &emsp;&emsp;数据库有个高速缓冲的概念，这个高速缓冲呢就是存放执行过的SQL语句，那数据库在执行sql语句的时候要做很多工作，例如解析sql语句， 估算索引利用率，绑定变量，读取数据块等等这些操作。假设高速缓冲里已经存储了执行过的sql语句，那就直接匹配执行了，少了步骤，自然就快了，但是经过 测试会发现高速缓冲只对简单的表起作用，多表的情况下完全没有效果啊，例如在查询单表的时候非常快，但是假设连接多个表，就会发现速度减了一倍不止。  
最重要一点，数据库的高速缓冲是全字符匹配的，什么意思呢，看下面三个select

	--No.1
	SELECT * FROM tableA;
	--No.2
	SELECT * FROM tableA; 
	--No.3 
	SELECT * FROM tableA;
&emsp;&emsp;是不是觉得除了表sql都一样?但是他看起来虽然一样但是高速缓存是不认得，全字符匹配，索引在高速缓存里会存储三条不同的语句，ORACLE执行的是三条不同的sql。
### <b>1. SQL语句尽量用大写的</b> ###
因为oracle总是先解析SQL语句，把小写的字母转换成大写的再执行。
### <b>2. 使用表的别名</b> ###
当在SQL语句中连接多个表时, 尽量使用表的别名并把别名前缀于每个列上。这样一来,就可以减少解析的时间并减少那些由列歧义引起的语法错误。
### <b>3. 选择最有效率的表名顺序(只在基于规则的优化器中有效)</b> ###
&ensp;&ensp;&ensp;&ensp;ORACLE 的解析器按照从右到左的顺序处理FROM子句中的表名，FROM子句中写在最后的表(基础表也称为驱动表,driving table)将被最先处理，在FROM子句中包含多个表的情况下,必须选择记录条数最少的表作为基础表。如果有3个以上的表连接查询, 那就需要选择交叉表(intersection table)作为基础表, 交叉表是指那个被其他表所引用的表。

例如下面的两个语句：

	--No.1  tableA 100w条记录  
	--      tableB 1w条记录 执行速度 十秒级
	SELECT COUNT(*) FROM tableA,tableB;

	--No.2  执行速度百秒级甚至更高
	SELECT COUNT(*) FROM tableB,tableA; 

	上面的结果肯定是No.2比No.1效率更高
如下写法： 

    SELSETCOUNT(1) FROM tableA a,tableB b ,tableC c WHERE a.id=b.id AND a.id=c.id;
上面的sql中tableA 就称为交叉表，根据oracle对From子句从右向左的扫描方式，应该把交叉表放在最末尾，然后才是最小表，所以上面的应该这样写

	--tableA a 交叉表 
	--tabelB b 100w
	--tableC c 1w
	SELECT COUNT(1) FROM tableB b ,tableC c ,tableA a WHERE a.id=b.id AND a.id=c.id;
### <b>4. Where子句后面的条件过滤有讲究</b> ###
&ensp;&ensp;&ensp;&ensp;ORACLE对where子句后面的条件过滤是自下向上，从右向左扫描的，所以和From子句一样一样的，把过滤条件排个序，按过滤数据的大小，自然就是**可以过滤掉最大数量记录的条件必须写在WHERE子句的末尾最下面，最右边**，依次类推，例如

<pre name="code" class="sql"><font size=4>
--No.1 不可取 性能低下
SELECT * FROM tableA a WHERE 
a.id>500
AND a.lx = '2b'
AND a.id < 'SELCET COUNT(1) FROM tableA  WHERE id=a.id '

--No.2 性能高，能过滤最多数据的条件写在最后【谨记】
SELECT * FROM tableA a WHERE 
a.id < 'SELECT COUNT(1) FROM tableA  WHERE id=a.id '
AND a.id>500
AND a.lx = '2b
</font></pre>
### <b>5. 在select的时候少用*</b> ###
多敲敲键盘，写上字段名吧，因为ORACLE的查询器会把*转换为表的全部列名，这个会浪费时间的，所以在大表中少用。
### <b>6. 使用rowid*</b> ###
常用于分页，删除查询重复记录，非常实用，给两个例子:
<pre name="code" class="sql"><font size=4>
--查找重复记录
SELECT* FROM  tableA  a WHERE
 a.ROWID> (
 SELECT MIN(ROWID) FROM tableB b WHERE 
 a.column=b.column
 ) 

--删除相同记录
DELETE FROM  tableA  a WHERE
 a.ROWID> (
 SELECT MIN(ROWID) FROM tableB b WHERE 
 a.column=b.column
 ) 

--分页 start=10 limit=10
--end 为 start + limit
SELECT * FROM 
(
  SELECT A.*,Rownum rn FROM 
    (SELECT * FROM tableA ORDER BY id) A
  WHERE rownum <= 20
) b WHERE rn> 10  ORDER BY id DESC
</font></pre>
### <b>7.减少对数据库表的查询</b> ###
ORACLE在内部执行了许多工作: 解析SQL语句, 估算索引的利用率, 绑定变量 , 读数据块等，所以少一次访问就能提高更高的效率。
### <b>8.使用DECODE函数来减少处理时间</b> ###
使用DECODE函数可以避免重复扫描相同记录或重复连接相同的表. 
使用方法：https://www.cnblogs.com/ghzjm/p/9517127.html
### <b>9.整合简单,无关联的数据库访问</b> ###
 如果你有几个简单的数据库查询语句,你可以把它们整合到一个查询中(即使它们之间没有关系) 
### <b>10.删除重复记录</b> ###
> 最高效的删除重复记录方法 ( 因为使用了ROWID)例子：

	DELETE  FROM  EMP E  WHERE  E.ROWID > (SELECT MIN(X.ROWID) 
	FROM  EMP X  WHERE  X.EMP_NO = E.EMP_NO); 
### 11.用TRUNCATE替代DELETE
&ensp;&ensp;&ensp;&ensp;当删除表中的记录时,在通常情况下, 回滚段(rollback segments ) 用来存放可以被恢复的信息. 如果你没有COMMIT事务,ORACLE会将数据恢复到删除之前的状态(准确地说是恢复到执行删除命令之前的状况) 而当运用TRUNCATE时, 回滚段不再存放任何可被恢复的信息.当命令运行后,数据不能被恢复.因此很少的资源被调用,执行时间也会很短. (译者按: TRUNCATE只在删除全表适用,TRUNCATE是DDL不是DML) 

### 12.存储过程中多用commit(谨慎使用)
这样程序的性能得到提高,需求也会因为COMMIT所释放的资源而减少  
COMMIT所释放的资源:  
&ensp;&ensp;&ensp;&ensp;a. 回滚段上用于恢复数据的信息.  
&ensp;&ensp;&ensp;&ensp;b. 被程序语句获得的锁   
&ensp;&ensp;&ensp;&ensp;c. redo log buffer 中的空间  
&ensp;&ensp;&ensp;&ensp;d. ORACLE为管理上述3种资源中的内部花费   
### 13.不要用in，not in，用exists，not exists 来代替

	--NO.1  IN的写法  
	SELECT * FROM TABLEA A WHERE 
	A.ID IN 
	( SELECT ID FORM TABLEB B WHERE B.ID>1)
	
	--NO.2 exists 写法
	SELECT * FROM TABLEA A WHERE
	EXISTS (
	SELECT 1 FROM TABLEB B WHERE A.ID=B.ID AND B.ID>1)
### 14.用Where子句替换HAVING子句
&ensp;&ensp;&ensp;&ensp;避免使用HAVING子句, **HAVING 只会在检索出所有记录之后才对结果集进行过滤**. 这个处理需要排序,总计等操作。 如果能通过WHERE子句限制记录的数目,那就能减少这方面的开销. (非oracle中)on、where、having这三个都可以加条件的子句中，on是最先执行，where次之，having最后，因为on是先把不 符合条件的记录过滤后才进行统计，它就可以减少中间运算要处理的数据，按理说应该速度是最快的，where也应该比having快点的，因为它过滤数据后 才进行sum，在两个表联接时才用on的，所以在一个表的时候，就剩下where跟having比较了。在这单表查询统计的情况下，如果要过滤的条件没有涉及到要计算字段，那它们的结果是一样的，只是where可以使用rushmore技术，而having就不能，在速度上后者要慢如果要涉及到计算的字 段，就表示在没计算之前，这个字段的值是不确定的，根据上篇写的工作流程，where的作用时间是在计算之前就完成的，而having就是在计算后才起作 用的，所以在这种情况下，两者的结果会不同。在多表联接查询时，on比where更早起作用。系统首先根据各个表之间的联接条件，把多个表合成一个临时表 后，再由where进行过滤，然后再计算，计算完后再由having进行过滤。由此可见，要想过滤条件起到正确的作用，首先要明白这个条件应该在什么时候起作用，然后再决定放在那里 。
### 15.减少对表的查询 ###
&ensp;&ensp;&ensp;&ensp;在含有子查询的SQL语句中,要特别注意减少对表的查询.例子：

	SELECT  TAB_NAME FROM TABLES
	WHERE (TAB_NAME,DB_VER) = ( SELECT TAB_NAME,DB_VER FROM  TAB_COLUMNS  WHERE  VERSION = 604) 
### 16.通过内部函数提高SQL效率 ###
&ensp;&ensp;&ensp;&ensp;复杂的SQL往往牺牲了执行效率. 能够掌握上面的运用函数解决问题的方法在实际工作中是非常有意义的
### 17.识别’低效执行’的SQL语句 ###

	SELECT  EXECUTIONS , DISK_READS, BUFFER_GETS, 
	ROUND((BUFFER_GETS-DISK_READS)/BUFFER_GETS,2) Hit_radio, 
	ROUND(DISK_READS/EXECUTIONS,2) Reads_per_run, 
	SQL_TEXT 
	FROM  V$SQLAREA 
	WHERE  EXECUTIONS>0 
	AND  BUFFER_GETS > 0 
	AND  (BUFFER_GETS-DISK_READS)/BUFFER_GETS < 0.8 
	ORDER BY  4 DESC;
### 18.用索引提高效率 ###
&ensp;&ensp;&ensp;&ensp;索引是表的一个概念部分,用来提高检索数据的效率，ORACLE使用了一个复杂的自平衡B-tree结构. 通常,通过索引查询数据比全表扫描要快. 当ORACLE找出执行查询和Update语句的最佳路径时, ORACLE优化器将使用索引. 同样在联结多个表时使用索引也可以提高效率. 另一个使用索引的好处是,它提供了主键(primary key)的唯一性验  
证.。那些LONG或LONG RAW数据类型, 你可以索引几乎所有的列. 通常, 在大型表中使用索引特别有效. 当然,你也会发现, 在扫描小表时,使用索引同样能提高效率. 虽然使用索引能得到查询效率的提高,但是我们也必须注意到它的代价. 索引需要空间来存储,也需要定期维护, 每当有记录在表中增减或索引列被修改时, 索引本身也会被修改. 这意味着每条记录的INSERT , DELETE , UPDATE将为此多付出4 , 5 次的磁盘I/O . 因为索引需要额外的存储空间和处理,那些不必要的索引反而会使查询反应时间变慢.。定期的重构索引是有必要的。

	ALTER  INDEX <INDEXNAME> REBUILD <TABLESPACENAME>
### 19.用EXISTS替换DISTINCT ###
当提交一个包含一对多表信息(比如部门表和雇员表)的查询时,避免在SELECT子句中使用DISTINCT. 一般可以考虑用EXIST替换, EXISTS 使查询更为迅速,因为RDBMS核心模块将在子查询的条件一旦满足后,立刻返回结果. 例子

	--(低效): 
	SELECT  DISTINCT  DEPT_NO,DEPT_NAME  FROM  DEPT D , EMP E 
	WHERE  D.DEPT_NO = E.DEPT_NO 
	--(高效): 
	SELECT  DEPT_NO,DEPT_NAME  FROM  DEPT D  WHERE  EXISTS ( SELECT ‘X’ 
	FROM  EMP E  WHERE E.DEPT_NO = D.DEPT_NO); 
### 20.在java代码中尽量少用连接符“＋”连接字符串！ ###

### 21.通常避免在索引列上使用NOT ###
我们要避免在索引列上使用NOT, NOT会产生在和在索引列上使用函数相同的影响. 当ORACLE”遇到”NOT,他就会停止使用索引转而执行全表扫描。
### 22.避免在索引列上使用计算 ###
WHERE子句中，如果索引列是函数的一部分．优化器将不使用索引而使用全表扫描。例如：

	--低效： 
	SELECT … FROM  DEPT  WHERE SAL * 12 > 25000; 
	--高效: 
	SELECT … FROM DEPT WHERE SAL > 25000/12;
### 23.用 >= 替代 > ###

	--高效: 
	SELECT * FROM  EMP  WHERE  DEPTNO >=4 
	--低效: 
	SELECT * FROM EMP WHERE DEPTNO >3 
&ensp;&ensp;&ensp;&ensp;两者的区别在于, 前者DBMS将直接跳到第一个DEPT等于4的记录而后者将首先定位到DEPTNO=3的记录并且向前扫描到第一个DEPT大于3的记录. 
### 24.用UNION替换OR (适用于索引列)  ###
&ensp;&ensp;&ensp;&ensp;通常情况下, 用UNION替换WHERE子句中的OR将会起到较好的效果. 对索引列使用OR将造成全表扫描. 注意, 以上规则只针对多个索引列有效. 如果有column没有被索引, 查询效率可能会因为你没有选择OR而降低. 在下面的例子中, LOC_ID 和REGION上都建有索引。

	--高效: 
	SELECT LOC_ID , LOC_DESC , REGION 
	FROM LOCATION 
	WHERE LOC_ID = 10 
	UNION 
	SELECT LOC_ID , LOC_DESC , REGION 
	FROM LOCATION 
	WHERE REGION = “MELBOURNE” 
	--低效: 
	SELECT LOC_ID , LOC_DESC , REGION 
	FROM LOCATION 
	WHERE LOC_ID = 10 OR REGION = “MELBOURNE”
　如果你坚持要用OR, 那就需要返回记录最少的索引列写在最前面（查看前面提到的 WHERE 条件查询顺序
### 25.避免在索引列上使用IS NULL和IS NOT NULL  ###
&ensp;&ensp;&ensp;&ensp;避免在索引中使用任何可以为空的列，ORACLE将无法使用该索引．对于单列索引，如果列包含空值，索引中将不存在此记录. 对于复合索引，如果每个列都为空，索引中同样不存在此记录.　如果至少有一个列不为空，则记录存在于索引中．举例: 如果唯一性索引建立在表的A列和B列上, 并且表中存在一条记录的A,B值为(123,null) , ORACLE将不接受下一条具有相同A,B值（123,null）的记录(插入). 然而如果所有的索引列都为空，ORACLE将认为整个键值为空而空不等于空. 因此你可以插入1000 条具有相同键值的记录,当然它们都是空! 因为空值不存在于索引列中,所以WHERE子句中对索引列进行空值比较将使ORACLE停用该索引。

	--低效: (索引失效) 
	SELECT … FROM  DEPARTMENT  WHERE  DEPT_CODE IS NOT NULL; 
	--高效: (索引有效) 
	SELECT … FROM  DEPARTMENT  WHERE  DEPT_CODE >=0; 
### 26.总是使用索引的第一个列 ###
&ensp;&ensp;&ensp;&ensp;如果索引是建立在多个列上, 只有在它的第一个列(leading column)被where子句引用时,优化器才会选择使用该索引. 这也是一条简单而重要的规则，当仅引用索引的第二个列时,优化器使用了全表扫描而忽略了索引。
### 27.用UNION-ALL 替换UNION ( 如果有可能的话) ###
&ensp;&ensp;&ensp;&ensp;当SQL 语句需要UNION两个查询结果集合时,这两个结果集合会以UNION-ALL的方式被合并, 然后在输出最终结果前进行排序. 如果用UNION ALL替代UNION, 这样排序就不是必要了. 效率就会因此得到提高. 需要注意的是，UNION ALL 将重复输出两个结果集合中相同记录. 因此各位还是要从业务需求分析使用UNION ALL的可行性. UNION 将对结果集合排序,这个操作会使用到SORT_AREA_SIZE这块内存. 对于这块内存的优化也是相当重要的  
&ensp;&ensp;&ensp;&ensp;下面的SQL可以用来查询排序的消耗量

	--低效： 
	SELECT  ACCT_NUM, BALANCE_AMT 
	FROM  DEBIT_TRANSACTIONS 
	WHERE TRAN_DATE = ’31-DEC-95′ 
	UNION 
	SELECT ACCT_NUM, BALANCE_AMT 
	FROM DEBIT_TRANSACTIONS 
	WHERE TRAN_DATE = ’31-DEC-95′ 
	--高效: 
	SELECT ACCT_NUM, BALANCE_AMT 
	FROM DEBIT_TRANSACTIONS 
	WHERE TRAN_DATE = ’31-DEC-95′ 
	UNION ALL 
	SELECT ACCT_NUM, BALANCE_AMT 
	FROM DEBIT_TRANSACTIONS 
	WHERE TRAN_DATE = ’31-DEC-95′
### 28.用WHERE替代ORDER BY ###
ORDER BY 子句只在两种严格的条件下使用索引.

　　(a).ORDER BY中所有的列必须包含在相同的索引中并保持在索引中的排列顺序.

　　(b).ORDER BY中所有的列必须定义为非空.

　　WHERE子句使用的索引和ORDER BY子句中所使用的索引不能并列.

　　例如:

　　表DEPT包含以下3列:DEPT_CODE PK NOT NULL------DEPT_DESC NOT NULL------DEPT_TYPE NULL

	--低效: (索引不被使用) 
	SELECT DEPT_CODE FROM  DEPT  ORDER BY  DEPT_TYPE 
	--高效: (使用索引) 
	SELECT DEPT_CODE  FROM  DEPT  WHERE  DEPT_TYPE > 0 
### 29.避免改变索引列的类型 ###
当比较不同数据类型的数据时, ORACLE自动对列进行简单的类型转换.

	--假设 EMPNO是一个数值类型的索引列.
	
	SELECT …  FROM EMP  WHERE  EMPNO = ‘123′
	
	--实际上,经过ORACLE类型转换, 语句转化为:
	
	SELECT …  FROM EMP  WHERE  EMPNO = TO_NUMBER(‘123′)
幸运的是,类型转换没有发生在索引列上,索引的用途没有被改变.

	--现在,假设EMP_TYPE是一个字符类型的索引列.
	
	SELECT …  FROM EMP  WHERE EMP_TYPE = 123
	
	--这个语句被ORACLE转换为:
	
	SELECT …  FROM EMP  WHERETO_NUMBER(EMP_TYPE)=123
**&ensp;&ensp;&ensp;&ensp;因为内部发生的类型转换, 这个索引将不会被用到!** 为了避免ORACLE对你的SQL进行隐式的类型转换, 最好把类型转换用显式表现出来. 注意当字符和数值比较时, ORACLE会优先转换数值类型到字符类型 
### 30.需要当心的WHERE子句 ###
　某些SELECT 语句中的WHERE子句不使用索引. 这里有一些例子.

　　在下面的例子里,

　　　　(1) ‘!=’ 将不使用索引. 记住, 索引只能告诉你什么存在于表中, 而不能告诉你什么不存在于表中.

　　　　(2) ‘ | |’是字符连接函数. 就象其他函数那样, 停用了索引.

　　　　(3) ‘+’是数学函数. 就象其他数学函数那样, 停用了索引.

　　　　(4)  相同的索引列不能互相比较,这将会启用全表扫描.
### 31.使用索引需要注意的两点 ###
　　a. 如果检索数据量超过30%的表中记录数.使用索引将没有显著的效率提高。  
　　b. 在特定情况下, 使用索引也许会比全表扫描慢, 但这是同一个数量级上的区别. 而通常情况下,使用索引比全表扫描要快几倍乃至几千倍! 
### 32.避免使用耗费资源的操作 ###
　　带有DISTINCT,UNION,MINUS,INTERSECT,ORDER BY的SQL语句会启动SQL引擎  
　　执行耗费资源的排序(SORT)功能. DISTINCT需要一次排序操作, 而其他的至少需要执行两次排序. 通常, 带有UNION, MINUS , INTERSECT的SQL语句都可以用其他方式重写. 如果你的数据库的SORT_AREA_SIZE调配得好, 使用UNION , MINUS, INTERSECT也是可以考虑的, 毕竟它们的可读性很强。

### 33.优化GROUP BY ###
　　提高GROUP BY 语句的效率, 可以通过将不需要的记录在GROUP BY 之前过滤掉.下面两个查询返回相同结果但第二个明显就快了许多。

	--低效: 
	SELECT JOB , AVG(SAL) 
	FROM EMP 
	GROUP by JOB 
	HAVING JOB = ‘PRESIDENT’ 
	OR JOB = ‘MANAGER’ 
	--高效: 
	SELECT JOB , AVG(SAL) 
	FROM EMP 
	WHERE JOB = ‘PRESIDENT’ 
	OR JOB = ‘MANAGER’ 
	GROUP by JOB
### 浅谈sql中的in与not in,exists与not exists的区别 ###
#### 1、in和exists ####
in是把外表和内表作hash连接，而exists是对外表作loop循环，每次loop循环再对内表进行查询，一直以来认为exists比in效率高的说法是不准确的。**如果查询的两个表大小相当，那么用in和exists差别不大；如果两个表中一个较小一个较大，则子查询表大的用exists，子查询表小的用in；**  
例如：表A(小表)，表B(大表)

	select * from A where cc in(select cc from B)　　-->效率低，用到了A表上cc列的索引；
	
	select * from A where exists(select cc from B where cc=A.cc)　　-->效率高，用到了B表上cc列的索引。
相反的：

	select * from B where cc in(select cc from A)　　-->效率高，用到了B表上cc列的索引
	
	select * from B where exists(select cc from A where cc=B.cc)　　-->效率低，用到了A表上cc列的索引。
#### 2、not in 和not exists ####
not in 逻辑上不完全等同于not exists，如果你误用了not in，小心你的程序存在致命的BUG，请看下面的例子：  

	create table #t1(c1 int,c2 int);
	
	create table #t2(c1 int,c2 int);
	
	insert into #t1 values(1,2);
	
	insert into #t1 values(1,3);
	
	insert into #t2 values(1,2);
	
	insert into #t2 values(1,null);
	
	select * from #t1 where c2 not in(select c2 from #t2);　　-->执行结果：无
	
	select * from #t1 where not exists(select 1 from #t2 where #t2.c2=#t1.c2)　　-->执行结果：1　　3

正如所看到的，not in出现了不期望的结果集，存在逻辑错误。如果看一下上述两个select 语句的执行计划，也会不同，后者使用了hash_aj，所以，请尽量不要使用not in(它会调用子查询)，而尽量使用not exists（它会调用关联子查询）。如果子查询中返回的任意一条记录含有空值，则查询将不返回任何记录。如果子查询字段有非空限制，这时可以使用not in，并且可以通过提示让它用hasg_aj或merge_aj连接。

如果查询语句使用了not in，那么对内外表都进行全表扫描，没有用到索引；而not exists的子查询依然能用到表上的索引。所以无论哪个表大，用not exists都比not in 要快。

<span style="color: #ff0000;">mysql中，索引，主键，唯一索引，联合索引的区别，对数据库的性能有什么影响。</span>  
（1）索引是一种特殊的文件（InnoDB数据表上的索引是表空间的一个组成部分），它们包含着对数据表里所有记录的引用指针。  
（2）普通索引（由关键字KEY或INDEX定义的索引）的唯一任务是加快对数据的访问速度。  
（3）普通索引允许被索引的数据列包含重复的值，如果能确定某个数据列只包含彼此各不相同的值，在为这个数据索引创建索引的时候就应该用关键字UNIQE把它定义为一个唯一所以，唯一索引可以保证数据记录的唯一性。  
（4）主键，一种特殊的唯一索引，在一张表中只能定义一个主键索引，逐渐用于唯一标识一条记录，是用关键字PRIMARY KEY来创建。  
（5）索引可以覆盖多个数据列，如像INDEX索引，这就是联合索引。  
（6）索引可以极大的提高数据的查询速度，但是会降低插入删除更新表的速度，因为在执行这些写操作时，还要操作索引文件。  