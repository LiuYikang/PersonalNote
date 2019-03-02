## 数据库事务
作为单个逻辑工作单元执行的一系列操作，要么完全地执行，要么完全地不执行。

### 事务的特性ACID
1. A:Atomicity，原子性，事务作为一个整体被执行，包含在其中的对数据库的操作要么全部被执行，要么都不执行。
2. C:Consistency，事务应确保数据库的状态从一个一致状态转变为另一个一致状态。一致状态的含义是数据库中的数据应满足完整性约束。
3. I:Isolation，多个事务并发执行时，一个事务的执行不应影响其他事务的执行。
4. D:Durability，已被提交的事务对数据库的修改应该永久保存在数据库中。

### 脏读、不可重复读和幻读
1. 脏读：一个事务读取到另一个事务没有提交的数据。
  * 事务 T1 更新了一行记录内容，但没有提交修改；
  * 事务 T2 读取更新后的行；
  * 然后 T1 执行回滚，取消了修改；
  * T2 读取的就是脏数据。
2. 不可重复读：在同一个事务中，两次读取同一数据，得到的内容不同。
  * T1 读取一行记录；
  * T2 修改了 T1 读取的记录；
  * T1 再次读取该行记录，发现读取的结果不同。
3. 幻读：同一事务中，用同样的操作读取两次，得到的记录数不同。
  * T1 读取一条指定的 where 子句返回的结果集；
  * T2 新插入一行记录，该记录正好满足 T1 查询要求；
  * T1 再次对表进行检索，看到了 T2 插入的数据。

## kv数据库和sql数据库
非关系型数据库的优势：
1. 性能NOSQL是基于键值对的，可以想象成表中的主键和值的对应关系，而且不需要经过SQL层的解析，所以性能非常高。
2. 可扩展性同样也是因为基于键值对，数据之间没有耦合性，所以非常容易水平扩展。

关系型数据库的优势：
1. 复杂查询可以用SQL语句方便的在一个表以及多个表之间做非常复杂的数据查询。
2. 事务支持使得对于安全性能很高的数据访问要求得以实现。