##### hadoop-streaming 实现 hdfs wc -l 功能
```
#!/usr/bin/env bash
source /home/wirelessdev/.bash_profile;

sudo -uwirelessdev hadoop jar /home/q/hadoop/hadoop-2.2.0/share/hadoop/tools/lib/hadoop-streaming-2.2.0.jar \
-D mapred.job.name="hdfs-count_abecedarian" \
-D mapred.job.queue.name=wirelessdev \
-D mapred.job.priority=VERY_HIGH \
-D mapreduce.map.memory.mb=8192 \
-D stream.memory.limit=8192 \
-D mapred.child.map.java.opts=-Xmx2048m \
-D mapreduce.map.java.opts=-Xmx2048m \
-D mapreduce.reduce.memory.mb=8192 \
-D mapred.child.reduce.java.opts=-Xmx2048m \
-D mapreduce.reduce.java.opts=-Xmx2048m \
-D mapred.compress.map.output=true \
-D mapred.map.output.compression.codec=org.apache.hadoop.io.compress.GzipCodec \
-D mapred.output.compress=true \
-D mapred.output.compression.codec=org.apache.hadoop.io.compress.GzipCodec \
-D stream.map.output.field.seperator="\t" \
-D mapreduce.job.reduces=1 \
-input tmp/tianle.li/test/traffic_train.txt \
-output tmp/tianle.li/stream/ads/out/hdfs-count7 \
-mapper  "wc -l" \
-reducer "awk 'BEGIN{sum=0} {sum+=\$0} END{print sum}'"

```

##### hadoop-streaming 实现随机抽样功能
```
#!/usr/bin/env bash
source /home/wirelessdev/.bash_profile;

basedir=$(cd "$(dirname "$0")"; pwd)
basedir=$(cd "$(dirname "$basedir")"; pwd)
export TRANSDATA_HOME=$(cd "$(dirname "$basedir")"; pwd)


sudo -uwirelessdev hadoop jar /home/q/hadoop/hadoop-2.2.0/share/hadoop/tools/lib/hadoop-streaming-2.2.0.jar \
-D mapred.job.name="hdfs-count_abecedarian" \
-D mapred.job.queue.name=wirelessdev \
-D mapred.job.priority=VERY_HIGH \
-D mapreduce.map.memory.mb=8192 \
-D stream.memory.limit=8192 \
-D mapred.child.map.java.opts=-Xmx2048m \
-D mapreduce.map.java.opts=-Xmx2048m \
-D mapreduce.reduce.memory.mb=8192 \
-D mapred.child.reduce.java.opts=-Xmx2048m \
-D mapreduce.reduce.java.opts=-Xmx2048m \
-D mapred.compress.map.output=true \
-D mapred.map.output.compression.codec=org.apache.hadoop.io.compress.GzipCodec \
-D mapred.output.compress=true \
-D mapred.output.compression.codec=org.apache.hadoop.io.compress.GzipCodec \
-D stream.map.output.field.seperator="\t" \
-D mapreduce.job.reduces=1 \
-input tmp/tianle.li/test/traffic_train.txt \
-output tmp/tianle.li/stream/ads/out/hdfs-random \
-mapper  'cat'
-reducer 'python hdfs-random-reducer.py 1000' \
-file $TRANSDATA_HOME/run/tools/hdfs-random-mapper.py

其中hdfs-random-reducer.py 脚本为：

#!/usr/bin/env python
# encoding: utf-8

import sys
import hashlib
import random
import Queue

K=int(sys.argv[1])
i=K
result=Queue.Queue()
for line in sys.stdin:
    line = line.strip()
    if result.qsize()<K :
        result.put(line)
    else:
        i=i+1
        if random.randint(0,i)==1:
                result.get()
                result.put(line)

while not result.empty():
        print result.get()
```

##### hadoop-streaming 实现条件过滤查询
```
#!/usr/bin/env bash
source /home/wirelessdev/.bash_profile;

sudo -uwirelessdev hadoop jar /home/q/hadoop/hadoop-2.2.0/share/hadoop/tools/lib/hadoop-streaming-2.2.0.jar \
-D stream.non.zero.exit.is.failure=false \
-D mapred.job.name="hdfs-contain_tianle.li" \
-D mapred.job.queue.name=wirelessdev \
-D mapred.job.priority=VERY_HIGH \
-D mapreduce.map.memory.mb=8192 \
-D stream.memory.limit=8192 \
-D mapred.child.map.java.opts=-Xmx2048m \
-D mapreduce.map.java.opts=-Xmx2048m \
-D mapreduce.reduce.memory.mb=8192 \
-D mapred.child.reduce.java.opts=-Xmx2048m \
-D mapreduce.reduce.java.opts=-Xmx2048m \
-D mapred.compress.map.output=true \
-D mapred.map.output.compression.codec=org.apache.hadoop.io.compress.GzipCodec \
-D mapred.output.compress=true \
-D mapred.output.compression.codec=org.apache.hadoop.io.compress.GzipCodec \
-D stream.map.output.field.seperator="\t" \
-D mapreduce.job.reduces=1 \
-input tmp/tianle.li/test/traffic_train.txt \
-output tmp/tianle.li/stream/ads/out/hdfs-contain \
-mapper  "egrep '0c5531fbe2658274bf25b8f230abfce5'" \
-reducer "cat"

⚠️： stream.non.zero.exit.is.failure=false 参数一定要设置，关闭mapper和reducer的返回值不是0，被认为异常任务的情况。
```

##### hadoop-streaming 实现文件合并
```
#!/usr/bin/env bash
source /home/wirelessdev/.bash_profile;

input=$1
output=$2
coalesceNum=$3

sudo -uwirelessdev hadoop jar /home/q/hadoop/hadoop-2.2.0/share/hadoop/tools/lib/hadoop-streaming-2.2.0.jar \
-D stream.non.zero.exit.is.failure=false \
-D mapred.job.name="hdfs-coalesce_tianle.li" \
-D mapred.job.queue.name=wirelessdev \
-D mapred.job.priority=VERY_HIGH \
-D mapreduce.map.memory.mb=2048 \
-D mapred.child.map.java.opts=-Xmx2048m \
-D mapreduce.map.java.opts=-Xmx2048m \
-D mapreduce.reduce.memory.mb=8192 \
-D mapred.child.reduce.java.opts=-Xmx8192m \
-D mapreduce.reduce.java.opts=-Xmx8192m \
-D stream.memory.limit=8192 \
-D mapred.compress.map.output=true \
-D mapred.map.output.compression.codec=org.apache.hadoop.io.compress.Lz4Codec \
-D mapred.output.compress=true \
-D mapred.output.compression.codec=org.apache.hadoop.io.compress.GzipCodec \
-D stream.map.output.field.seperator="\t" \
-D mapreduce.job.reduces=1 \
-input ${input} \
-output ${output} \
-mapper  "cat" \
-reducer "cat"
```
