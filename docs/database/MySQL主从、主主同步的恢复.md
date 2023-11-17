## 背景
tce的数据库在某些情况下同步异常，需要重建主从同步状态
## 实施
适用于**非gtid**模式的主从
### 备份数据库
！！**将数据库vip所在节点视为当前主库**，视为最新数据，在vip所在的节点做备份
```python
 mysqldump  -uroot -p  --quick --master-data=2 --single-transaction  --add-drop-database --databases   tenxcloud_2_0 uaa cicd gateway_management > /tmp/bak20231102.sql
```
参数说明：

- --master-data:  [详见](https://dev.mysql.com/doc/refman/5.7/en/mysqldump.html#option_mysqldump_master-data)

--master-data=2   表示在dump过程中记录主库的binlog和pos点，并在dump文件中注释掉这一行；
--master-data=1   表示在dump过程中记录主库的binlog和pos点，并在dump文件中直接执行这一行；
![image.png](https://raw.githubusercontent.com/imdingtalk/img/images/pub/202311171057668.png)
![image.png](https://raw.githubusercontent.com/imdingtalk/img/images/pub/202311171057481.png)

- --single-transaction  

**简单来说就是不锁表备份**
此选项将事务隔离级别设置为REPEATABLE READ，并在转储数据之前向服务器发送START TRANSACTION SQL语句。它仅适用于事务表，因为它会在发出STARTTRANSACTION时转储数据库的一致状态，而**不会阻塞**任何应用程序。适用于操作过程中不涉及[ALTER TABLE](https://dev.mysql.com/doc/refman/5.7/en/alter-table.html), [CREATE TABLE](https://dev.mysql.com/doc/refman/5.7/en/create-table.html), [DROP TABLE](https://dev.mysql.com/doc/refman/5.7/en/drop-table.html), [RENAME TABLE](https://dev.mysql.com/doc/refman/5.7/en/rename-table.html), [TRUNCATE TABLE](https://dev.mysql.com/doc/refman/5.7/en/truncate-table.html)的情况
术语解释：

   - 事务隔离级别：指定多个事务同时执行时，彼此之间的影响程度。REPEATABLE READ是其中一种隔离级别，它确保在同一事务中多次读取相同数据时，返回的结果始终一致。
   - STARTTRANSACTION：开始一个新的数据库事务的SQL语句。
   - 转储数据：将数据库中的数据导出到文件或其他存储介质中。
-  [--quick](https://dev.mysql.com/doc/refman/5.7/en/mysqldump.html#option_mysqldump_quick) 

为了转储大型表格，应将 --quick 选项与 --single-transaction 选项结合使用。其中，--quick 选项是指在转储数据时，MySQL将尽可能快地将查询结果发送给客户端，而不是缓存所有结果；而 --single-transaction 选项是指将整个转储操作视为一个事务，确保在转储期间不会对表格进行更改。
![image.png](https://raw.githubusercontent.com/imdingtalk/img/images/pub/202311171058015.png)

- --add-drop-database

在每个CREATE DATABASE语句之前添加DROP DATABASE语句

-  --databases

该参数后的值都视为数据库，即备份多个数据库



### 从库还原数据

1. 在**从库**还原数据

将备份数据发送到从节点
`scp bak20231102.sql   sltpro-harbor-02:/opt/tenxcloud/`
`docker cp bak20231102.sql  db:/tmp/`

2. **!!!在执行还原前，务必停止slave线程和复制**

在**主节点和从节点**都执行
`stop slave`

3. 在从节点导入完整数据

`source /tmp/bak20231102.sql`

4. 导入完成后，获取pos信息

`head -n 40  bak20231102.sql | grep POS`

5. 改变POS

` CHANGE MASTER TO MASTER_LOG_FILE='mysql-bin.000013', MASTER_LOG_POS=2436639;`

6. 启动slave进程

`start slave`
### 主主同步
此时就做好了主从同步，我们原始状态是主主同步，接下来继续
主主同步，即把刚恢复好的从库，作为一个主库， 我们把刚恢复好的从库称为为**db2**,原始主库称为**db1**，以便接下来的描述
接下来我们要把db2当做主库

1. 在db2上查看master的pos信息

`show master status\G;`
![image.png](https://raw.githubusercontent.com/imdingtalk/img/images/pub/202311171058167.png)

2. db1此时视作从库，改变从库的pos信息

`CHANGE MASTER TO MASTER_LOG_FILE='mysql-bin.000005', MASTER_LOG_POS=926136933;`

3. 在db1启动slave线程

`start  slave;`

4. 查看db1的从库状态

`show slave status\G;`
![image.png](https://raw.githubusercontent.com/imdingtalk/img/images/pub/202311171058342.png)

此时主主同步恢复正常
![image.png](https://raw.githubusercontent.com/imdingtalk/img/images/pub/202311171058176.png)

## 参考
[https://dev.mysql.com/doc/refman/5.7/en/mysqldump.html#option_mysqldump_single-transaction](https://dev.mysql.com/doc/refman/5.7/en/mysqldump.html#option_mysqldump_single-transaction)

 
