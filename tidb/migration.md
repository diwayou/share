# 迁移 TiDB

## 配置注意事项

- ONLY_FULL_GROUP_BY 需要关闭，因为代码大量 select 非 group by 字段
- @@sql_mode

## SQL 注意事项

- 使用 union 时候，null as [string]会出现检测不出编码而乱码，换成'' as [string]
- 自增主键不是全局有序
- 不能在单条 ALTER TABLE 语句中完成多个操作
- CONVERT 按照 GBK 排序不支持，因为 tidb 只支持 ascii/latin1/binary/utf8/utf8mb4 字符集

## 限制

|    名称    |    限制    |
| :--------: | :--------: |
| 单行的限制 |    6MB     |
| 单列的限制 |    6MB     |
|    CHAR    |  256 字符  |
|   BINARY   |  256 字节  |
| VARBINARY  | 65535 字节 |
|  VARCHAR   | 16383 字符 |
|    TEXT    |  6MB 字节  |
|    BLOB    |  6MB 字节  |

## explain
