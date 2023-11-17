## 


# 性能测试
本段文旨在提供一个性能测试方法参考，提供一个一般情况下的数据库性能测试方案

！！！建议业务根据自身的数据库使用情况，在生产环境使用数据库前做好相关测试
## 准备工作
### 测试环境准备
 通过tce平台部署mysql集群或者单机版mysql，获取mysql连接地址
### 压测工具-sysbench
```sql
yum install sysbench  -y
```
### 压测前提说明
测试时使用的脚本为lua脚本，可以使用sysbench自带脚本，也可以自己开发。对于大多数应用，使用sysbench自带的脚本就足够了。  大多数数据服务都是oltp类型的_MySQL OLTP（On-Line Transaction Processing联机事务处理过程）；_

- 尽量不要在MySQL服务器运行的机器上进行测试，一方面可能无法体现网络（哪怕是局域网）的影响，另一方面，sysbench的运行（尤其是设置的并发数较高时）会影响MySQL服务器的表现。
- 可以逐步增加客户端的并发连接数（--thread参数），观察在连接数不同情况下，MySQL服务器的表现；如分别设置为10,20,50,100等。
- 一般执行模式选择complex即可，如果需要特别测试服务器只读性能，或不使用事务时的性能，可以选择simple模式或nontrx模式。
- 如果连续进行多次测试，注意确保之前测试的数据已经被清理干净。
## 测试步骤

1. 获取集群负载均衡地址：
2. 准备数据prepare 
```shell
sysbench oltp_read_only --tables=10 --table_size=100000  --mysql-host={IP}  --mysql-port={port} --mysql-user={root} --mysql-password={密码} --mysql-db=sbtest  prepare


# 示例
sysbench oltp_read_only --tables=10 --table_size=10000  --mysql-host=172.42.135.221  --mysql-port=3306 --mysql-user=root --mysql-password=root --mysql-db=test_database  prepare

## 清理数据
sysbench oltp_read_only --tables=20 --table_size=50000  --mysql-host=172.42.206.144  --mysql-port=3306 --mysql-user=root --mysql-password=root --mysql-db=test_database   cleanup
```
参数解释：

- --tables 创建10个表
- --table_size 每个条10万条数据，
- --mysql-host 所连接 MySQL集群的IP:172.42.135.221
- --mysql-port 所连接MySQL集群的端口号
- --mysql-user MySQL集群用户名 
- --mysql-password MySQL集群密码
- --mysql-db 测试的db （这里写的test_database ，需要先登录到MySQL，通过 create database test_database 提前创建）
3. 测试执行

只读性能测试 `oltp_read_only`
```shell
sysbench oltp_read_only --tables=10 --table_size=10000  --mysql-host={IP}  --mysql-port={port} --mysql-user={root} --mysql-password={密码} --mysql-db=sbtest --time=120 --threads=100 --report-interval=10 run
# 示例
sysbench oltp_read_only --tables=10 --table_size=10000  --mysql-host=172.42.135.221  --mysql-port=3306 --mysql-user=root --mysql-password=root --mysql-db=test_database --time=120 --threads=100 --report-interval=10 run
```
参数解释：

- --time=120 测试持续时间120s
- --threads=100   100个并发数
-  --report-interval=10  每10s生成一个报告

读写性能测试 `oltp_read_write`
```shell
sysbench oltp_read_write --tables=10 --table_size=10000  --mysql-host={IP}  --mysql-port={port} --mysql-user={root} --mysql-password={密码} --mysql-db=sbtest --time=120 --threads=16 --report-interval=10 run
# 示例
sysbench oltp_read_write --tables=10 --table_size=10000  --mysql-host=172.42.135.221    --mysql-port=3306  --mysql-user=root  --mysql-password=root --mysql-db=test_database --time=120 --threads=16 --report-interval=10 run
```


数据清理
```shell
sysbench   --mysql-host={IP}  --mysql-port={port} --mysql-user={root} --mysql-password={密码} --mysql-db=sbtest cleanup

```


## 数据记录

规格1： 

- 2C4G*3  一主二从
- 10表，每表10000数据
| 类型 | 并发数 | TPS | QPS | 95th percentile(ms) |
| --- | --- | --- | --- | --- |
| 只读 | 20 | 456.47 | 7303.55 | 89.16 |
| 只读 | 50 | 418.22 | 6691.58 | 176.73 |
| 只读 | 100 | 395.74  | 6331.92 | 356.70 |
| 只读 | 300 | 342.83 | 5485.35 | 1453.01 |
| 读写 | 20 | 273.12 | 5462.42 | 101.13 |
| 读写 | 50 | 234.65 | 4693.12 | 350.33 |
| 读写 | 100 | 209.11 | 4182.29 | 802.05 |
| 读写 | 150 | 212.57 | 4251.36  | 1327.91 |

资源使用情况：
![image.png](https://raw.githubusercontent.com/imdingtalk/img/images/202311081205716.png)

## 数据说明
TPS和QPS是对系统性能的度量，值越高表示系统处理能力越强。95th percentile的响应时间表示在所有请求中，有95%的请求的响应时间低于该值，该值越小表示系统响应速度越快。根据数据记录结果，可以看出随着并发数的增加，系统的吞吐量下降，而响应时间则增加。
系统上线前，应该根据业务实际情况做对应压测，以便确认合适的资源值
mysql的request和limits值应该保持一致，以便获得较好的QoS

# 稳定性测试

测试说明
模拟在读写数据时候，主库因故宕机

1. 数据持续写入
```sql
sysbench oltp_read_write --tables=10 --table_size=10000  --mysql-host=172.45.234.40    --mysql-port=3306  --mysql-user=root  --mysql-password=root --mysql-db=test_database --time=120 --threads=100  --report-interval=10 run
```

2. 模拟主库宕机
```sql
[root@stcs-master-01 ~]# kubectl exec -it mysql-demo-0  -c mysql -n demo bash
bash-4.2# kill 1
```

3. 观察集群状态

在重连数据库后，读写无问题
![image.png](https://raw.githubusercontent.com/imdingtalk/img/images/202311081204750.png)

集群状态可以自愈
![image.png](https://raw.githubusercontent.com/imdingtalk/img/images/202311081205935.png)

集群本身有自愈能力，在遇到一些意外情况的时候可以自愈；但是意外情况会使得客户端的连接端开，客户端应该做好重连或者重试机制













