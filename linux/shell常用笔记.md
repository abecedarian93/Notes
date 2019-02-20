##### 查看当前目录下的子文件和子目录占用的磁盘容量:
```
sudo du -h --max-depth=1 /home/q |grep [TG]|sort -nr
```
##### 查看当前目录下所有修改时间不在指定时间内的文件并删除

```
find /home/q/tianle.li/data -maxdepth 1  -mtime +10  -name "*" -delete
```
##### 时间
```
day=`date +'%Y%m%d' -d $1`
day2=`date +'%Y-%m-%d' -d "$day"`
year=${day:0:(4-0)}
month=${day:4:(6-4)}
date=${day:6:(8-6)}
before_day=`date +'%Y%m%d' -d "$day 1 day ago"`
```

##### awk去重
```
awk '!a[$0]++' fileName
```

##### 按指定时间循环执行脚本
```
##开始时间默认为昨天
if [ -n "$1" ]
then
        start_day=`date +'%Y%m%d' -d $1`
else
        start_day=`date +"%Y%m%d" -d "-1 days"`
fi

###### 结束时间默认为昨天
if [ -n "$2" ]
then
        end_day=`date +'%Y%m%d' -d $2`
else
        end_day=`date +"%Y%m%d" -d "${day} -1 days"`
fi

run_day=`date +'%Y%m%d' -d "$start_day"`

while [[ ${run_day} -le ${end_day} ]]
do
        echo "$run_day run"
        //脚本
        run_day=`date +"%Y%m%d" -d "$run_day +1 days"`

done
```

##### 字符串截取
```
echo "hello world" |cut -c2-8
```

##### nohup后台执行命令
```
nohup sh script.sh 2>&1 &
```

##### 服务器账户相关
```
添加一个用户:
adduser abecedarian
设置密码:
passwd abecedarian
添加一个组:
groupadd dev
给已用的用户指定组:
usermod -G dev abecedarian
添加sudo用户:
vim /etc/sudoers
更改文件用户:
sudo chown tianle.li tianle.li/  
更改文件组:
sudo chgrp qunarengineer tianle.li/  

```

##### sed /n替换为,
```
sudo -uwirelessdev hadoop fs -ls log/ads/show  |awk -F' ' '{if(length($8)>0) print $8}' |sed ':a;N;$!ba;s/\n/,/g'
```

##### split文件分割
```
split -l10000 file newFile
```

##### 获取root权限
```
ln -s /bin/sh xxx
sudo ./xxx
```
##### tar压缩解压
```
tar -zcvf tar-archive-name.tar.gz source-folder-name
tar -zxvf tar-archive-name.tar.gz
```
