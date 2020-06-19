# postgresql + pgpool 构建容灾高可用集群(数据同步流复制/主备自动切换)

整个流程分为以下几部分:
1. postgresql-12 安装
2. postgresql-12 流复制配置以及验证
3. pgpoll-ii-4.1 安装
4. pgpool-ii-4.1 主备机器自动切换配置
5. pgpoll-ii-4.1 配置说明
6. 方案宕机效果测试

## 高可用(容灾)效果:
先说说这套方案要达到的效果/解决的问题,目的不一致的/没有想了解的可以不用往下看了.
解决三种宕机
1. 某一个 postgresql 数据库挂掉 (多台数据库启动后 *其中一台作为主机,其余作为备机* 构成一个`数据库集群`);
    - 如果是主机primary,集群检测到挂掉会通过配置的策略重新选一个**备机standby**切换为**主机primary**, 整个集群仍旧保证可用, 当原主机恢复服务后, 重新作为一个新备机standby,**同步完数据后加入集群**
    - 如果是备机standby,对整个集群无可见影响, 当备机恢复服务后,从主库同步完数据后,恢复正常状态加入集群;
2. 某一台机器上的pgpool-ii 程序挂掉;
    - 监测每个pgpool-ii进程的状态, 监测到挂掉之后,及时"切换"虚拟ip所在的主机以保证可用性(有些人叫IP漂移);
    - 整个集群始终对外提供一个唯一的,可用的虚拟IP 来提供访问;
    - 监测每个主机postgresql数据库的状态, 以即使切换数据库的主备角色;
3. 某一台主机直接宕机;
    - 当pgpool-ii监测主机挂掉之后, 需要进行数据库角色的切换和ip的切换两个操作(如果需要)

## 方案结构:
> 方案模型基于**两台装有postgresql数据库的主机**通过每台机器上的pgpool-ii程序来**维护一个高可用体系**, 从而保证能`始终提供一个可用的IP地址`,用于`外界数据操作或者访问`.

|发行版|ip|hostname|补充说明
|:-:|:-:|:-:|-|
|Cent OS7|10.242.111.204|master|安装`postgresql 12.1` + `pgpool-ii 4.1`并进行配置
|Cent OS7|10.242.111.207|slave|安装`postgresql 12.1` + `pgpool-ii 4.1`并进行配置
|/|10.242.111.203|vip|virtual ip, 通过一个虚拟的IP统一对外提供访问


* 2(n)台主机均安装有`postgresql 12` 版本的数据库和`pgpool-ii 4.1` 版本的中间件;
* 2(n)个数据库之间可以做到数据同步以(通过流复制来实现, 但同一时刻**主机primary**只有一台,其余作为**备机standby**)及`身份切换`;
* pgpool-ii 是一个介于postgresql 服务器和postgresql数据库之间的中间件, 提供了`链接池(Connection Pooling)`,`看门狗(WatchDog)`,`复制`,`负载均衡`,`缓存`等功能(具体的可以查看官方文档);
* 通过pgpool-ii 维护的虚拟ip, 向外界提供一个始终可用的访问地址, 屏蔽掉具体的`主机数据库`地址概念;
* 通过pgpool-ii 程序来自动处理宕机后相关方案(后面有讲)
* 数据库down之后需要通过`pcp_attach_node`将节点加入集群

> 流复制数据同步: 通过postgresql数据库配置来实现
> 虚拟ip自动切换: 通过pgpool-ii 配置实现
> 数据库主备角色切换: 通过pgpool-ii 监测机 + 执行 postgresql 中的`promote`命令来实现


