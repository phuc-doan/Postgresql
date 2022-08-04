## How to backup point-in-time with PSQL
### Prerequisite 
- Đã cài PSQL
- Đã có 1 lượng data DB nhất định
### Vào việc

#### *NOTE*
- Theo docs chính thức của postgresql thì việc dùng pg_basebackups hoàn toàn được khuyến khích nhưng việc dùng backupfilesystem thì vẫn thành công


### B1: Đảm bảo những parameter bên dưới đây đã được điều chỉnh như hình


![image](https://user-images.githubusercontent.com/83824403/182826123-36aabca9-aef8-409d-b94a-297cc493edfa.png)


```
psql (14.4 (Ubuntu 14.4-1.pgdg20.04+1))
Type "help" for help.

postgres=# show archive_mode ;
 archive_mode 
--------------
 on
(1 row)

postgres=# show archive_command ;
                     archive_command                      
----------------------------------------------------------
 test ! -f /backup/archive/%f && cp %p /backup/archive/%f
(1 row)

postgres=# show wal_level;
 wal_level 
-----------
 replica
(1 row)

postgres=# 
```


### B2: Tại thời điểm 10h, ta tạo 1 DB với 1 table bất kì, sau đó ta nạp vào DB 1 lượng data lớn nhất định (4GB DB) 

- Tạo DB, table

![image](https://user-images.githubusercontent.com/83824403/182827156-3be1ad2b-a018-4654-b87b-aa0f971e0698.png)


```
postgres=# create database phucdv;

CREATE DATABASE
postgres=# 
postgres=# \c phucdv;
You are now connected to database "phucdv" as user "postgres".
phucdv=# create table thong_tin(ten varchar(5), tuoi varchar(3), hocvan varchar(30));
phucdv=# insert into thong_tin values ( 'phuc','22','dh');
```

![image](https://user-images.githubusercontent.com/83824403/182827318-4c657b17-516e-4cc2-8dfb-58e510199ddf.png)

- Clone đâu được 1 doan DB đã được dump lại (4GB)
- Chạy lệnh restore DB này

```
postgres@psql:~/14/main/pg_wal$ psql < /mnt/all.sql 
```



- Việc làm này bởi vì mục đích muốn đẩy những thông tin thay đổi với db vào thư mục  test /backup/archive/. Bởi vì theo setting default thì size DB cần đạt đủ đến ngưỡng mới được postgres đọc đến archive_command và chuyển những file WAL này sang đó. Còn không nó vẫn ở **/var/lib/postgresql/14/main/pg_wal**

- show DB khi xong


![image](https://user-images.githubusercontent.com/83824403/182827670-f34d53fc-8fba-4ef8-8511-8c82c78fdf17.png)

### B3: Backup lại DB tại lúc 10:10

- Show DB size

![image](https://user-images.githubusercontent.com/83824403/182519013-179e73b3-30ab-4429-8d5e-739b0f00799a.png)

- Backup lại DB bằng filesystem ( Làm khác docs)

```
cd /var/lib/postgresql/14/main
tar -zcf backup10_10.tar.gz main #backup và nén lại
```

### B4 Ngay sau đó tại lúc 10:12 thêm 1 vài trường nữa vào DB phucdv, table thong_tin

```
phucdv=# insert into thong_tin values ( 'NVanA','23','giaosu');
INSERT 0 1
phucdv=# insert into thong_tin values ( 'NVanB','33','tiensi');
INSERT 0 1
phucdv=# select * from thong_tin;
  ten  | tuoi | hocvan 
-------+------+--------
 phuc  | 22   | dh
 NVanA | 23   | giaosu
 NVanB | 33   | tiensi
(3 rows)
```


![image](https://user-images.githubusercontent.com/83824403/182828534-3232fd07-87ce-48b1-b178-5874459eac4b.png)


### B5 THời điểm 10:15 xóa Table thong_tin trong DB phucdv

```
phucdv=# drop table thong_tin;
```


![image](https://user-images.githubusercontent.com/83824403/182519463-eb4159e7-9ee6-45c1-ad9c-3652c8d018d1.png)


### B6 Tiến hành recovery DB 
![image](https://user-images.githubusercontent.com/83824403/182829400-2defa580-d983-484d-9fcd-9584d14f8940.png)
