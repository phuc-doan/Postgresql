# Failover with Pacemaker/ DRDB

## Reference:

- *````https://www.enterprisedb.com/postgres-tutorials/postgresql-replication-and-automatic-failover-tutorial````*

- *```https://www.postgresql.vn/blog/pg_pacemaker_drbd```*

### Mô hình DB sẽ sử dụng là M-M
- DRBD (viết tắt của Distributed Replicated Block Device) sao chép dữ liệu trên các thiết bị chính cho các thiết bị phụ trong một cách mà đảm bảo rằng cả hai bản sao của dữ liệu vẫn còn giống hệt nhau.
