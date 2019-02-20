##### 建表语句(外表):
```
create external table  if not exists tmp_ads_feature_click
(
 line string
)
comment 'tmp_ads_feature_click'
partitioned by (t string)
row format delimited
fields terminated by '\t'
lines terminated by '\n'
stored as textfile
location 'hdfs://qunarcluster/user/wirelessdev/result/adx/feature/click_feature';
```
##### 指定HDFS数据加载到外表
```
alter table tmp_model_estimate add if not exists partition(t='cvr_20180720') location '/user/wirelessdev/result/ad_toutiao/train/growth_cvr/v1/20180720/raw';   
```
##### 指定本地数据加载到外表
```
load data local inpath '/home/q/tianle.li/data/material.txt' into table tmp_tow_tuple partition(t='material_department')
```
##### 卸载指定分区
```
alter table tmp_ads_feature_click drop if exists partition (t='20181031');
```

##### 查看分区信息
```
describe formatted tmp_ads_feature_click partition (t='20181031');
```

##### 设置内存参数
```
set mapreduce.map.memory.mb=12288;
set mapred.child.map.java.opts=-Xmx12288m;
set mapreduce.map.java.opts=-Xmx12288m;
set mapreduce.reduce.memory.mb=12288;
set mapred.child.reduce.java.opts=-Xmx12288m;
set mapreduce.reduce.java.opts=-Xmx12288m;
```

##### 添加简单UDF函数
```
add jar /home/q/tianle.li/data_ml/data_ml_release/wireless-data_ml-1.0.0-jar-with-dependencies.jar;
create temporary function ipLevel as 'com.qunar.mobile.hive.IpUtilUDF';
```
##### 查询结果导出本地
```
insert overwrite local directory '/home/q/tianle.li/data/material'
row format delimited fields terminated by ","
select * from tmp_tow_tuple where t='material_department' limit 10;
```
##### 查询结果导出HDFS
```
insert overwrite directory 'tmp/tianle.li/test'
row format delimited fields terminated by "\t"
select * from tmp_tow_tuple where t='material_department' limit 10;
```
##### 
