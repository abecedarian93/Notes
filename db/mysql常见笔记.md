##### 本地文件加载到数据库中
```
LOAD DATA LOCAL INFILE '/Users/abecedarian/Documents/Data/robot/xing.txt' into table name_chinese_xing FIELDS TERMINATED BY '\t'
```
##### 数据备份
```
mysqldump --single-transaction -u$mysql_user -h$mysql_host -p"$mysql_password" $database > ${dbDir}/${database}_${day}.sql
```
