# 1\. Giới thiệu

PostgreSQL streaming replication sử dụng service Repmgr để cấu hình replication, failover phục vụ nhu cầu HA cho Database Postgre Cluster
Repmgr là một tool opensource dùng dể quản lý replication và failovẻ trong cluster PostgreSQL Server. Nó cung cấp PostgreSQL hot-standby để setup standby server, monitor replication, và thực hiện chuyển đổi dự phòng tự động hoặc thủ công

# 2\. Replication trong PostgreSQL hoạt động thế nào ?

- Trong 1 cụm setup replication PostgreSQL gồm 2 kiểu server (Master & Slave). Các bản ghi (record) database được Slave sao chép từ Master. Các record này có thể đọc(read) từ Slave Server, nhưng chỉ có thể được ghi (write) trên Master Server. Các server được sync với nhau. Khi một Master Server fail, Slave Server sẽ đứng lên giữ vai trò Master trong cụm.
    
- Replication Flow

# 3\. Cấu hình PostgreSQL master-standby sử dụng repmgr


![image](https://user-images.githubusercontent.com/83824403/165925482-9a525032-e748-4f76-8b36-df20e30e1cbb.png)


## Let's get started!

1. Install PostgreSQL

2. Install repmgr

3. Configure PostgreSQL

4. Create users

5. Configure pg_hba.conf

6. Configure the repmgr file

7. Register the primary server

8. Build/clone the standby server

9. Register the standby server

10. Start repmgrd daemon process






### `Prerequisites`

**The following software must be installed on both master and standby servers:**

- PostgreSQL
- repmgr (matching the installed PostgreSQL major version)
- At the network level, connections with the PostgreSQL port (default: 5432) must be possible in both directions.


**- Primary: 192.168.187.141**
**- Standby: 192.168.187.131**


## B1: Install PostgreSQL


- Create two clusters/servers with the PostgreSQL installation.
- You can follow the PostgreSQL instructions at the link below for installation using PostgreSQL’s PGDG repo package. For the sake of naming conventions, we will consider master and standby as two servers.

```
yum -y install https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm

yum -y install epel-release yum-utils

yum-config-manager --enable pgdg12

yum install postgresql12-server postgresql12

/usr/pgsql-12/bin/postgresql-12-setup initdb
```

- *Note: The above step of initialization of the cluster is not needed on the standby server.*

```
systemctl enable --now postgresql-12

systemctl status postgresql-12
```


## B2: Install repmgr 


- You will need to install repmgr on the master as well as standby.


```
yum -y  install repmgr12*
```

## Step 3: Configure PostgreSQL


- On the primary server, a PostgreSQL instance must be initialized and running. 
- The following replication settings may need to be adjusted:

```
max_wal_senders = 10

max_replication_slots = 10

wal_level = 'hot_standby' or 'replica' or 'logical'

hot_standby = on

archive_mode = on

archive_command = '/bin/true'

shared_preload_libraries = 'repmgr'
```


- Thêm cả phần `listen = IP server` để nó connect hoặc 'listen= '*' '



![image](https://user-images.githubusercontent.com/83824403/165926505-d061f0e8-1133-4f75-8ace-e0a3feaae484.png)


## Step 4: Create users 

- Create a dedicated PostgreSQL superuser account and a database for the repmgr metadata:

![image](https://user-images.githubusercontent.com/83824403/165927736-cceaa3a9-98d1-4fed-b2ba-70de6ef10af3.png)




```
create user repmgr;

create database repmgr with owner repmgr;
ALTER USER repmgr WITH NOSUPERUSER;

```
- Lúc đầu mình ko phân quyền cho repmgr nó báo lỗi ngay, nên phải bổ sung. Lỗi được resolve bởi :`https://chartio.com/resources/tutorials/how-to-change-a-user-to-superuser-in-postgresql/`


![image](https://user-images.githubusercontent.com/83824403/165926918-05373e10-79a9-4d9e-af11-5b197afda741.png)


## Step 5:Configure pg_hba.conf

- Đại khái nó như này


- ![image](https://user-images.githubusercontent.com/83824403/165927060-22158fe8-5e5a-48b1-b22f-bc4b1021e28a.png)



## Step 6: Configure the repmgr file 


- Create a repmgr.conf on the master server with the following entries:

```
cluster='failovertest'

node_id=1

node_name=node1

conninfo='host=node1 user=repmgr dbname=repmgr connect_timeout=2'

data_directory='/var/lib/pgsql/12/data/'

failover=automatic

promote_command='/usr/pgsql-12/bin/repmgr standby promote -f /var/lib/pgsql/repmgr.conf --log-to-file'

follow_command='/usr/pgsql-12/bin/repmgr standby follow -f /var/lib/pgsql/repmgr.conf --log-to-file --upstream-node-id=%n'
```

![image](https://user-images.githubusercontent.com/83824403/165927500-274edf38-906d-4e15-8421-a5bb78e8ff09.png)



*Note: The location and file must be accessible for the user we are using for repmgr.*



## Step 7: Register the primary server


- Register the primary server with repmgr:


```
/usr/pgsql-12/bin/repmgr -f /etc/repmgr/12/repmgr.conf primary register
/usr/pgsql-12/bin/repmgr -f /etc/repmgr/12/repmgr.conf cluster show
```

- Dai khai no nhu vay 


![image](https://user-images.githubusercontent.com/83824403/165927778-b3725097-0997-4fd0-b71b-428df616947f.png)



## Step 8: Build/clone the standby server


- Create the repmgr.conf file on standby server




```
node_id=2

node_name=slave2

conninfo='host=slave2 user=repmgr dbname=repmgr connect_timeout=2'

data_directory='/var/lib/pgsql/12/data'

failover=automatic

promote_command='/usr/pgsql-12/bin/repmgr standby promote -f /var/lib/pgsql/repmgr.conf --log-to-file'

[2022-04-29 17:27:38] [INFO] node "slave2" (ID: 2) monitoring upstream node "master2" (ID: 1) in normal statem-node-id=%n'


```

*dam bao rang file host da duoc tro dung ve ip cua 2 server primary va standby



![image](https://user-images.githubusercontent.com/83824403/165929287-0e66b255-f023-4d72-bbba-f84ef2258bbb.png)




## Step 9: Register the standby server


- Register the standby server with repmgr:

```
 /usr/pgsql-12/bin/repmgr -h 192.168.187.141 -U repmgr -d repmgr -f /etc/repmgr/12/repmgr.conf standby clone
/usr/pgsql-12/bin/repmgr -f /etc/repmgr/12/repmgr.conf standby register
```


- Sau do thi gap loi nhu nay la do file /etc/repmgr/12/repmgr.conf chua dat gia tri listen. chuyen sang listen '*' la ok 


![image](https://user-images.githubusercontent.com/83824403/165928290-9e93b698-fe54-4340-b166-d560546b69dd.png)

tiep theo

```
/usr/pgsql-12/bin/repmgr -f /etc/repmgr/12/repmgr.conf cluster show
```

- Dai khaai no nhu nay

![image](https://user-images.githubusercontent.com/83824403/165928640-396170e3-9c86-4eee-97e6-14944e569e73.png)


## Step 10: Start repmgrd daemon process
- To enable the automatic failover, we now need to start the repmgrd daemon process on master slave and witness:

For example:

```
/usr/pgsql-12/bin/repmgrd -f /etc/repmgr/12/repmgr.conf
/usr/pgsql-12/bin/repmgr -f /etc/repmgr/12/repmgr.conf cluster event
```
- Dai khai no nhu the nay

![image](https://user-images.githubusercontent.com/83824403/165928801-6667c8f3-f75a-4728-baf3-f03e0a05a3a3.png)

## Step 11: test thôi

- Tạo 1 db trên master xem slave đồng bộ
- Kết quả nhuư sau

![image](https://user-images.githubusercontent.com/83824403/165928949-8e2b6ea5-15c7-456d-931b-af8f6eb9a21f.png)


- Và nếu ở trên slave tạo DB thì sẽ bị lỗi RO như này là được

![image](https://user-images.githubusercontent.com/83824403/165929057-e5b6ee68-e139-4e5d-ae7b-10cee012ad5c.png)















## Reference:

- https://www.enterprisedb.com/postgres-tutorials/how-implement-repmgr-postgresql-automatic-failover












