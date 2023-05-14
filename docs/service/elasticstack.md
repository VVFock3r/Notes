# Elastic Stack

官网：[https://www.elastic.co](https://www.elastic.co)

<br />

## Docker部署

下载地址：[https://www.docker.elastic.co](https://www.docker.elastic.co)

::: details （1）部署 ElasticSearch

指定版本文档：[https://www.elastic.co/guide/en/elasticsearch/reference/6.8/docker.html](https://www.elastic.co/guide/en/elasticsearch/reference/6.8/docker.html)

最新版本文档：[https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html](https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html)

```bash
# 拉取镜像
docker pull docker.elastic.co/elasticsearch/elasticsearch:6.8.6

# 启动服务(开发模式)
docker container run --name elasticsearch \
    -p 9200:9200 \
    -p 9300:9300 \
    -e "discovery.type=single-node" \
    --restart=always \
    -d \
  docker.elastic.co/elasticsearch/elasticsearch:6.8.6
  
# 测试
[root@localhost ~]# curl http://127.0.0.1:9200/_cat/health
1684026741 01:12:21 docker-cluster green 1 1 0 0 0 0 0 0 - 100.0%

# 配置文件目录
# /usr/share/elasticsearch/config
```

:::

::: details （2）部署 Kibana

指定版本文档：[https://www.elastic.co/guide/en/kibana/6.8/docker.html](https://www.elastic.co/guide/en/kibana/6.8/docker.html)

最新版本文档：[https://www.elastic.co/guide/en/kibana/current/docker.html](https://www.elastic.co/guide/en/kibana/current/docker.html)

```bash
# 拉取镜像
docker pull docker.elastic.co/kibana/kibana:6.8.6

# 启动服务(开发模式)
# 备注：可以使用docker自定义网络通过DNS通信,避免写死elasticsearch主机地址
docker container run --name kibana \
    -p 5601:5601 \
    -e SERVER_HOST=0.0.0.0 \
    -e SERVER_PORT=5601 \
    -e ELASTICSEARCH_HOSTS=http://192.168.48.129:9200 \
    --restart=always \
    -d \
  docker.elastic.co/kibana/kibana:6.8.6

# 配置文件目录
# /usr/share/kibana/config

# 修改语言为中文(临时,未做持久化)
[root@localhost ~]# docker exec -it kibana sh
sh-4.2$ vi /usr/share/kibana/config/kibana.yml       # 修改配置
i18n.locale: "zh-CN"                                 # 添加这行
[root@localhost ~]# docker container restart kibana  # 重启容器
```

![image-20230514095219800](https://tuchuang-1257805459.cos.accelerate.myqcloud.com//image-20230514095219800.png)

:::

::: details （3）部署 FileBeat

指定版本文档：[https://www.elastic.co/guide/en/beats/filebeat/6.8/running-on-docker.html](https://www.elastic.co/guide/en/beats/filebeat/6.8/running-on-docker.html)

最新版本文档：[https://www.elastic.co/guide/en/beats/filebeat/current/running-on-docker.html](https://www.elastic.co/guide/en/beats/filebeat/current/running-on-docker.html)

**1、拉取镜像，并执行setup子命令进行初始化**

```bash
# 拉取镜像
docker pull docker.elastic.co/beats/filebeat:6.8.6

# 执行 setup 子命令
docker container run --rm docker.elastic.co/beats/filebeat:6.8.6 setup \
    -E setup.kibana.host=http://192.168.48.129:5601 \
    -E output.elasticsearch.hosts=["http://192.168.48.129:9200"]

Loaded index template
Loading dashboards (Kibana must be running and reachable)
Loaded dashboards
Loaded machine learning job configurations

# 解释:
# 1、加载索引模板: Filebeat 将加载与其版本相对应的索引模板, 索引模板定义了将日志数据索引到 Elasticsearch 时的字段映射和设置
# 2、加载仪表板:   Filebeat 将尝试加载与其版本相对应的 Kibana 仪表板, 仪表板提供了可视化和交互式的界面，用于显示和分析采集到的日志数据
# 3、加载机器学习作业配置: 如果启用了机器学习相关功能，Filebeat 还将加载与其版本相对应的机器学习作业配置
```

**2、下载配置文件并启动服务**

```bash
# 下载配置文件: 自动发现Docker容器并发送日志到ElasticSearch
[root@localhost ~]# wget -c https://raw.githubusercontent.com/elastic/beats/6.8/deploy/docker/filebeat.docker.yml -O /etc/filebeat.yaml
[root@localhost ~]# cat /etc/filebeat.yaml
filebeat.config:
  modules:
    path: ${path.config}/modules.d/*.yml
    reload.enabled: false

filebeat.autodiscover:
  providers:
    - type: docker
      hints.enabled: true

processors:
- add_cloud_metadata: ~

output.elasticsearch:
  hosts: '${ELASTICSEARCH_HOSTS:elasticsearch:9200}'
  username: '${ELASTICSEARCH_USERNAME:}'
  password: '${ELASTICSEARCH_PASSWORD:}'

# 启动 filebeat
docker container run --name=filebeat  \
    --user=root \
    --volume="/etc/filebeat.yaml:/usr/share/filebeat/filebeat.yml:ro" \
    --volume="/var/lib/docker/containers:/var/lib/docker/containers:ro" \
    --volume="/var/run/docker.sock:/var/run/docker.sock:ro" \
    --restart=always \
    -d \
  docker.elastic.co/beats/filebeat:6.8.6 filebeat \
     -e \
     -strict.perms=false \
     -E output.elasticsearch.hosts=["http://192.168.48.129:9200"]
```

**3、在kibana上查看filebeat容器的日志**

![image-20230514163848865](https://tuchuang-1257805459.cos.accelerate.myqcloud.com//image-20230514163848865.png)

:::

<br />

## ElasticSearch

### 基础操作

::: details （1）文档的增删改查

```bash
# 这是系统自带的, 不用管
GET _search
{
  "query": {
    "match_all": {}
  }
}

# --------------------------------------------------

# 新增文档
POST /account/person/1
{
  "name": "bob",
  "age": "21"
}


# 查询文档
GET /account/person/1


# 修改文档
POST /account/person/1/_update
{
  "doc": {
      "name": "bob",
      "age": "22"
  }
}


# 删除文档
DELETE /account/person/1
```

输出结果

```json
// 新增文档
{
  "_index" : "account",
  "_type" : "person",
  "_id" : "1",
  "_version" : 1,
  "result" : "created",
  "_shards" : {
    "total" : 2,
    "successful" : 1,
    "failed" : 0
  },
  "_seq_no" : 0,
  "_primary_term" : 1
}

// 查询文档
{
  "_index" : "account",
  "_type" : "person",
  "_id" : "1",
  "_version" : 1,
  "_seq_no" : 0,
  "_primary_term" : 1,
  "found" : true,
  "_source" : {
    "name" : "bob",
    "age" : "21"
  }
}

// 修改文档
{
  "_index" : "account",
  "_type" : "person",
  "_id" : "1",
  "_version" : 2,
  "result" : "updated",
  "_shards" : {
    "total" : 2,
    "successful" : 1,
    "failed" : 0
  },
  "_seq_no" : 1,
  "_primary_term" : 1
}

// 删除文档
{
  "_index" : "account",
  "_type" : "person",
  "_id" : "1",
  "_version" : 3,
  "result" : "deleted",
  "_shards" : {
    "total" : 2,
    "successful" : 1,
    "failed" : 0
  },
  "_seq_no" : 2,
  "_primary_term" : 1
}

// 删除文档后再次查询
{
  "_index" : "account",
  "_type" : "person",
  "_id" : "1",
  "found" : false
}
```

:::

::: details （2）数据查询

```bash
# 使用查询字符串, 默认区分大小写
GET /account/person/_search?q=bob

# 使用DSL查询语法, 默认不区分大小写
GET /account/person/_search
{
  "query": {
    "match": {
      "name": "bob"
    }
  }
}
```

输出结果

```json
{
  "took" : 1,
  "timed_out" : false,
  "_shards" : {
    "total" : 5,
    "successful" : 5,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : 1,
    "max_score" : 0.2876821,
    "hits" : [
      {
        "_index" : "account",
        "_type" : "person",
        "_id" : "1",
        "_score" : 0.2876821,
        "_source" : {
          "name" : "bob",
          "age" : "21"
        }
      }
    ]
  }
}
```

:::

<br />

### 1