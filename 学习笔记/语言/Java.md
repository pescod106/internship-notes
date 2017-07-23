## Java
1. [Java中常用缓存Cache机制的实现](http://developer.51cto.com/art/201411/456219.htm)
2. 确定某个正在运行的类，是不是正确的版本或从正确的路径下加载的。常用这个方法.`XXXCLASS.getProtectionDomain().getCodeSource().location`
`org.apache.http.impl.conn.DefaultHttpRoutePlanner.class.getProtectionDomain().getCodeSource()
Class.forName("org.apache.http.impl.conn.DefaultHttpRoutePlanner").getProtectionDomain().getCodeSource();`
`Thread.currentThread().getContextClassLoader().loadClass("org.apache.http.impl.conn.DefaultHttpRoutePlanner").getProtectionDomain().getCodeSource()
ClassLoader.getSystemClassLoader().loadClass("org.apache.http.impl.conn.DefaultHttpRoutePlanner").getProtectionDomain().getCodeSource()`
如果是个实例，则这样
`user.getClass().getProtectionDomain().getCodeSource().location`

## MySQL
1. 按日期的小时分组查询`select date_format(parsed_at,"%Y%m%d %h"),count(*) from third_response_record_snapshot where parsed_at between "2017-04-19 00:00:00" and  "2017-04-20 00:00:00" group by date_format(parsed_at,"%Y%m%d %h");`
2. `not in`与`not exists`的区别：使用`not in`查询时，会把null值忽略掉，比如 `select name from student where name not in (select name from user)`,结果中并不会包含name为空的结构.而`not exists`不同，并不会忽略，比如`select name from student where not eixsts (select null from user where user.name = student.name)`
3. `ORDER BY` 和`GROUP BY`组合使用的时候，`ORDER BY`子句中的列必须包含在聚合函数或 `GROUP BY` 子句中 
4. 在查找最大ID(auto_increment)时，使用`order by id desc limit 0,1`比`max(id)`效率高很多，比如：`select id from cell_local where created_at > "2017-05-20" order by id desc limit 0,1`比`select max(id) from cell_local where created_at > "2017-05-20"`快很多。
5. `group by`有一个原则，就是select后面的所有列中，没有使用聚合函数的列，必须出现在`group by`后面。
6. 查看正在执行的SQL语句：`show processlist`(输出SQL前面的n个字符，SQL较长时不能完整输出)、`show full processlist`(输出完整SQL)
7. 快速删除大量数据优化：(1)删除之前先保存当前索引的DDL，然后删除其索引，然后根据使用的删除条件建立一个临时索引，开始删除数据，删除完成之后重新建立之前的索引。注意，记得在删除的时候不要在记录日志的模式下(2)
8. 查看MySQL的超时时间设置`SHOW GLOBAL VARIABLES LIKE '%timeout%';`