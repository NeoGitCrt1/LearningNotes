###一些基础的商品搜索功能设计与实现

参考
https://juejin.cn/post/6844904126321524744
https://mp.weixin.qq.com/s/cohWZy_eUOUqbmUxhXzzNA
https://www.elastic.co/guide/cn/elasticsearch/guide/current/function-score-query.html
[docs.spring.io/spring-data…](https://docs.spring.io/spring-data/elasticsearch/docs/3.2.6.RELEASE/reference/html/#reference)
[github.com/macrozheng/…](https://github.com/macrozheng/mall)


1. 安装中文分词器
```
elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v6.4.0/elasticsearch-analysis-ik-6.4.0.zip
```
测试分词
```
GET /这里是index名/_analyze
{
  "text": "小米手机性价比很高",
  "tokenizer": "ik_max_word"
}
```

2. 在SpringBoot中定义索引
```
/**
 * 搜索中的商品信息
 * Created by macro on 2018/6/19.
 */
@Document(indexName = "pms", type = "product",shards = 1,replicas = 0)
public class EsProduct implements Serializable {
    private static final long serialVersionUID = -1L;
    @Id
    private Long id;
    @Field(analyzer = "ik_max_word",type = FieldType.Text)
    private String name;
    @Field(analyzer = "ik_max_word",type = FieldType.Text)
    private String subTitle;
    @Field(analyzer = "ik_max_word",type = FieldType.Text)
    private String keywords;
    //后略
}
```

3. 根据关键字搜索
- Query DSL
```
POST /pms/product/_search
{
  "from": 0, 
  "size": 2, 
  "query": {
    "multi_match": {
      "query": "小米",
      "fields": [
        "name",
        "subTitle",
        "keywords"
      ]
    }
  }
}
```
- SpringBoot Elasticsearch Repositories
```
/**
 * 商品搜索管理Service实现类
 * Created by macro on 2018/6/19.
 */
@Service
public class EsProductServiceImpl implements EsProductService {
    @Override
    public Page<EsProduct> search(String searchWord, Integer pageNum, Integer pageSize) {
        Pageable pageable = PageRequest.of(pageNum, pageSize);
        return productRepository.findByNameOrSubTitleOrKeywords(searchWord, searchWord, searchWord, pageable);
    }
}
```
- 转化规则
![1.png](https://upload-images.jianshu.io/upload_images/26104743-3431bad5c20009ac.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

3. 过滤，匹配权重，排序
- 需求
![2.png](https://upload-images.jianshu.io/upload_images/26104743-d69c20fd9371aeb6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- 特殊需求
匹配重要度   商品名称  > 副标题 > 关键字
> [`function_score` 查询](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/query-dsl-function-score-query.html) 是用来控制评分过程的终极武器，它允许为每个与主查询匹配的文档应用一个函数，以达到改变甚至完全替换原始查询评分 `_score` 的目的。
实际上，也能用过滤器对结果的 *子集* 应用不同的函数，这样一箭双雕：既能高效评分，又能利用过滤器缓存。

> `function_score` 预定义函数
weight  :  为每个文档应用一个简单而不被规范化的权重提升值：当 weight 为 2 时，最终结果为 2 * _score 。
 field_value_factor  :  使用这个值来修改 _score ，如将 popularity 或 votes （受欢迎或赞）作为考虑因素。
 random_score  :  为每个用户都使用一个不同的随机评分对结果排序，但对某一具体用户来说，看到的顺序始终是一致的。
linear 、 exp 、 gauss  :  衰减函数将浮动值结合到评分 _score 中，例如结合 publish_date 获得最近发布的文档，结合 geo_location 获得更接近某个具体经纬度（lat/lon）地点的文档，结合 price 获得更接近某个特定价格的文档。
 script_score  :  如果需求超出以上范围时，用自定义脚本可以完全控制评分计算，实现所需逻辑。

- Query DSL
```
POST /pms/product/_search
{
  "query": {
    "function_score": {
      "query": {
        "bool": {
          "must": [
            {
              "match_all": {}
            }
          ],
          "filter": {
            "bool": {
              "must": [
                {
                  "term": {
                    "brandId": 6
                  }
                },
                {
                  "term": {
                    "productCategoryId": 19
                  }
                }
              ]
            }
          }
        }
      },
      "functions": [
        {
          "filter": {
            "match": {
              "name": "小米"
            }
          },
          "weight": 10
        },
        {
          "filter": {
            "match": {
              "subTitle": "小米"
            }
          },
          "weight": 5
        },
        {
          "filter": {
            "match": {
              "keywords": "小米"
            }
          },
          "weight": 2
        }
      ],
      "score_mode": "sum",
      "min_score": 2
    }
  },
  "sort": [
    {
      "_score": {
        "order": "desc"
      }
    }
  ]
}
```
- SpringBoot Elasticsearch Repositories
```
/**
 * 商品搜索管理Service实现类
 * Created by macro on 2018/6/19.
 */
@Service
public class EsProductServiceImpl implements EsProductService {
    @Override
    public Page<EsProduct> search(String keyword, Long brandId, Long productCategoryId, Integer pageNum, Integer pageSize,Integer sort) {
        Pageable pageable = PageRequest.of(pageNum, pageSize);
        NativeSearchQueryBuilder nativeSearchQueryBuilder = new NativeSearchQueryBuilder();
        //分页
        nativeSearchQueryBuilder.withPageable(pageable);
        //过滤
        if (brandId != null || productCategoryId != null) {
            BoolQueryBuilder boolQueryBuilder = QueryBuilders.boolQuery();
            if (brandId != null) {
                boolQueryBuilder.must(QueryBuilders.termQuery("brandId", brandId));
            }
            if (productCategoryId != null) {
                boolQueryBuilder.must(QueryBuilders.termQuery("productCategoryId", productCategoryId));
            }
            nativeSearchQueryBuilder.withFilter(boolQueryBuilder);
        }
        //搜索
        if (StringUtils.isEmpty(keyword)) {
            nativeSearchQueryBuilder.withQuery(QueryBuilders.matchAllQuery());
        } else {
            List<FunctionScoreQueryBuilder.FilterFunctionBuilder> filterFunctionBuilders = new ArrayList<>();
            filterFunctionBuilders.add(new FunctionScoreQueryBuilder.FilterFunctionBuilder(QueryBuilders.matchQuery("name", keyword),
                    ScoreFunctionBuilders.weightFactorFunction(10)));
            filterFunctionBuilders.add(new FunctionScoreQueryBuilder.FilterFunctionBuilder(QueryBuilders.matchQuery("subTitle", keyword),
                    ScoreFunctionBuilders.weightFactorFunction(5)));
            filterFunctionBuilders.add(new FunctionScoreQueryBuilder.FilterFunctionBuilder(QueryBuilders.matchQuery("keywords", keyword),
                    ScoreFunctionBuilders.weightFactorFunction(2)));
            FunctionScoreQueryBuilder.FilterFunctionBuilder[] builders = new FunctionScoreQueryBuilder.FilterFunctionBuilder[filterFunctionBuilders.size()];
            filterFunctionBuilders.toArray(builders);
            FunctionScoreQueryBuilder functionScoreQueryBuilder = QueryBuilders.functionScoreQuery(builders)
                    .scoreMode(FunctionScoreQuery.ScoreMode.SUM)
                    .setMinScore(2);
            nativeSearchQueryBuilder.withQuery(functionScoreQueryBuilder);
        }
        //排序
        if(sort==1){
            //按新品从新到旧
            nativeSearchQueryBuilder.withSort(SortBuilders.fieldSort("id").order(SortOrder.DESC));
        }else if(sort==2){
            //按销量从高到低
            nativeSearchQueryBuilder.withSort(SortBuilders.fieldSort("sale").order(SortOrder.DESC));
        }else if(sort==3){
            //按价格从低到高
            nativeSearchQueryBuilder.withSort(SortBuilders.fieldSort("price").order(SortOrder.ASC));
        }else if(sort==4){
            //按价格从高到低
            nativeSearchQueryBuilder.withSort(SortBuilders.fieldSort("price").order(SortOrder.DESC));
        }else{
            //按相关度
            nativeSearchQueryBuilder.withSort(SortBuilders.scoreSort().order(SortOrder.DESC));
        }
        nativeSearchQueryBuilder.withSort(SortBuilders.scoreSort().order(SortOrder.DESC));
        NativeSearchQuery searchQuery = nativeSearchQueryBuilder.build();
        LOGGER.info("DSL:{}", searchQuery.getQuery().toString());
        return productRepository.search(searchQuery);
    }
}
```
4. 同类商品搜索
- 需求
根据ID获取指定商品信息，然后以指定商品的名称、品牌和关键字来搜索商品，并且要过滤掉当前商品，调整搜索条件中的权重以获取最好的匹配度；

- Query DSL
```
POST /pms/product/_search
{
  "query": {
    "function_score": {
      "query": {
        "bool": {
          "must": [
            {
              "match_all": {}
            }
          ],
          "filter": {
            "bool": {
              "must_not": {
                "term": {
                  "id": 28
                }
              }
            }
          }
        }
      },
      "functions": [
        {
          "filter": {
            "match": {
              "name": "红米5A"
            }
          },
          "weight": 8
        },
        {
          "filter": {
            "match": {
              "subTitle": "红米5A"
            }
          },
          "weight": 2
        },
        {
          "filter": {
            "match": {
              "keywords": "红米5A"
            }
          },
          "weight": 2
        },
        {
          "filter": {
            "term": {
              "brandId": 6
            }
          },
          "weight": 5
        },
        {
          "filter": {
            "term": {
              "productCategoryId": 19
            }
          },
          "weight": 3
        }
      ],
      "score_mode": "sum",
      "min_score": 2
    }
  }
}
```
- SpringBoot Elasticsearch Repositories
```
/**
 * 商品搜索管理Service实现类
 * Created by macro on 2018/6/19.
 */
@Service
public class EsProductServiceImpl implements EsProductService {
    @Override
    public Page<EsProduct> recommend(Long id, Integer pageNum, Integer pageSize) {
        Pageable pageable = PageRequest.of(pageNum, pageSize);
        List<EsProduct> esProductList = productDao.getAllEsProductList(id);
        if (esProductList.size() > 0) {
            EsProduct esProduct = esProductList.get(0);
            String keyword = esProduct.getName();
            Long brandId = esProduct.getBrandId();
            Long productCategoryId = esProduct.getProductCategoryId();
            //根据商品标题、品牌、分类进行搜索
            List<FunctionScoreQueryBuilder.FilterFunctionBuilder> filterFunctionBuilders = new ArrayList<>();
            filterFunctionBuilders.add(new FunctionScoreQueryBuilder.FilterFunctionBuilder(QueryBuilders.matchQuery("name", keyword),
                    ScoreFunctionBuilders.weightFactorFunction(8)));
            filterFunctionBuilders.add(new FunctionScoreQueryBuilder.FilterFunctionBuilder(QueryBuilders.matchQuery("subTitle", keyword),
                    ScoreFunctionBuilders.weightFactorFunction(2)));
            filterFunctionBuilders.add(new FunctionScoreQueryBuilder.FilterFunctionBuilder(QueryBuilders.matchQuery("keywords", keyword),
                    ScoreFunctionBuilders.weightFactorFunction(2)));
            filterFunctionBuilders.add(new FunctionScoreQueryBuilder.FilterFunctionBuilder(QueryBuilders.matchQuery("brandId", brandId),
                    ScoreFunctionBuilders.weightFactorFunction(5)));
            filterFunctionBuilders.add(new FunctionScoreQueryBuilder.FilterFunctionBuilder(QueryBuilders.matchQuery("productCategoryId", productCategoryId),
                    ScoreFunctionBuilders.weightFactorFunction(3)));
            FunctionScoreQueryBuilder.FilterFunctionBuilder[] builders = new FunctionScoreQueryBuilder.FilterFunctionBuilder[filterFunctionBuilders.size()];
            filterFunctionBuilders.toArray(builders);
            FunctionScoreQueryBuilder functionScoreQueryBuilder = QueryBuilders.functionScoreQuery(builders)
                    .scoreMode(FunctionScoreQuery.ScoreMode.SUM)
                    .setMinScore(2);
            //用于过滤掉相同的商品
            BoolQueryBuilder boolQueryBuilder = new BoolQueryBuilder();
            boolQueryBuilder.mustNot(QueryBuilders.termQuery("id",id));
            //构建查询条件
            NativeSearchQueryBuilder builder = new NativeSearchQueryBuilder();
            builder.withQuery(functionScoreQueryBuilder);
            builder.withFilter(boolQueryBuilder);
            builder.withPageable(pageable);
            NativeSearchQuery searchQuery = builder.build();
            LOGGER.info("DSL:{}", searchQuery.getQuery().toString());
            return productRepository.search(searchQuery);
        }
        return new PageImpl<>(null);
    }
}
```
5. 聚合搜索
- 需求
根据搜索关键字获取到与关键字匹配商品相关的分类、品牌以及属性，返回出现次数最多的前十
- Query DSL
```
POST /pms/product/_search
{
  "query": {
    "multi_match": {
      "query": "小米",
      "fields": [
        "name",
        "subTitle",
        "keywords"
      ]
    }
  },
  "size": 0,
  "aggs": {
    "brandNames": {
      "terms": {
        "field": "brandName",
        "size": 10
      }
    },
    "productCategoryNames": {
      "terms": {
        "field": "productCategoryName",
        "size": 10
      }
    },
    "allAttrValues": {
      "nested": {
        "path": "attrValueList"
      },
      "aggs": {
        "productAttrs": {
          "filter": {
            "term": {
              "attrValueList.type": 1
            }
          },
          "aggs": {
            "attrIds": {
              "terms": {
                "field": "attrValueList.productAttributeId",
                "size": 10
              },
              "aggs": {
                "attrValues": {
                  "terms": {
                    "field": "attrValueList.value",
                    "size": 10
                  }
                },
                "attrNames": {
                  "terms": {
                    "field": "attrValueList.name",
                    "size": 10
                  }
                }
              }
            }
          }
        }
      }
    }
  }
}
```
- ElasticsearchTemplate
```
/**
 * 商品搜索管理Service实现类
 * Created by macro on 2018/6/19.
 */
@Service
public class EsProductServiceImpl implements EsProductService {
    @Override
    public EsProductRelatedInfo searchRelatedInfo(String keyword) {
        NativeSearchQueryBuilder builder = new NativeSearchQueryBuilder();
        //搜索条件
        if(StringUtils.isEmpty(keyword)){
            builder.withQuery(QueryBuilders.matchAllQuery());
        }else{
            builder.withQuery(QueryBuilders.multiMatchQuery(keyword,"name","subTitle","keywords"));
        }
        //聚合搜索品牌名称
        builder.addAggregation(AggregationBuilders.terms("brandNames").field("brandName"));
        //集合搜索分类名称
        builder.addAggregation(AggregationBuilders.terms("productCategoryNames").field("productCategoryName"));
        //聚合搜索商品属性，去除type=1的属性
        AbstractAggregationBuilder aggregationBuilder = AggregationBuilders.nested("allAttrValues","attrValueList")
                .subAggregation(AggregationBuilders.filter("productAttrs",QueryBuilders.termQuery("attrValueList.type",1))
                .subAggregation(AggregationBuilders.terms("attrIds")
                        .field("attrValueList.productAttributeId")
                        .subAggregation(AggregationBuilders.terms("attrValues")
                                .field("attrValueList.value"))
                        .subAggregation(AggregationBuilders.terms("attrNames")
                                .field("attrValueList.name"))));
        builder.addAggregation(aggregationBuilder);
        NativeSearchQuery searchQuery = builder.build();
        return elasticsearchTemplate.query(searchQuery, response -> {
            LOGGER.info("DSL:{}",searchQuery.getQuery().toString());
            return convertProductRelatedInfo(response);
        });
    }

    /**
     * 将返回结果转换为对象
     */
    private EsProductRelatedInfo convertProductRelatedInfo(SearchResponse response) {
        EsProductRelatedInfo productRelatedInfo = new EsProductRelatedInfo();
        Map<String, Aggregation> aggregationMap = response.getAggregations().getAsMap();
        //设置品牌
        Aggregation brandNames = aggregationMap.get("brandNames");
        List<String> brandNameList = new ArrayList<>();
        for(int i = 0; i<((Terms) brandNames).getBuckets().size(); i++){
            brandNameList.add(((Terms) brandNames).getBuckets().get(i).getKeyAsString());
        }
        productRelatedInfo.setBrandNames(brandNameList);
        //设置分类
        Aggregation productCategoryNames = aggregationMap.get("productCategoryNames");
        List<String> productCategoryNameList = new ArrayList<>();
        for(int i=0;i<((Terms) productCategoryNames).getBuckets().size();i++){
            productCategoryNameList.add(((Terms) productCategoryNames).getBuckets().get(i).getKeyAsString());
        }
        productRelatedInfo.setProductCategoryNames(productCategoryNameList);
        //设置参数
        Aggregation productAttrs = aggregationMap.get("allAttrValues");
        List<LongTerms.Bucket> attrIds = ((LongTerms) ((InternalFilter) ((InternalNested) productAttrs).getProperty("productAttrs")).getProperty("attrIds")).getBuckets();
        List<EsProductRelatedInfo.ProductAttr> attrList = new ArrayList<>();
        for (Terms.Bucket attrId : attrIds) {
            EsProductRelatedInfo.ProductAttr attr = new EsProductRelatedInfo.ProductAttr();
            attr.setAttrId((Long) attrId.getKey());
            List<String> attrValueList = new ArrayList<>();
            List<StringTerms.Bucket> attrValues = ((StringTerms) attrId.getAggregations().get("attrValues")).getBuckets();
            List<StringTerms.Bucket> attrNames = ((StringTerms) attrId.getAggregations().get("attrNames")).getBuckets();
            for (Terms.Bucket attrValue : attrValues) {
                attrValueList.add(attrValue.getKeyAsString());
            }
            attr.setAttrValues(attrValueList);
            if(!CollectionUtils.isEmpty(attrNames)){
                String attrName = attrNames.get(0).getKeyAsString();
                attr.setAttrName(attrName);
            }
            attrList.add(attr);
        }
        productRelatedInfo.setProductAttrs(attrList);
        return productRelatedInfo;
    }
}
```