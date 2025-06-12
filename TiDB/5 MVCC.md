前文已经提到了TiDB的TSO和GC。
TSO是MVCC的必要条件，用来追踪和管理数据的版本。
而GC则为历史数据提供了一个生命周期管理。

# 读取历史数据
TiDB提供了两种方法读取历史数据，不管哪种方法，TiDB都会追踪范围内的所有数据选择合适的时间段来保证数据的事物一致性。
## 语句级别

在SQL语句中添加 `AS OF TIMESTAMP`子句来进行历史数据的读取，配合`TIDB_BOUNDED_STALENESS`函数可以选取一个时间段，而不是一个时间点。
## 会话级别
通过在会话中设置系统变量，可以让该会话的所有查询按照变量设置的时间点进行查询。

### 用dumpling导出历史数据
```
tiup dumpling -u root -P 4000 -p $PASSWORD -h TiDB_ENDPOINT \
--filetype sql -t 8 -o /home/xxxxxxxxxx/pre-data  -r 200000 -F 256MiB \
--filter "database.table"  \
 --snapshot "2025-03-07 06:10:00" --where "id=1234567890123"   --no-schemas 
```

