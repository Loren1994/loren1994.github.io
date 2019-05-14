---
layout: post
title: Elastic Search记录
date: 2019-05-14
tags: [ES,大数据]
author: loren1994
category: blog
---

# Elastic Search记录
[ES官方手册](https://www.elastic.co/guide/en/elasticsearch/reference/current/getting-started.html)

[ES教程1](<https://wangkang007.gitbooks.io/java/spring_data_elasticsearchjie_shao.html>)

[ES教程2](<https://endymecy.gitbooks.io/elasticsearch-guide-chinese/content/java-api/client.html>)

[ES教程3](<https://es.xiaoleilu.com/070_Index_Mgmt/50_Reindexing.html>)

##### ES介绍

ElasticSearch是一个基于Lucene的搜索服务器。它提供了一个分布式多用户能力的全文搜索引擎，基于RESTful web接口。Elasticsearch是用Java开发的，并作为Apache许可条款下的开放源码发布，是当前流行的企业级搜索引擎。Elasticsearch不仅仅是Lucene和全文搜索，我们还能这样去描述它：

- 分布式的实时文件存储，每个字段都被索引并可被搜索
- 分布式的实时分析搜索引擎
- 可以扩展到上百台服务器，处理PB级结构化或非结构化数据

Elasticsearch是**面向文档(document oriented)**的，这意味着它可以存储整个对象或**文档(document)**。然而它不仅仅是存储，还会**索引(index)**每个文档的内容使之可以被搜索。在Elasticsearch中，你可以对文档（而非成行成列的数据）进行索引、搜索、排序、过滤。这种理解数据的方式与以往完全不同，这也是Elasticsearch能够执行复杂的全文搜索的原因之一。

##### 基本概念

与关系型数据库的对比

~~~
关系数据库       ⇒ 数据库        ⇒ 表          ⇒ 行             ⇒ 列(Columns)

Elasticsearch  ⇒ 索引(Index)   ⇒ 类型(type)  ⇒ 文档(Docments)  ⇒ 字段(Fields)  
~~~

##### 使用方式

ES支持RESTful接口，项目使用springboot框架，推荐使用javaApi的调用方式。引入时可以使用spring的starter引入，更加简单。

* ##### 封装获取ES实例

~~~~ java
public static TransportClient getClient() {
        if (client == null) {
            Settings settings = Settings.settingsBuilder()
                    .put("thread_pool.generic.core", 5)
                    .put("thread_pool.generic.max", 10)
                    .put("processors", 5)
                    .put("cluster.name", clusterName).build();
            try {
                client = TransportClient.builder().settings(settings).build()
                        .addTransportAddress(new InetSocketTransportAddress(InetAddress.getByName(clusterHost), clusterPort));
            } catch (Exception e) {
                log.info("ES Connect Error:", e);
            }
        }
        return client;
}
~~~~

* ##### 单条插入

~~~~java
public static Boolean indexById(String index, String type, String id, String json) {
        TransportClient client = ESClient.getClient();
        IndexResponse response = null;
        try {
            if (id == null) {
                response = client.prepareIndex(index, type).setSource(json).execute().actionGet();
            } else {
                response = client.prepareIndex(index, type, id).setSource(json).execute().actionGet();
            }
        } catch (Exception e) {
            log.error("创建索引失败", e);
        }
        return response != null && response.getShardInfo().getSuccessful() == response.getShardInfo().getTotal();
}
~~~~

> 若插入很频繁，请使用批量插入，比如定时两秒批量插入一次两秒内的数据。ES插入效率不高，主要为查询操作。随着index的数据量增大插入时间也变长，所以也需要动态分index。
>
> 使用host:9200/_plugin/head/查看ES数据时，index内的数量以docs数量为准，因为不会实时显示出，特别是频繁且多量插入时。

* ##### 批量插入(可选择设置id)

~~~~java
public static Boolean bulkInsert(String index, String type, List<String> docs) {
        TransportClient client = ESClient.getClient();
        BulkRequestBuilder bulkRequest = client.prepareBulk();
        for (String doc : docs) {
            bulkRequest.add(client.prepareIndex(index, type).setSource(doc));
        }
        BulkResponse bulkResponse = bulkRequest.get();
        return bulkResponse.hasFailures();
}
~~~~

> 从ES 2.0开始，插入时若不存在索引会自动创建。

* ##### 查询返回json

~~~~java
public static List<String> getByQueryBean(String index, String type,
                                              BoolQueryBuilder fieldQuery) {
        List<String> result = new ArrayList<>();
        TransportClient client = ESClient.getClient();
        SearchResponse searchResponse = client.prepareSearch()
                .setIndices(index)
                .setTypes(type)
                .setQuery(fieldQuery)
                .setSize(10000)
                .execute()
                .actionGet();
        for (SearchHit hit : searchResponse.getHits().hits()) {
            result.add(hit.getSourceAsString());
        }
        return result;
}
~~~~

* ##### 多条件组合查询

~~~~java
val boolQuery = QueryBuilders.boolQuery();
//增加时间条件
boolQuery.must(QueryBuilders.rangeQuery("timestamp").gte(startTime).lte(endTime));
//指定字段的数组查询
boolQuery.must(QueryBuilders.termsQuery("device_code", deviceList));
//指定字段匹配
boolQuery.must(QueryBuilders.termQuery("name", name));
~~~~

> 注意termQuery和termsQuery的区别，一个传入一个匹配值，一个可传入List查询若干匹配值。

##### 踩坑记录

1.match query搜索的时候，首先会解析查询字符串，进行分词，然后查询，而term query,输入的查询内容是什么，就会按照什么去查询，并不会解析查询内容，对它分词。 

2.matchPhraseQuery和matchQuery等的区别，在使用matchQuery等时，在执行查询时，搜索的词会被分词器分词，而使用matchPhraseQuery时，不会被分词器分词，而是直接以一个短语的形式查询，而如果你在创建索引所使用的field的value中没有这么一个短语（顺序无差，且连接在一起），那么将查询不出任何结果。 

3.搜索时参数值中的字母全部转为小写搜索,所以接口中的传值要注意一律小写才可以搜索出结果。不会影响实际已经存入的值。

4.Springboot集成ES 

引入方式spring-boot-starter-data-elasticsearch或单独引入 

starter引入无需写ESClient等,单独引入需要写 

yml中配置ES服务要正确(cluster-name等) 

5.报错：

~~~~java
failed to map source XXX 
~~~~

解决办法:

Bean中加入无参构造方法 

这是因为在实体类中为了方便实例化添加了一个有参构造函数，导致JVM不能添加默认的无参构造函数了，但是jackson的反序列化需要使用无参构造函数，所以报错. 

6.与Redis冲突的问题。报错：

~~~~java
availableProcessors is already set to [4], rejecting [4] 
~~~~

解决办法：

~~~~java
public static void main(String[] args) {
    System.setProperty("es.set.netty.runtime.available.processors", "false");
    SpringApplication.run(CoderhubApplication.class, args);
}
~~~~

7.ES分页 

"浅"分页: 

from/size: 偏移值/返回量 

~~~~
{
"from" : 0, "size" : 10, 
"query" : { "term" : { "user" : "kimchy" }}
} 

~~~~

越往后的分页，执行的效率越低，也就是说，分页的偏移值越大，执行分页查询时间就会越长。 

深分页：

scroll分页通俗来说就是滚屏翻页，设置每页查询数量之后，每次查询会返回一个scroll_id，即当前文档的位置，下次查询再传这个scroll_id给es返回下一页的数据以及一个新的scroll_id，类似于按书页码顺序翻书和游标，遗憾的是不支持跳页。适用于不需要跳页持续批量拉取结果、对所有数据分页或一次性查询大量数据的场景。