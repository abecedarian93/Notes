##### elasticsearch开启跨域访问,在elasticsearch.yml配置文件中加入一下内容，然后重启服务
```
network.host: 0.0.0.0
http.cors.enabled: true
http.cors.allow-origin: "*"
```
##### 安装elasticsearch
```
docker pull elasticsearch:5.6.11

mkdir -p /Users/abecedarian/Documents/Docker/elasticsearch/config
mkdir -p /Users/abecedarian/Documents/Docker/elasticsearch/data

docker run -it --name elasticsearch -p 9200:9200 -p 9300:9300  -p 5601:5601 -e "discovery.type=single-node" \
-v /Users/abecedarian/Documents/Docker/elasticsearch/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml \
-v /Users/abecedarian/Documents/Docker/elasticsearch/data:/usr/share/elasticsearch/data -d elasticsearch:5.6.11
```

##### elasticsearch-head是elasticsearch的一个集群管理工具,安装elasticsearch-head  
```
docker run -p 9100:9100 -d mobz/elasticsearch-head:5
```

##### 安装Kibana
```
docker run -it -d -e ELASTICSEARCH_URL=http://127.0.0.1:9200 --name kibana --network=container:elasticsearch kibana:5.6.11
```
```
--network 指定容器共享elasticsearch容器的网络栈 (使用了--network 就不能使用-p 来暴露端口)
```
