## springboot-elasticsearch
> 基于springboot的web项目，通过elasticsearch提供的Java API 进行查询操作.

#### 起因
项目在一个查询要在亚秒级计算（分组、累加、平均）大量数据的结果。官方提供的API过于简单，自己在做项目中遇到了一些坑，并总结了一些API的使用，简单分享一下。

#### 前置条件
有一个elasticsearch服务

#### 配置
> demo是基于springboot快速构建了一个web应用。elasticsearch提供了一个客户端:`TransportClient`,首先我们来配置一下

##### 如果在main方法里面执行可以直接创建一个客户端连接
```
TransportClient client = new PreBuiltTransportClient(Settings.EMPTY)
                .addTransportAddress(new InetSocketTransportAddress(InetAddress.getByName("172.16.3.121"), 9300));
```

##### 如果在springboot项目中。配置一下，这样我们可以直接注入使用了。

```
@Configuration
public class ElasticSearchConfig {

    @Value("${spring.elasticsearch.host}")
    private String host;//elasticsearch的地址

    @Value("${spring.elasticsearch.port}")
    private Integer port;//elasticsearch的端口

    private static final Logger LOG = LogManager.getLogger(ElasticSearchConfig.class);

    @Bean
    public TransportClient client(){
        TransportClient client = null;
        try {

            client = new PreBuiltTransportClient(Settings.EMPTY)
                    .addTransportAddress(new InetSocketTransportAddress(InetAddress.getByName(host), port));
        } catch (UnknownHostException e) {
            LOG.error("创建elasticsearch客户端失败");
            e.printStackTrace();
        }
        LOG.info("创建elasticsearch客户端成功");
        return client;
    }

}

配置完毕之后，我们就可以使用了。这里写了一个简单的demo，汇总了常用的一些API使用。
```

#### 代码示例
> 这个代码示例满足了查询所需，查询条件，分组计算，分组排序等

```
    Map<String,Object> map = Collections.emptyMap();

        Script script = new Script(ScriptType.INLINE, "painless","params._value0 > 0",map);  //提前定义好查询销量是否大于1000的脚本，类似SQL里面的having

        long beginTime = System.currentTimeMillis();

        SearchResponse sr = client.prepareSearch("adele").setTypes("sale")
                .setQuery(QueryBuilders.boolQuery()
                        .must(QueryBuilders.termQuery("store_name.keyword", "xxx旗舰店"))  //挨个设置查询条件，没有就不加，如果是字符串类型的，要加keyword后缀
                        .must(QueryBuilders.termQuery("department_name.keyword", "玎"))
                        .must(QueryBuilders.termQuery("category_name.keyword", "T恤"))
                        .must(QueryBuilders.rangeQuery("pay_date").gt("2017-03-07").lt("2017-07-09"))
                ).addAggregation(
                        AggregationBuilders.terms("by_product_code").field("product_code.keyword").size(500) //按货号分组，最多查500个货号.SKU直接改字段名字就可以
                                .subAggregation(AggregationBuilders.terms("by_store_name").field("store_name.keyword").size(50) //按店铺分组，不显示店铺可以过滤掉这一行，下边相应减少一个循环
                                        .subAggregation(AggregationBuilders.sum("total_sales").field("quantity"))  //分组计算销量汇总
                                        .subAggregation(AggregationBuilders.sum("total_sales_amount").field("amount_actual"))  //分组计算实付款汇总，需要加其他汇总的在这里依次加
                                        .subAggregation(PipelineAggregatorBuilders.bucketSelector("sales_bucket_filter",script,"total_sales")))//查询是否大于指定值
                                .order(Terms.Order.compound(Terms.Order.aggregation("total_calculate_sale_amount",false)))) //分组排序

                .execute().actionGet();

        Terms terms = sr.getAggregations().get("by_product_code");   //查询遍历第一个根据货号分组的aggregation

        System.out.println(terms.getBuckets().size());
        for (Terms.Bucket entry : terms.getBuckets()) {
            System.out.println("------------------");
            System.out.println("【 " + entry.getKey() + " 】订单数 : " + entry.getDocCount() );

            Terms subTerms = entry.getAggregations().get("by_store_name");    //查询遍历第二个根据店铺分组的aggregation
            for (Terms.Bucket subEntry : subTerms.getBuckets()) {
                Sum sum1 = subEntry.getAggregations().get("total_sales"); //取得销量的汇总
                double total_sales = sum1.getValue();
                System.out.println(subEntry.getKey() + " 订单数:  " + subEntry.getDocCount() + "  销量: " + total_sales); //店铺和订单数量和销量
            }
        }

        long endTime = System.currentTimeMillis();
        System.out.println("查询耗时" + ( endTime - beginTime ) + "毫秒");
    
```

#### demo地址

[springboot-elasticsearch](https://github.com/whiney/springboot-elasticsearch.git)

1. 启动项目
2. 访问http://localhost:9999/test
3. 查看后端打印

#### 参考链接
[Search API](https://www.elastic.co/guide/en/elasticsearch/client/java-api/current/java-search.html)

有一些坑是我领导踩得，部分代码已得授权。
