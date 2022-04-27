
# Mô hình được đề cập là M-S/ M-M

- Now let's get started!

## Mô hình M-S Postgresql

- **reference:** https://www.cybertec-postgresql.com/en/setting-up-postgresql-streaming-replication/


![image](https://user-images.githubusercontent.com/83824403/165493902-1b7b0d4e-a389-45a0-b4ad-0659c7fbe809.png)

- Primary: 192.168.187.136 (node1)
- Secondary: 192.168.187.137 (node2)

### B1: Installing PostgreSQL 2 node

```
 yum -y install https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm
 yum -y install postgresql13 postgresql13-server
/usr/pgsql-13/bin/postgresql-13-setup initdb
systemctl restart postgresql-13
 systemctl enbale postgresql-13

```
### B2: Config node master

- Kiểm tra những gì chúng ta sẽ thấy bây giờ trên node1:

```
 ps axf | grep post
 ```
 
 
 - Và tắt luôn firewalld đi :v


*Coz The replica will connect to the master on port 5432. If the firewall is still active, the replica will not be able to access port 5432. In our example, the firewall will be disabled completely to make it easier for the reader. In a more secure setup, you might want to do this in a more precise manner.*



 - Đại khái nó như này:
 
 ![image](https://user-images.githubusercontent.com/83824403/165495628-2aa79879-83a7-4b85-925d-9283c6df5321.png)


### - *Trích dẫn*
#### After this process, you should have:

1. Installed binaries
- A fully working node1
- Systemd scripts set up
- Database instances initialized

2.A prepared 2nd node (node2)
- Binaries installed
- Systemd scripts set up

### step 3: Bốn việc cần làm trên node 1 :))

- Enable networking (bind addresses) in postgresql.conf


- Create a replication user (best practice but not mandatory)


- Allow remote access in pg_hba.conf


- Restart the primary server

### Sửa file conf của psql 

```
listen_addresses = '*'
```

![image](https://user-images.githubusercontent.com/83824403/165497446-ae676809-89f2-4f3e-9af1-cc5ccb604ddd.png)


### Then we can already create the user in the database:

```
postgres=# CREATE USER repuser REPLICATION;
CREATE ROLE
```

*Of course, you can also set a password. What is important here is that the user has the REPLICATION flag set. The basic idea is to avoid using the superuser to stream the transaction log from the primary to the replica.*

###  Edit file pg_hba.conf

- Đây là file mô tả các cài đặt xác thực liên quan đến máy client

```
host replication repuser 192.168.187.137/24 trust

```



- Đại khái nó như vậy:

![image](https://user-images.githubusercontent.com/83824403/165498278-05489717-2dab-4193-9feb-3e9c918298ab.png)

### Cuối cùng là restart 

```
systemctl restart postgresql-13
```


## Ở phía Slave

### B1: Stop psql-13

 ```
  systemctl stop postgresql-13
 ```
 
 
 - chúng ta cần đảm bảo rằng thư mục dữ liệu trống:

```
cd /var/lib/pgsql/13/data/
ls
rm -f *
ls #kiểm tra lại lần cuối
```


### B2: 

```
[root@node2 data]# su postgres
bash-4.4$ pwd
/var/lib/pgsql/13/data
bash-4.4$ pg_basebackup -h 192.168.187.136 -U repuser --checkpoint=fast \
      -D /var/lib/pgsql/13/data/ -R --slot=some_name -C
 ```