### postgreslq-12安装(2台机器均安装)
> yum 在线安装  
```bash
#设置rpm源
curl -O  https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm
rpm -ivh pgdg-redhat-repo-latest.noarch.rpm

#安装(这里是版本为12的postgresql)
yum -y install postgresql12 postgresql12-server  -O

#如果后续发现连接不上,可以关闭防火墙
systemctl stop firewalld
systemctl disable firewalld

```
> `rpm`离线安装  
```bash
# ftp下载rpm离线包
curl -O https://yum.postgresql.org/12/redhat/rhel-7-x86_64/postgresql12-12.3-1PGDG.rhel7.x86_64.rpm
curl -O https://yum.postgresql.org/12/redhat/rhel-7-x86_64/postgresql12-contrib-12.3-1PGDG.rhel7.x86_64.rpm
curl -O https://yum.postgresql.org/12/redhat/rhel-7-x86_64/postgresql12-libs-12.3-1PGDG.rhel7.x86_64.rpm
curl -O https://yum.postgresql.org/12/redhat/rhel-7-x86_64/postgresql12-server-12.3-1PGDG.rhel7.x86_64.rpm
#contrib 是安装扩展的 没有这个包就没有 ossp-uuid的插件
#server 是数据库的安装文件
#libs 用来客户端进行连接. 
#注意 如果是centos8 的话,修改为 rhel-8 进行下载就可以了.
```
```bash
# 上传文件到服务器之后, 执行安装命令
rpm -ivh postgresql*.rpm
```
执行完安装之后(查看状态**可跳过, 直接进行数据库初始化**):
- 会帮我们创建一个`postgresql-12`服务, 此时未进行数据库初始化, 还无法访问.
- 会帮我们创建一个`postgres`/`postgres` 的用户，密码相同.

此时使用`systemctl status postgreslq-12` 查看服务状态:
```bash
● postgresql-12.service - PostgreSQL 12 database server
   Loaded: loaded (/usr/lib/systemd/system/postgresql-12.service; disabled; vendor preset: disabled)
   Active: inactive (dead)
     Docs: https://www.postgresql.org/docs/12/static/
```
我们可以找到默认配置文件地址: `/usr/lib/systemd/system/postgresql-12.service`  

如果`cat`命令查看配置文件, 我们可以得到一些基础信息:  
`数据库数据目录`: Environment=`PGDATA=/var/lib/pgsql/12/data/`  
`postgresql安装目录`: `PGHOME=/usr/pgsql-12/`

#### 数据库初始化
```bash
# 切换到postgres用户
su - postgres
cd /usr/pgsql-12/bin
./initdb -D /var/lib/pgsql/12/data
```
> 退回root用户, 重启`postgrersql-12`服务, 并设置开机自启  
```bash
systemctl enable  postgresql-12 && systemctl restart postgresql-12
```
> 本机测试访问  
```bash
su - postgres
-bash-4.2$  psql
postgres=# (这里就可以开始写查询语句了)
```
> 配置远程访问  

修改数据目录下配置文件: `pg_hba.conf`
```bash
vim /var/lib/pgsql/12/data/pg_hba.conf
# 在文件中添加:
host     all              all             0.0.0.0/0               md5
```
修改数据目录下配置文件: `postgresql.conf`
```bash
vim /var/lib/pgsql/12/data/postgresql.conf
# 在文件中修改(此配置仅用于远程访问, 流复制后续还有额外配置):
listen_addresses = '*'
port = 5432
max_connections = 100 
```
修改完成重启服务, 使用远程命令或者远程客户端测试访问即可
```sql
-- 使用远程命令访问
psql -h 10.242.111.204 -p 5432 -U postgres
-- 登陆成功可以执行操作
-- 如修改密码 (引号中为新密码)
postgres=# alter role postgres with password 'postgres';
```


### postgresql-12 流复制(replication)/数据同步配置
> 流复制原理简述

