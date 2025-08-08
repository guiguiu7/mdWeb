### 概述

> ES是一个开源的高扩展的**分布式全文检索引擎**，它可以近乎**实时的存储、检索数据**；本身扩展性很好，可以扩展到上百台服务器，处理PB级别（大数据时代）的数据。ES也使用Java开发并使用Lucene作为其核心来实现所有索引和搜索的功能，但是它的目的是通过简单的**RestFul API**来隐藏Lucene的复杂性，从而让全文检索变得简单。
>

### docker部署

需要先创建data、config、plugins、logs文件夹，并且需要创建配置文件

```bash
echo "http.host:0.0.0.0" > /home/elaticsearch/config/elaticsearch.yml
```

```bash
docker run --name elasticsearch -p 9200:9200 -p 9300:9300 \
--restart=always \
-e  "discovery.type=single-node" \
-e ES_JAVA_OPTS="-Xms64m -Xmx256m" \
-v /home/elasticsearch/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml \
-v /home/elasticsearch/data:/usr/share/elasticsearch/data \
-v /home/elasticsearch/plugins:/usr/share/elasticsearch/plugins \
-v /home/elasticsearch/logs:/usr/share/elasticsearch/logs \
-d elasticsearch:9.0.2
```

8.0.x版本后默认开启认证  如果需要关闭功能需要

```bash
  -e "network.host=0.0.0.0" \
  -e "xpack.security.enabled=false" \
  -e "discovery.type=single-node" \
或者
echo "network.host: 0.0.0.0
xpack.security.enabled: false
xpack.security.http.ssl.enabled: false" > /home/elasticsearch/config/elasticsearch.yml
```

### 可视化界面kibana

### docker部署

```bash
docker run -d --name kibana -p 5601:5601 kibana:9.0.2
```

### 文档操作API

https://elasticsearch.bookhub.tech/rest_apis/document_apis/

### IK分词器安装

https://github.com/infinilabs/analysis-ik找到与ES相同的版本安装到plugins文件夹中(注意需要解压到同一个文件夹中)

```bash
wget https://release.infinilabs.com/analysis-ik/stable/elasticsearch-analysis-ik-9.0.2.zip
unzip 文件名
docker exec -it XXX /bin/bash
cd /bin
elasticsearch-plugin list 查看是否安装成功
```

测试是否能够使用

```json
POST _analyze
{
  "analyzer": "ik_max_word", #"ik_smart"
  "text": "今天天气不错啊！"
}
```

##### 设置自定义分词

利用nginx代理自定义分词，将分词文件（xx.txt）放到nginx下的html目录中，随后修改ES的config中的IKAnalyzer.cfg.xml设置连接指定远程扩展字典

##### TIP

如果要保存Array的数据，**ES默认会进行扁平化处理**，为了避免该行为可以将类型设置为nested



### SpringBoot整合ES

##### 导入依赖

```xml
<dependency>
	<groupId>co.elastic.clients</groupId>
	<artifactId>elasticsearch-java</artifactId>
	<version>9.0.2</version>
</dependency>
```

##### 配置ES

```java
@Configuration
public class MyESConfig {
    @Bean
    public ElasticsearchClient ESConfig() {
        return ElasticsearchClient.of(builder -> builder.host("http://103.210.237.3:9200"));
    }
}
```

#### TIP

查询出来的数据可以为**Map、ObjectNode或者是自定义类**

如果是**嵌入式的属性，查询、聚合、分析都要应用嵌入式**
