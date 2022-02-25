# ElasticSearch快速开始
> 原文地址: https://www.elastic.co/guide/en/elasticsearch/reference/7.17/getting-started.html

## 快速开始
本文主要提供一下内容：
在测试环境安装并运行Elasticsearch
添加数据至Elasticsearch
数据搜索和排序
从非结构化搜索数据中提取字段

## 运行Elasticsearch
运行Elasticsearch最简单的方式是在Elastic云上部署管理Elasticsearch服务。如果您偏好自主管理，推荐通过docker安装管理。

<!-- tabs:start -->
#### **Elasticsearch服务**
 - [获取免费试用](https://es.xingquzu.com/cloud/elasticsearch-service/signup?baymax=docs-body&elektra=docs)
 - 登录[Elastic云](https://cloud.elastic.co/?baymax=docs-body&elektra=docs)
 - 点击<strong class="bold">Create deployment</strong>
 - 设置部署名称
 - 点击<strong class="bold">Create deployment</strong>并下载elastic用户的密码
 - 点击<strong class="bold">continue</strong>打开Kibana
 - 点击<strong class="bold">Explore on my own</strong>

#### **自主管理**
##### 安装并运行Elasticsearch
 - 安装并启动[Docker应用](https://www.docker.com/products/docker-desktop)
 - 运行：
    ```shell
   docker network create elastic
   docker pull docker.elastic.co/elasticsearch/elasticsearch:7.17.0
   docker run --name es01-test --net elastic -p 127.0.0.1:9200:9200 -p 127.0.0.1:9300:9300 -e "discovery.type=single-node" docker.elastic.co/elasticsearch/elasticsearch:7.17.0
   ```

##### 安装并运行Kibana
安装Kibana可以更直观的协助我们可视化分析管理Elasticsearch数据.

 - 新开终端,执行：
 ```shell
 docker pull docker.elastic.co/kibana/kibana:7.17.0
 docker run --name kib01-test --net elastic -p 127.0.0.1:5601:5601 -e "ELASTICSEARCH_HOSTS=http://es01-test:9200" docker.elastic.co/kibana/kibana:7.17.0
 ```
 - 本地进入Kibana,地址：[http://localhost:5601](http://localhost:5601)

<!-- tabs:end -->

## 请求Elasticsearch

