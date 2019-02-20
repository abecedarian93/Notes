##### 分析函数row_number,dense_rank,rank说明
```
row_number() 是没有重复值的排序(即使两天记录相等也是不重复的)，可以利用它来实现分页
dense_rank() 是连续排序，两个第二名仍然跟着第三名
rank()       是跳跃拍学，两个第二名下来就是第四名
```
##### 对表中某个字段去重,保留时间最近的数据-row_number()
```
select *
from
(
   select *,row_number() over (partition by uid order by logdate desc) num
   from table
)t
where t=1;
```
##### 字符串转时间戳
```
unix_timestamp(‘2009-03-20 12:23:31’, ‘yyyy-MM-dd HH:mm:ss’’)=1237532400
```
##### 字符串转Date
```
select from_unixtime(unix_timestamp('201801', 'yyyymm'),'yyyy-mm-dd');
```
##### 判断字符串时间str为周几(时间格式为yyyy-MM-dd)
```
pmod(datediff(str, '2012-01-01') , 7)
```

##### mapjoin
```
大表join小表，小表在前，小表数据会被缓存。
```
