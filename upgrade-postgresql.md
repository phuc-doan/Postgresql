## Upgrade Version lên một version mới bất kì.

- Giả sử hệ thống đang chạy Psql-9.6

![image](https://user-images.githubusercontent.com/83824403/174271090-6569fef1-e3d0-4a71-a275-7759ff7b2596.png)


- Target:


  +) Lên version mới nhất (14.4)
  
  +) Rollback về version (9.6) - Downgrade về sersion thấp hơn (Bất kỳ)
  
  +) Upgrade lại version (14.4)
  

  
  
## - Demo: Từ 9.6 lên 14.4

- Tạo 1 db mới là demodb để chút nữa sang ver 14.4 check thử xem có bị mất dữ liệu không

![image](https://user-images.githubusercontent.com/83824403/174279211-590e5d47-b67a-4816-976f-5222503f1b47.png)


- Add repo và cài psql bản 14.4 như bình thường

```
sudo apt update
sudo apt -y install gnupg2 wget vim
sudo apt-cache search postgresql | grep postgresql
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
sudo apt -y update
sudo apt -y install postgresql-14
```

- Sau khi xong thì server tồn tại 2 lib của 2 version


![image](https://user-images.githubusercontent.com/83824403/174273981-4c0b3973-8427-420c-8557-a29a39ab84a0.png)



- Tiến hành upgrade từ ver 9.6 lên 14


- Chạy pg_lsclusters, các cụm chính 9.6 và 14 của bạn phải "online".

![image](https://user-images.githubusercontent.com/83824403/174274328-f38bcf63-b989-4508-8f29-24c6761d8489.png)


- Thả psql version mới:

```
pg_dropcluster --stop 14 main
```

- Sau đó, di chuyển dữ liệu:

```
pg_upgradecluster -m upgrade 9.6 main
```
- Sau đó dừng phiên bản trước của PotsgreSQL:

```
pg_dropcluster 9.6 main --stop
```
- Xóa phiên bản cũ: (Có thể làm hoặc không)

```
apt-get autoremove --purge postgresql-9.6 
```
- Sau đó, khởi động lại PostgreSQL, phiên bản mới với cơ sở dữ liệu được di chuyển:

```
service postgresql restart
```


![image](https://user-images.githubusercontent.com/83824403/174275094-49eca0c6-507a-4c87-bffc-56c31b0a7e2d.png)


### =>> DB đang chạy ở verison 14.4 và list xem còn db tạo ở version 9.6 không



![image](https://user-images.githubusercontent.com/83824403/174279469-193befaa-0d9f-42ef-815e-533bd56d7fcb.png)



## Rollback ngược về version 9.6 



- Lúc này chỉ có psql 14.4 online

![image](https://user-images.githubusercontent.com/83824403/174279715-09e94eb6-da8a-4559-9425-d8d62cbf7d87.png)


- Nêu chưa chạy **apt-get autoremove --purge postgresql-9.6** thì có thể rollback đựơc luôn, còn không sẽ lại add repo và cài lại bản 9.6 bình thường

```
sudo apt-get update -y
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
echo "deb http://apt.postgresql.org/pub/repos/apt/ $(lsb_release -cs)-pgdg main" | sudo tee /etc/apt/sources.list.d/postgresql-pgdg.list > /dev/null
sudo apt-get update -y
sudo apt-get install postgresql-9.6
```




-  check các version psql đang hiện diện trong máy bằng **pg_lsclusters **




![image](https://user-images.githubusercontent.com/83824403/174280368-90513769-e100-4eb2-a96c-194721e7f27e.png)



- Các bước tương tự như trên nhưng nếu rollback thì phải dumpdb. DB `demo` đã tạo trước đó sẽ không auto từ bản 14.4 về 9.6

#### Tham khảo:

![image](https://user-images.githubusercontent.com/83824403/174282679-f40e4d82-d8ef-45b9-a289-0af5f550008e.png)


- Stop bản 14 cluster and drop it.

```
sudo pg_dropcluster 14 main --stop
```

- Bản 14 cluster đuọc hiển thị now be "down".
```
pg_lsclusters 
Ver Cluster Port Status Owner    Data directory               Log file
14 main    5433 down   postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log
9.6 main    5432 online postgres /var/lib/postgresql/9.6/main /var/log/postgresql/postgresql-9.6-main.log
```

- Check that the upgraded cluster works, then remove the 14 cluster.

sudo pg_dropcluster 14 main



- Database lúc này đã về bản 9.6 và không còn DB demo

![image](https://user-images.githubusercontent.com/83824403/174283207-2d70ac50-b8e3-4cd3-a4db-e51f13323570.png)


## Upgrade lại version khác (VD:13)

- Đừng từ version thấp hơn (9.6) vẫn tải và add repo của các version cao hơn như bình thường ( ở đây là VD với psql-13)

- Tạo 1 DB là **test** để chút nữa sang version 13 check

![image](https://user-images.githubusercontent.com/83824403/174284052-a91dbeef-426b-4ac6-a799-dcd50f970e33.png)


- Đừng từ version thấp hơn (9.6) vẫn tải và add repo của các version cao hơn như bình thường ( ở đây là VD với psql-13)

![image](https://user-images.githubusercontent.com/83824403/174284293-3880705a-1afe-4ac8-965c-fe8ffc4f89d8.png)



- Thả psql version mới:

```
pg_dropcluster --stop 13 main
```

- Sau đó, di chuyển dữ liệu:

```
pg_upgradecluster -m upgrade 9.6 main
```
- Sau đó dừng phiên bản trước của PotsgreSQL:

```
pg_dropcluster 9.6 main --stop
```
- Xóa phiên bản cũ: (Có thể làm hoặc không)

```
apt-get autoremove --purge postgresql-9.6 
```
- Sau đó, khởi động lại PostgreSQL, phiên bản mới với cơ sở dữ liệu được di chuyển:

```
service postgresql restart
```



- DB test vẫn còn nguyên và DB đã lên version 13 đúng mong muốn
![image](https://user-images.githubusercontent.com/83824403/174284560-8f3938b9-aff1-4836-9cab-f77475f7c462.png)


