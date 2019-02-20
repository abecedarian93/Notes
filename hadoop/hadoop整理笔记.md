##### hadoop常用压缩算法对比

压缩格式 | 类名 | split(是否可分割) | native (本地库) | 压缩率 | 压缩速度 | hadoop是否自带 | linux命令 | 换成压缩格式后，原来的程序是否需要修改 | 推荐使用场景
---|---|---|---|---|---|---|---|---|---
gzip | org.apache.hadoop.io.compress.GzipCodec | 否 | 是 | 很高(21%) |比较快(17.5MB/s-58MB/s)|是，直接使用|有|不需要修改|压缩率和速度均较好，可reducer端输出使用（文件在1个块大小内效果最佳）
lzo | org.apache.hadoop.io.compress.Lz4Codec | 是 | 是 | 比较高(34%) |很快(49.3MB/s-74.6MB/s)|否，需要安装|有|需要指定格式以及建索引|压缩速度要求较高的场景，可mapper端输出使用（单个文件越大，其优点越明显）
snappy | org.apache.hadoop.io.compress.SnappyCodec | 否 |是 | 比较高(38%) |很快(53.2MB/s-74.0MB/s)|否，需要安装|没有|不需要修改|压缩速度要求较高的场景，可mapper端输出使用 （中间数据压缩最常用）
bzip2 | org.apache.hadoop.io.compress.BZip2Codec | 是 |否 |最高(13%) |慢(2.4MB/s-9.5MB/s)|是，直接使用|有|不需要修改|对于压缩率要求较高可忽略效率的场景，可reducer端输出使用 （大大减少磁盘空间）


#### 列式存储和行式存储相比有哪些优势呢？
```
可以跳过不符合条件的数据，只读取需要的数据，降低IO数据量。
压缩编码可以降低磁盘存储空间。由于同一列的数据类型是一样的，可以使用更高效的压缩编码（例如Run Length Encoding和Delta Encoding）进一步节约存储空间。
只读取需要的列，支持向量运算，能够获取更好的扫描性能。
```

##### hdfs文件复制(distcp命令):
```
sudo -uwirelessdev hadoop distcp -Dmapred.job.queue.name=root.wirelessdev source_path aim_path
```
