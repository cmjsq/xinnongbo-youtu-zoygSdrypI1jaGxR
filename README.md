
## 环境说明


1. Docker
2. Windows 11
3. PostgreSql 16


## 方案步骤


#### 0\. 宿主机准备：


1. 找个地方创建一个文件夹用来挂载容器中数据库Data文件夹，这里我用的是：
`C:\Users\Administrator\docker\Postgresql\replication`


#### 1\. 主数据库准备：


1. 执行docker run 命令，创建主数据库容器：pgsmaster



```
docker run --name pgsmaster -p 5400:5432 -e POSTGRES_PASSWORD=123456 -v C:\Users\Administrator\docker\Postgresql\replication\pgsmaster:/var/lib/postgresql/data -d postgres

```

2. 添加复制角色用户：



```
# 1.进入容器 
docker exec -it pgsmaster bash 
# 2.连接PostgreSQL 
psql -U postgres 
# 3.创建用户 
// replicator: 复制账号; 123456: 认证密码 
create role replicator login replication encrypted password '123456'; 
# 4.验证用户 
\du
# 出现如下列表则用户创建成功
List of roles 
Role name  | Attributes 
-----------+-------------------------------------------
postgres   | Superuser, Create role, Create DB, Replication, Bypass RLS 
replicator | Replication

```

3. 关闭容器：`Docker stop pgsmaster`
4. 宿主机找到挂载文件夹：`C:\Users\Administrator\docker\Postgresql\replication\pgsmaster` 下的 `postgresql.conf` 文件，修改配置：



```
1. listen_addresses ='*'
2. wal_log_hints = on
3. wal_level = replica

```
5. 找到 `pg_hba.conf` 文件，最后一行添加配置：



```
	host    replication     replicator  172.17.0.0/16       md5

```

7. 启动容器：`Docker start pgsmaster`


到这里主数据库就准备好了\~


#### 2\. 从数据库准备：


1. 同样执行docker run 命令，创建一个从数据库容器：



```
docker run --name pgsslave -p 5401:5432 -e POSTGRES_PASSWORD=123456 -v C:\Users\Administrator\docker\Postgresql\replication\pgsslave:/var/lib/postgresql/data -d postgres

```

2. 到宿主机挂在文件夹`C:\Users\Administrator\docker\Postgresql\replication\pgsslave`下创建一个`tempdata`文件夹
3. 进入容器，进行主库复制：



```
	# 1. 进入容器
	docker exec -it pgsslave bash
	# 2. 切换postgres用户
	su postgres
	# 3. 执行复制, 注意IP, 需要输入密码即主数据库创建复制用户replicator的密码‘123456’
	pg_basebackup -h 172.17.0.6 -U replicator -D /var/lib/postgresql/data/tempdata -X stream -P

```

4. 关闭从数据库容器：
`docker stop pgsslave`
5. 进入宿主机挂在文件夹下的`tempdata`文件夹下，将所文件夹下的所有文件覆盖到上级文件夹，并进入上级文件夹，我这里是：`C:\Users\Administrator\docker\Postgresql\replication\pgsslave`；
6. 修改 `postgresql.conf` 文件参数：



```
	1. primary_conninfo = 'host=172.17.0.6 port=5432 user=replicator password=123456' # 配置连接主数据库，注意IP和端口号
	2. hot_standby = on # 说明这台机器不仅仅是用于数据归档，也用于数据查询
	3. max_standby_streaming_delay = 30s # 数据流备份的最大延迟时间 
	4. wal_receiver_status_interval = 10s # 多久向主报告一次从的状态 
	5. hot_standby_feedback = on # 如果有错误的数据复制，是否向主进行反馈

```

7. 添加 `standby.signal` 文件：



```
	standby_mode = 'on'

```

8. 启动从数据库：
`docker start pgsslave`


到这里从数据库也配置好了\~


### 3\. 验证：


1. 进入主数据库容器：
`docker exec -it pgsmaster bash`
2. 执行命令：



```
	psql -U postgres -x -c "select * from pg_stat_replication;"

```

3. 出现如下信息则表示配置成功\~



```
	-[ RECORD 1 ]----+------------------------------ 
	pid              | 65 
	usesysid         | 16388 
	usename          | replicator 
	application_name | walreceiver 
	client_addr      | 172.17.0.7 
	client_hostname  | 
	client_port      | 56664 
	backend_start    | 2024-11-28 05:44:17.805887+00 
	backend_xmin     | 743 
	state            | streaming 
	sent_lsn         | 0/5000148 
	write_lsn        | 0/5000148 
	flush_lsn        | 0/5000148 
	replay_lsn       | 0/5000148 
	write_lag        | 
	flush_lag        | 
	replay_lag       | 
	sync_priority    | 0 
	sync_state       | async 
	reply_time       | 2024-11-28 05:45:57.855411+00

```

到这里，配置完毕\~


 本博客参考[veee加速器](https://youhaochi.com)。转载请注明出处！
