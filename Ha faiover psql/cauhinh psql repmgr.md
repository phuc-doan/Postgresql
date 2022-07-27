-https://www.2ndquadrant.com/en/blog/how-to-automate-postgresql-12-replication-and-failover-with-repmgr-part-1/


# Let's get started!


# 1\. Giới thiệu

PostgreSQL streaming replication sử dụng service Repmgr để cấu hình replication, failover phục vụ nhu cầu HA cho Database Postgre Cluster
Repmgr là một tool opensource dùng dể quản lý replication và failovẻ trong cluster PostgreSQL Server. Nó cung cấp PostgreSQL hot-standby để setup standby server, monitor replication, và thực hiện chuyển đổi dự phòng tự động hoặc thủ công

# 2\. Replication trong PostgreSQL hoạt động thế nào ?

- Trong 1 cụm setup replication PostgreSQL gồm 2 kiểu server (Master & Slave). Các bản ghi (record) database được Slave sao chép từ Master. Các record này có thể đọc(read) từ Slave Server, nhưng chỉ có thể được ghi (write) trên Master Server. Các server được sync với nhau. Khi một Master Server fail, Slave Server sẽ đứng lên giữ vai trò Master trong cụm.
    
- Replication Flow
    

# 3\. Cấu hình PostgreSQL master-standby sử dụng repmgr
![image](https://user-images.githubusercontent.com/83824403/181224915-047ab1c9-af32-4a2d-8a89-5fc116995340.png)


## 3.1. Mô hình

![image](https://user-images.githubusercontent.com/83824403/181224955-e5052ba5-8176-4108-b9b7-cb9f10fcbbf7.png)


PostgreSQL:
PG-Master: 10.3.54.116
PG-slave: 10.3.54.102

Haproxy:
VIP: 10.3.54.139
Haproxy-01: 10.3.53.253
Haproxy-02: 10.3.55.106

## 3.1. Cài đặt PostgreSQL version 11 & Repmgr

```
# apt-get install apt-transport-https
# echo "deb https://dl.2ndquadrant.com/default/release/apt stretch-$2ndquadrant main"
# wget --quiet -O - https://dl.2ndquadrant.com/gpg-key.asc | apt-key add -
# apt-get install postgresql-11 postgresql-11-repmgr
# systemctl stop postgresql
# echo "postgres ALL = NOPASSWD: /usr/bin/pg_ctlcluster" > /etc/sudoers.d/postgres
```

## 3.2. Cấu hình PostgreSQL

## a, Master Node

- Thêm file hosts

```
10.3.54.102 pg-slave
10.3.54.116 pg-master
```

- Thêm vào file config `/etc/postgresql/11/main/postgresql.conf`

```
listen_addresses = '10.3.54.116'
shared_preload_libraries = 'repmgr'
max_wal_senders = 15
max_replication_slots = 15
wal_level = 'replica'
hot_standby = on
archive_mode = on
archive_command = 'cp'
```

- Thêm vào file `/etc/postgresql/11/main/pg_hba.conf`

```
host repmgr      repmgr 10.3.54.116/22   trust
host repmgr      repmgr 10.3.54.102/22 trust
host replication repmgr 10.3.54.116/22   trust
host replication repmgr 10.3.54.102/22 trust
```

- Tạo file config repmgr `/etc/repmgr.conf` và thêm :

```
node_id=1
node_name=pg-master
conninfo='host=pg-master user=repmgr dbname=repmgr connect_timeout=2'
data_directory='/var/lib/postgresql/11/main'
failover=automatic
promote_command='repmgr standby promote -f /etc/repmgr.conf --log-to-file'
follow_command='repmgr standby follow -f /etc/repmgr.conf --log-to-file'
log_file='/var/log/postgresql/repmgr.log'
log_level=NOTICE
reconnect_attempts=4
reconnect_interval=5
```

- Sửa file `/etc/default/repmgrd`

```
REPMGRD_ENABLED=yes
REPMGRD_CONF="/etc/repmgr.conf"
```
- Tạo symbol link
```
ln -s /usr/lib/postgresql/11/bin/pg_ctl /usr/bin/pg_ctl
```
- Restart Service

```
service postgresql restart
service repmgrd restart
```

## b, Slave Node

- Thêm file hosts

```
10.3.54.102 pg-slave
10.3.54.116 pg-master
```

- Thêm vào file config `/etc/postgresql/11/main/postgresql.conf`

```
listen_addresses = '10.3.54.102'
shared_preload_libraries = 'repmgr'
max_wal_senders = 15
max_replication_slots = 15
wal_level = 'replica'
hot_standby = on
archive_mode = on
archive_command = 'cp'
```

- Thêm vào file `/etc/postgresql/11/main/pg_hba.conf`

```
host repmgr      repmgr 10.3.54.116/22   trust
host repmgr      repmgr 10.3.54.102/22 trust
host replication repmgr 10.3.54.116/22   trust
host replication repmgr 10.3.54.102/22 trust
```

- Tạo file config repmgr `/etc/repmgr.conf` và thêm :

```
node_id=2
node_name=pg-slave
conninfo='host=pg-slave user=repmgr dbname=repmgr connect_timeout=2'
data_directory='/var/lib/postgresql/11/main'
failover=automatic
promote_command='repmgr standby promote -f /etc/repmgr.conf --log-to-file'
follow_command='repmgr standby follow -f /etc/repmgr.conf --log-to-file'
log_file='/var/log/postgresql/repmgr.log'
log_level=NOTICE
reconnect_attempts=4
reconnect_interval=5
```

- Sửa file `/etc/default/repmgrd`

```
REPMGRD_ENABLED=yes
REPMGRD_CONF="/etc/repmgr.conf"
```
- Tạo symbol link
```
ln -s /usr/lib/postgresql/11/bin/pg_ctl /usr/bin/pg_ctl
```
- Restart Service

```
service postgresql restart
service repmgrd restart
```

## 3.3. Cấu hình user Postgres login sử dụng ssh key giữa 2 Server

```
# passwd postgre
# ssh-keygen
```

## 3.4. Cấu hình cluster

## a, Master Node

- Tạo user và db repmgr

```
# sudo -i -u postgres
# /usr/lib/postgresql/11/bin/pg_ctl /usr/bin/pg_ctl
# createuser --replication --createdb --createrole --superuser repmgr
# psql -c 'ALTER USER repmgr SET search_path TO repmgr_test, "$user", public;'
# createdb repmgr --owner=repmgr
```

- Register master node

```
# sudo -i -u postgres
# repmgr primary register
# repmgr cluster show
```

## a, Slave Node

- Register standby node

```
# sudo systemctl stop postgresql
# sudo -i -u postgres
# rm -rf /var/lib/postgresql/11/main
# repmgr -h pg-master -U repmgr -d repmgr standby clone
# pg_ctl -D /etc/postgresql/11/main start
# repmgr standby register
```

## 3.5. Test cluster
3.5.1. Show cluster
```
postgres@pg-master:~$ repmgr cluster show
 ID | Name      | Role    | Status    | Upstream  | Location | Priority | Timeline | Connection string
----+-----------+---------+-----------+-----------+----------+----------+----------+------------------------------------------------------------
 1  | pg-master | primary | * running |           | default  | 100      | 1        | host=pg-master user=repmgr dbname=repmgr connect_timeout=2
 2  | pg-slave  | standby |   running | pg-master | default  | 100      | 1        | host=pg-slave user=repmgr dbname=repmgr connect_timeout=2
```
### 3.5.2. Test node master down
```
root@pg-master:~# service postgresql stop 
```
- Check cluster
```
postgres@pg-slave:~$ repmgr cluster show
 ID | Name      | Role    | Status    | Upstream | Location | Priority | Timeline | Connection string                                         
----+-----------+---------+-----------+----------+----------+----------+----------+------------------------------------------------------------
 1  | pg-master | primary | - failed  | ?        | default  | 100      |          | host=pg-master user=repmgr dbname=repmgr connect_timeout=2
 2  | pg-slave  | primary | * running |          | default  | 100      | 2        | host=pg-slave user=repmgr dbname=repmgr connect_timeout=2 

WARNING: following issues were detected
  - unable to connect to node "pg-master" (ID: 1)

HINT: execute with --verbose option to see connection error messages
```
```
postgres@pg-slave:~$ repmgr cluster event
 Node ID | Name      | Event                     | OK | Timestamp           | Details                                                                                      
---------+-----------+---------------------------+----+---------------------+-----------------------------------------------------------------------------------------------
 2       | pg-slave  | repmgrd_standby_reconnect | t  | 2021-07-20 14:29:42 | node restored as standby after 52 seconds, monitoring connection to upstream node -1         
 2       | pg-slave  | standby_clone             | t  | 2021-07-20 14:29:33 | cloned from host "pg-master", port 5432; backup method: pg_basebackup; --force: N            
 1       | pg-master | repmgrd_reload            | t  | 2021-07-20 14:28:39 | monitoring cluster primary "pg-master" (ID: 1)                                               
 1       | pg-master | repmgrd_failover_promote  | t  | 2021-07-20 14:28:39 | node "pg-master" (ID: 1) promoted to primary; old primary "pg-slave" (ID: 2) marked as failed
 1       | pg-master | standby_promote           | t  | 2021-07-20 14:28:39 | server "pg-master" (ID: 1) was successfully promoted to primary                              
 2       | pg-slave  | child_node_new_connect    | t  | 2021-07-20 14:27:51 | new standby "pg-master" (ID: 1) has connected                                                
 1       | pg-master | standby_register          | t  | 2021-07-20 14:27:48 | standby registration succeeded; upstream node ID is 2                                        
 1       | pg-master | standby_unregister        | t  | 2021-07-20 14:27:15 |                                                                                              
 1       | pg-master | repmgrd_standby_reconnect | t  | 2021-07-20 14:23:37 | node restored as standby after 24 seconds, monitoring connection to upstream node -1         
 1       | pg-master | standby_clone             | t  | 2021-07-20 14:23:19 | cloned from host "pg-slave", port 5432; backup method: pg_basebackup; --force: N             
 2       | pg-slave  | repmgrd_reload            | t  | 2021-07-20 10:13:17 | monitoring cluster primary "pg-slave" (ID: 2)                                                
 2       | pg-slave  | repmgrd_failover_promote  | t  | 2021-07-20 10:13:17 | node "pg-slave" (ID: 2) promoted to primary; old primary "pg-master" (ID: 1) marked as failed
 2       | pg-slave  | standby_promote           | t  | 2021-07-20 10:13:17 | server "pg-slave" (ID: 2) was successfully promoted to primary                               
 2       | pg-slave  | repmgrd_start             | t  | 2021-07-20 10:11:50 | monitoring connection to upstream node "pg-master" (ID: 1)                                   
 1       | pg-master | repmgrd_local_reconnect   | t  | 2021-07-20 10:10:38 | reconnected to primary node after 44 seconds, resuming monitoring                            
 2       | pg-slave  | standby_register          | t  | 2021-07-20 10:09:27 | standby registration succeeded; upstream node ID is 1                                        
 2       | pg-slave  | standby_unregister        | t  | 2021-07-20 10:09:26 |                                                                                              
 1       | pg-master | child_node_reconnect      | t  | 2021-07-20 10:08:58 | standby node "pg-slave" (ID: 2) has reconnected after 899 seconds                            
 2       | pg-slave  | standby_clone             | t  | 2021-07-20 10:08:25 | cloned from host "pg-master", port 5432; backup method: pg_basebackup; --force: N            
 1       | pg-master | child_node_disconnect     | t  | 2021-07-20 09:53:59 | standby node "pg-slave" (ID: 2) has disconnected
```      

### 3.5.3. Register Failed Node
```
rm -rf /var/lib/postgresql/11/main
repmgr -h pg-slave -U repmgr -d repmgr standby clone
pg_ctl -D /etc/postgresql/11/main start
repmgr standby register
```
- Node master lúc nào sẽ chuyển qua role standby trong cluster
```
postgres@pg-slave:~$ repmgr cluster show
 ID | Name      | Role    | Status    | Upstream | Location | Priority | Timeline | Connection string                                         
----+-----------+---------+-----------+----------+----------+----------+----------+------------------------------------------------------------
 1  | pg-master | standby |   running |          | default  | 100      | 2        | host=pg-master user=repmgr dbname=repmgr connect_timeout=2
 2  | pg-slave  | primary | * running |          | default  | 100      | 2        | host=pg-slave user=repmgr dbname=repmgr connect_timeout=2 

```
## 3.6. Cấu hình Xinetd
- Cài đặt gói Xinetd
```
# apt install xinetd
```
### 3.6.1. Cấu hình các Node
- Tạo script `/opt/pgsqlchk` xinetd check port PostgreSQL
```
root@pg-master:~# cat /opt/pgsqlchk 
#!/bin/bash
PGBIN=/usr/pgsql-10/bin
PGSQL_HOST="localhost"
PGSQL_PORT="5432"
PGSQL_DATABASE="postgres"
PGSQL_USERNAME="postgres"
export PGPASSWORD="passwd"
TMP_FILE="/tmp/pgsqlchk.out"
ERR_FILE="/tmp/pgsqlchk.err"
 
VALUE=`psql -t -h localhost -U postgres -p 5432 -c "select pg_is_in_recovery()" 2> /dev/null`
# Check the output. If it is not empty then everything is fine and we return something. Else, we just do not return anything.
 
 
if [[ $VALUE == " t" ]]
then
    /bin/echo -e "HTTP/1.1 206 OK\r\n"
    /bin/echo -e "Content-Type: Content-Type: text/plain\r\n"
    /bin/echo -e "\r\n"
    /bin/echo "Standby"
    /bin/echo -e "\r\n"
elif [[ $VALUE == " f" ]]
then
    /bin/echo -e "HTTP/1.1 200 OK\r\n"
    /bin/echo -e "Content-Type: Content-Type: text/plain\r\n"
    /bin/echo -e "\r\n"
    /bin/echo "Primary"
    /bin/echo -e "\r\n"
else
    /bin/echo -e "HTTP/1.1 503 Service Unavailable\r\n"
    /bin/echo -e "Content-Type: Content-Type: text/plain\r\n"
    /bin/echo -e "\r\n"
    /bin/echo "DB Down"
    /bin/echo -e "\r\n"
fi
```
- Cấu hình xinetd
```
root@pg-master:/opt# cat /etc/xinetd.d/pgsqlchk
service pgsqlchk
{
        flags           = REUSE
        socket_type     = stream
        port            = 23267
        wait            = no
        user            = root
        server          = /opt/pgsqlchk
        log_on_failure  += USERID
        disable         = no
        only_from       = 0.0.0.0/0
        per_source      = UNLIMITED
}
```
- Restart Service
```
# service xinetd restart
```
## 3.. Cấu hình Haproxy
- Cài đặt trên các node Haproxy
```
# apt-get install haproxy
```
### 3.7.1. Cấu hình Haproxy trên các Node

- Cấu hình file `/etc/haproxy/haproxy.cfg`
```
global
    log 127.0.0.1 local0 debug
    maxconn 100

defaults
    log global
    mode tcp
    retries 2
    timeout client 30m
    timeout connect 4s
    timeout server 30m
    timeout check 5s

listen stats
    mode http
    bind *:7000
    stats enable
    stats uri /

listen ReadWrite
    bind *:5000
    option httpchk
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server pg-master 10.3.54.116:5432 maxconn 100 check port 23267
    server pg-slave 10.3.54.102:5432 maxconn 100 check port 23267

listen ReadOnly
    bind *:5001
    option httpchk
    http-check expect status 206
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server pg-master 10.3.54.116:5432 maxconn 100 check port 23267
    server pg-slave 10.3.54.102:5432 maxconn 100 check port 23267
```
- Restart Service
```
# service haproxy restart
```

## 3.8. Cấu hình Keepalive cho 2 node Haproxy
- Cài đặt gói keepalived
```
# apt install haproxy
```
### 3.8.1. Cấu hình keepalive cho Node Haproxy01
- Sửa file config `/etc/keepalived/keepalived.conf`
```
vrrp_script chk_haproxy {
  script "killall -0 haproxy"
  interval 2
  weight 2
}
vrrp_instance VI_1 {
  interface eth0
  priority 100
  virtual_router_id 23
  state MASTER
  virtual_ipaddress {
    10.3.54.139/22 dev eth0
  }
  authentication {
     auth_type PASS
     auth_pass 123456
     }
  track_script {
    chk_haproxy
  }
}
```
### 3.8.2. Cấu hình keepalive cho Node Haproxy02
- Sửa file config `/etc/keepalived/keepalived.conf`
```
vrrp_script chk_haproxy {
  script "killall -0 haproxy"
  interval 2
  weight 2
}
vrrp_instance VI_1 {
  interface eth0
  priority 99
  virtual_router_id 23
  state BACKUP
  virtual_ipaddress {
    10.3.54.139/22 dev eth0
  }
  authentication {
     auth_type PASS
     auth_pass 123456
     }
  track_script {
    chk_haproxy
  }
}
```
### 3.8.3. Restart keepalive và check
```
# service keepalived restart
```
## 3.9. Check với sysbench
```
# sysbench --db-driver=pgsql --report-interval=2 --oltp-table-size=100000 --oltp-tables-count=24 --threads=64 --time=60 --pgsql-host=10.3.54.139 --pgsql-port=5000 --pgsql-user=sbtest --pgsql-password=password --pgsql-db=sbtest /usr/share/sysbench/tests/include/oltp_legacy/oltp.lua run
```



## Reference:

- https://www.enterprisedb.com/postgres-tutorials/how-implement-repmgr-postgresql-automatic-failover


- https://docs.vmware.com/en/VMware-Cloud-Director/10.0/com.vmware.vcloud.install.doc/GUID-B3DFDBB7-1F53-4A94-96F2-7784A1E47A6D.html

- https://groups.google.com/g/repmgr/c/FVLdMRmT7tg`


```
setsebool -P haproxy_connect_any=1
``` 
nếu haproxy bị lỗi port