1. 流复制大约是从pg9版本之后使用, 流复制其原理为：备库不断的从主库同步相应的数据，并在备库apply每个WAL record，这里的流复制每次传输单位是WAL日志的record。(关于预写式日志**WAL**,是一种事务日志的实现)

    ![模型](https://img2020.cnblogs.com/blog/1039023/202006/1039023-20200619110706124-934472502.png)
2. 图中可以看到流复制中日志提交的大致流程为:
   1. 事务commit后，日志在主库写入wal日志，还需要根据配置的日志同步级别，等待从库反馈的接收结果。
   2. 主库通过日志传输进程将日志块传给从库，从库接收进程收到日志开始回放，最终保证主从数据一致性。
3. 流复制同步级别
   通过在`postgresql.conf`配置`synchronous_commit`参数来设置同步级别
    ```
    synchronous_commit = off            # synchronization level;  
                                        # off, local, remote_write, or on 

    ```
   - **remote_apply**：事务commit或rollback时，等待其redo在primary、以及同步standby(s)已持久化，并且其redo在同步standby*(s)已apply。
   - **on**：事务commit或rollback时，等待其redo在primary、以及同步standby(s)已持久化。
   - **remote_write**：事务commit或rollback时，等待其redo在primary已持久化; 其redo在同步standby(s)已调用write接口(写到 OS, 但是还没有调用持久化接口如fsync)。
   - **local**：事务commit或rollback时，等待其redo在primary已持久化;
   - **off**：事务commit或rollback时，等待其redo在primary已写入wal buffer，不需要等待其持久化;

> 配置时需要注意的点:
1. postgresql-12版本不再支持通过recovery.conf的方式进行主备切换，如果数据目录中存在recovery.conf，则数据库无法启动;
2. 新增 recovery.signal 标识文件，表示数据库处于 recovery 模式;
3. 新增加 **standby.signal** 标识文件，表示数据库处于 **standby** 模式(这个需要重点关注一下);
4. 以前版本中 standby_mode 参数不再支持;
5. recovery.conf文件取消, 合并到了`postgresql.conf`文件中;
6. 配置中`war_level`存储级别, postgresql-9.6以后有改变:
    |等级|说明|
    |:-:|-|
    |minimal|不能通过基础备份和wal日志恢复数据库
    |replica|9.6新增,将之前版本的 archive 和 hot_standby合并, 该级别支持wal归档和复制
    |logical|在replica级别的基础上添加了支持逻辑解码所需的信息|

### 流复制配置过程
#### 主库(10.242.111.204)
1. 创建一个账户`repuser` 用户, 提供给备库远程访问, 用来获取流:  
    ```bash
    su - postgres 
    -bash-4.2$ psql
    postgres=# create role repuser login replication encrypted password 'repuser';
    CREATE ROLE
    postgres=# \q
    ```
2. 修改数据目录下配置文件: `pg_hba.conf`
    ```bash
    vim /var/lib/pgsql/12/data/pg_hba.conf
    # 在文件中添加:
    host  replication     repuser        0.0.0.0/0             md5
    ```
3. 修改数据目录下配置文件: `postgresql.conf`
    ```bash
    vim /var/lib/pgsql/12/data/postgresql.conf
    # 在文件中修改(此配置仅用于远程访问, 流复制后续还有额外配置):
    listen_addresses = '*'
    port = 5432
    max_connections = 100       # 最大连接数，据说从机需要大于或等于该值

    # 控制是否等待wal日志buffer写入磁盘再返回用户事物状态信息。同步流复制模式需要打开。
    synchronous_commit = on
    # *=all，意思是所有slave都被允许以同步方式连接到master，但同一时间只能有一台slave是同步模式。
    # 另外可以指定slave，将值设置为slave的application_name即可。
    synchronous_standby_names = '*'
    wal_level = replica
    max_wal_senders = 2   		#最多有2个流复制连接
    wal_keep_segments = 16  	
    wal_sender_timeout = 60s	#流复制超时时间
    ```
修改完成**重启postgresql-12**服务`systemctl restart postgresql-12`
#### 从库(10.242.111.204)
1. 关闭备库postgresql服务  
   ```bash
   systemctl stop postgresql-12
   ```
2. 如果备库机器上没有 PGDATA(/var/lib/pgsql/12/data)目录(恢复出故障数据目录消失同样操作) 
    ```bash
   mkdir /var/lib/pgsql/12/data
   chmod 0700 data
   chown postgres.postgres data
   ```
3. 把主库整个备份到从库  
    其实后续的pgpool的主库挂了, 从库升级主库之后, 主库恢复为从库的过程就是: 备份data目录,然后重复这里的第2,3步骤
    ```bash
    # 切换到postgres用户，否则备份的文件属主为root, 导致启动出错
    su – postgres
    pg_basebackup -h 10.242.111.204 -p 5432 -U repuser -Fp -Xs -Pv -R -D /var/lib/pgsql/12/data
    # -h 的ip是当前的主库, -U 就是前面个船舰的用来复制的那个用户
    ```
    额外需要注意的是:
    - 如果主库的pg_hba.conf中配置的策略为trust, 这里不需要口令, 如果为md5模式,需要输你创建用户时的那个密码;
    - 这里 -R 参数一定要加, 拷贝完在$PGDATA目录下生成standby.signal标志文件(用于表示此库为备库);
    - 使用命令同步完之后, 在data目录下会自动生成`postgresql.auto.conf`文件中, 优先级是大于postgresql.conf的;
    - **这里面的参数请严格参照官网释义**;
4. 启动备库postgresql-12 服务
    ```bash
   systemctl restart postgresql-12
   ```
### 流复制验证
使用命令或者远程客户端工具登入postgresql主(primary)数据库
```bash
# 切换垂直显示
postgres=# \x
Expanded display is on.
# 查询备机连接
postgres=# select * from pg_stat_replication;
    -[ RECORD 1 ]----+------------------------------
    pid              | 27132
    usesysid         | 16384
    usename          | repuser
    application_name | walreceiver
    client_addr      | 10.242.111.207
    client_hostname  |
    client_port      | 34726
    backend_start    | 2020-06-19 16:23:01.309172+08
    backend_xmin     |
    state            | streaming
    sent_lsn         | 0/11000148
    write_lsn        | 0/11000148
    flush_lsn        | 0/11000148
    replay_lsn       | 0/11000148
    write_lag        |
    flush_lag        |
    replay_lag       |
    sync_priority    | 1
    sync_state       | sync
    reply_time       | 2020-06-19 18:39:43.346062+08
# 可以发现207的那台备机已经连接上, 并且使用的是 sync 同步模式(async 为异步)
# 此时可以在主库上创建数据库/表/添加数据, 然后去备库上查询 以显示效果, 如:
postgres=# create database test_204;
postgres=# \test_204
postgres=# create table test_table(name text);
postgres=# insert into test_table(name) values ('china');

# 补充基础查询命令
# \list 列出可用数据库
# \db_name 选择数据库
# \dt 查询关系表
```

至此, 流复制(数据同步)策略完成.

#### postgresql 主从切换(primary->standby)
1. 主数据库是读写的，备数据库是只读的。
2. 使用`/usr/pgsql-12/bin/pg_controldata /var/lib/pgsql/12/data/` 可以查看数据库运行状态
   - Database cluster state: in production  (主库为运行中)
   - Database cluster state: in archive recovery    (从库为正在归档回复)
   -  
3. 当主库primary 出现故障, 我们需要将从库standby提升为主库primary;
   * pg12版本以前的切换方式有两种:
      - pg_ctl 方式: 在备库主机执行 pg_ctl promote shell 脚本
      - 触发器文件方式: 备库配置 recovery.conf 文件的 trigger_file 参数，之后在备库主机上创建触发器文件
   * pg12开始新增了一个`pg_promote()`函数, 支持知用promote指令进行切换
   * `pg_promote(wait boolean DEFAULT true, wait_seconds integer DEFAULT 60)`
     * wait: 表示是否等待备库的 promotion 完成或者 wait_seconds 秒之后返回成功，默认值为 true
     * wait_seconds: 等待时间，单位秒，默认 6
4. 切换举例
    ```bash
    # 关闭主库primary
    systemctl stop postgresql-12
    # 从库上执行切换
    # 第一种
    /usr/pgsql-12/bin/pg_ctl promote -D /var/lib/pgsql/12/data
    # 第三种
    su postgres
    psql
    postgres=# select pg_promote(true,60);
    # 此时从库会提升为主库接管集群中的读写操作
    ```

pgpool-ii 中默认的 failover_command 切换数据库主备角色脚本, 是通过`pg_ctl promote` 来实现的


参考资料:
1. [流复制](https://olei.me/829/#title-8)
2. [容灾测试](https://www.jianshu.com/p/ef183d0a9213)
3. [pg_basebackup介绍](https://postgres.fun/20130702100240.html?spm=a2c4e.10696291.0.0.4f9b19a4Qb015w)





<!-- 搭建PostgreSQL11集群 https://www.jianshu.com/p/9e4065016b06#pgpool%E9%85%8D%E7%BD%AE -->
<!-- markdown 优化https://blog.csdn.net/xiaojin21cen/article/details/78752561 -->
