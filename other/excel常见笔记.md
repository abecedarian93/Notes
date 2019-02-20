##### 根据相同列合并数据,若不存在则为0
```
=IF(ISERROR(VLOOKUP(A2,I:J,2,FALSE)),0,VLOOKUP(A2,I:J,2,FALSE))
```
