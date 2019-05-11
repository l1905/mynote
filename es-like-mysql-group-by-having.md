## ES实现类似MYSQL GROUP BY HAVING 操作


## 背景

业务中遇到如下两个SQL需要使用ES完成搜索

```
#sql-01
SELECT user_id FROM `comment_data` WHERE user_id>? AND `status`=1 AND creation_date>=? AND creation_date<=?  GROUP BY user_id HAVING count(1)>=? ORDER BY user_id LIMIT ?

#sql-02
SELECT user_id FROM comment_data WHERE user_id>? and `status`=1 GROUP BY user_id HAVING max(creation_date)>=? AND max(creation_date)<=? ORDER BY user_id LIMIT ?
```

我们使用伪代码进行描述`sql-01`：

1. 筛选条件，过滤符合条件的数据，即 `where`部分
2. 指定field 进行分桶， 即`group by user_id`
3. 根据`count`总数来筛选符合条件的桶， 即 `having count(1) >= ?`
4. 对桶进行排序， 即`ORDER BY user_id`
5. 限制返回桶的个数， 即 `limit ?`部分


## ES实现

已经拆解为子问题， 我们去ES中寻找对应的解决方案。



### 步骤1. 使用query完成条件过滤

对应PHP代码实现为：

```
$search = new \ONGR\ElasticsearchDSL\Search();

$boolQuery = new \ONGR\ElasticsearchDSL\Query\Compound\BoolQuery();
//设置过滤条件
$boolQuery->add(
    new \ONGR\ElasticsearchDSL\Query\TermLevel\RangeQuery("creation_date", [\ONGR\ElasticsearchDSL\Query\TermLevel\RangeQuery::GTE => '2019-04-20 00:00:00']),
    \ONGR\ElasticsearchDSL\Query\Compound\BoolQuery::FILTER
);
$search->addQuery($boolQuery);
```

### 步骤2: 指定field分桶进行聚合操作

我们选择使用 `Terms Aggregation`桶，指定`field`为`user_id`

对应PHP实现为

```
#常规分桶
$aggregation_term = new \ONGR\ElasticsearchDSL\Aggregation\Bucketing\TermsAggregation('group_by_user_id');
$aggregation_term->setField("user_id");
```



### 步骤3: 筛选桶

#### 方案01

找到`Bucket Selector Aggregation`聚合， 属于`Pipeline Aggregations`类型， 即输入为桶， 可以过滤筛选出我们希望的桶数据。

`优点`：

支持复杂的过滤桶筛选

`缺点`:

父集合中如果指定`size`参数后，筛选出的桶结构为其输入， 会影响正常过滤桶。 即步骤5在步骤3之前执行了。
并且其自身不支持`限制返回桶的个数`参数。 目前ES版本是5.4

对应的PHP实现分方式为：

```
$agg_bucket_selector = new \ONGR\ElasticsearchDSL\Aggregation\Pipeline\BucketSelectorAggregation('selector_count', ['orderCount' => '_count']);

$agg_bucket_selector->setScript("params.orderCount >=40 && params.orderCount <=451");

$aggregation_term->addAggregation($agg_bucket_selector);

```

#### 方案02

我们发现 `Terms Aggregation`聚合实现中，支持`min_doc_count`, 选择最小匹配文档总数， 正好契合我们`having count(1) >1` 操作。但是不支持`max_doc_count`操作，无法完成`having count(1) < 3`类似操作

对应的PHP实现为

```
$aggregation_term->addParameter('min_doc_count', 2);

```

### 步骤4: 对桶进行排序

`Terms Aggregation` 支持对桶排序， 对应PHP实现为

```
$aggregation_term->addParameter('order', ['_term' => 'asc']);
```

### 步骤5: 对返回桶个数进行限制

```
$aggregation_term->addParameter('size', 10); #必须设置， 如果不设置，则默认为10，
```

筛选桶如果选方案1， 则执行顺序为

步骤1---->步骤2 ---->步骤4 ---->步骤5 ----->步骤3(方案1)

我们的PHP完整实现为

```
$search = new \ONGR\ElasticsearchDSL\Search();

$boolQuery = new \ONGR\ElasticsearchDSL\Query\Compound\BoolQuery();
//设置过滤条件
$boolQuery->add(
    new \ONGR\ElasticsearchDSL\Query\TermLevel\RangeQuery("creation_date", [\ONGR\ElasticsearchDSL\Query\TermLevel\RangeQuery::GTE => '2019-04-20 00:00:00']),
    \ONGR\ElasticsearchDSL\Query\Compound\BoolQuery::FILTER
);
$search->addQuery($boolQuery);

//常规分桶
$aggregation_term = new \ONGR\ElasticsearchDSL\Aggregation\Bucketing\TermsAggregation('group_by_user_id');
$aggregation_term->setField("user_info.user_id");

//选择使用bucketSelector聚合方式(这里需要注意，这里是指定的桶路径)
$agg_bucket_selector = new \ONGR\ElasticsearchDSL\Aggregation\Pipeline\BucketSelectorAggregation('selector_count', ['orderCount' => '_count']);
$agg_bucket_selector->setScript("params.orderCount >=2");
$aggregation_term->addAggregation($agg_bucket_selector);

//设置排序
$aggregation_term->addParameter('order', ['_count' => 'desc']);

//设置返回总数
$aggregation_term->addParameter('size', 10);

//设置过滤条件
$search->addAggregation($aggregation_term);

// 请求参数
$params = [
    'index' => 'comment_bgm',
    'type' => 'doc',
    'from' => 0,
    'size' => 0,
];

$params['body'] = $search->toArray();

//        exit(json_encode($params, JSON_PRETTY_PRINT));


$result = $this->zfilter->search($params);

exit(json_encode($result, JSON_PRETTY_PRINT));

```

打印返回ES的json格式为：

```
{
    "index": "comment_bgm",
    "type": "doc",
    "from": 0,
    "size": 0,
    "body": {
        "query": {
            "bool": {
                "filter": [
                    {
                        "range": {
                            "creation_date": {
                                "gte": "2019-04-20 00:00:00"
                            }
                        }
                    }
                ]
            }
        },
        "aggregations": {
            "group_by_user_id": {
                "terms": {
                    "field": "user_info.user_id",
                    "order": {
                        "_count": "desc"
                    },
                    "size": 10
                },
                "aggregations": {
                    "selector_count": {
                        "bucket_selector": {
                            "buckets_path": {
                                "orderCount": "_count"
                            },
                            "script": "params.orderCount >=2"
                        }
                    }
                }
            }
        }
    }
}
```

筛选桶如果选方案2， 则执行顺序为:

步骤1 --->步骤2----> 步骤3(方案2)----->步骤4 ----->步骤5

PHP完整实现为

```
$search = new \ONGR\ElasticsearchDSL\Search();

$boolQuery = new \ONGR\ElasticsearchDSL\Query\Compound\BoolQuery();
//设置过滤条件
$boolQuery->add(
    new \ONGR\ElasticsearchDSL\Query\TermLevel\RangeQuery("creation_date", [\ONGR\ElasticsearchDSL\Query\TermLevel\RangeQuery::GTE => '2019-04-20 00:00:00']),
    \ONGR\ElasticsearchDSL\Query\Compound\BoolQuery::FILTER
);
$search->addQuery($boolQuery);

//常规分桶
$aggregation_term = new \ONGR\ElasticsearchDSL\Aggregation\Bucketing\TermsAggregation('group_by_user_id');
$aggregation_term->setField("user_info.user_id");

//设置min_doc_count
$aggregation_term->addParameter('min_doc_count', 2);
//设置size
$aggregation_term->addParameter('size', 5);
//设置排序
$aggregation_term->addParameter('order', ['_count' => 'desc']);

//添加到聚合
$search->addAggregation($aggregation_term);

// 请求参数
$params = [
    'index' => 'comment_bgm',
    'type' => 'doc',
    'from' => 0,
    'size' => 0,
];

$params['body'] = $search->toArray();

//        exit(json_encode($params, JSON_PRETTY_PRINT).PHP_EOL);


$result = $this->zfilter->search($params);

exit(json_encode($result, JSON_PRETTY_PRINT));

```

对应返回的ES-json格式为

```
{
    "index": "comment_bgm",
    "type": "doc",
    "from": 0,
    "size": 0,
    "body": {
        "query": {
            "bool": {
                "filter": [
                    {
                        "range": {
                            "creation_date": {
                                "gte": "2019-04-20 00:00:00"
                            }
                        }
                    }
                ]
            }
        },
        "aggregations": {
            "group_by_user_id": {
                "terms": {
                    "field": "user_info.user_id",
                    "min_doc_count": 2,
                    "size": 5,
                    "order": {
                        "_count": "desc"
                    }
                }
            }
        }
    }
}
```

## 其他

针对sql-02， 因为having条件不是count, 所以我们没办法用筛选桶方案02， 因此我们只能选择使用 `bucket selector`来实现。 对应的PHP实现为

```
public function user_id_bucket_by_max_creation_date($param = [])
{
    $default_param = [
        'agg_max_creation_date_start' => date('Y-m-d H:i:s'),
        'agg_max_creation_date_end' => date('Y-m-d H:i:s'),
    ];
    $param = array_merge($default_param, $param);

    //东8区比标准时间早8小时
    $param['agg_max_creation_date_start'] = (strtotime($param['agg_max_creation_date_start']) + 28800) * 1000;
    $param['agg_max_creation_date_end'] = (strtotime($param['agg_max_creation_date_end']) + 28800) * 1000;
//        exit(json_encode($param));

    $agg_params = [
        'order' => ['_term' => 'asc'], //限制order顺序 //todo 排序有问题，需要着重查看 先排序， 后limit
    ];

    #常规分桶
    $aggregation_term = new \ONGR\ElasticsearchDSL\Aggregation\Bucketing\TermsAggregation('group_by_user_id');
    $aggregation_term->setField("user_info.user_id");

    //添加max最大桶
    $aggregation_max = new \ONGR\ElasticsearchDSL\Aggregation\Metric\MaxAggregation('max_creation_date');
    $aggregation_max->setField('creation_date');
    $aggregation_term->addAggregation($aggregation_max);

    //1557124134755

    //设置bucket_selector桶
    $agg_bucket_selector = new \ONGR\ElasticsearchDSL\Aggregation\Pipeline\BucketSelectorAggregation('selector_max_creation_date', ['max_creation_date' => 'max_creation_date']);
    $agg_bucket_selector->setScript("params.max_creation_date >= {$param['agg_max_creation_date_start']}L && params.max_creation_date <= {$param['agg_max_creation_date_end']}L ");


    #设置排序规则
    foreach ($agg_params as $key => $param) {
        $aggregation_term->addParameter($key, $param);
    }

    $aggregation_term->addAggregation($agg_bucket_selector);

    return $aggregation_term;

}
```

对应的es-json格式为：

```
{
    "index": "comment_bgm",
    "type": "doc",
    "from": "0",
    "size": "0",
    "body": {
        "query": {
            "bool": {
                "filter": [
                    {
                        "range": {
                            "user_info.user_id": {
                                "gte": 25855
                            }
                        }
                    },
                    {
                        "range": {
                            "creation_date": {
                                "gte": "2019-04-20 00:00:00"
                            }
                        }
                    }
                ]
            }
        },
        "sort": [
            {
                "creation_date": {
                    "order": "desc"
                }
            }
        ],
        "aggregations": {
            "group_by_user_id": {
                "terms": {
                    "field": "user_info.user_id",
                    "order": {
                        "_term": "asc"
                    }
                },
                "aggregations": {
                    "max_creation_date": {
                        "max": {
                            "field": "creation_date"
                        }
                    },
                    "selector_max_creation_date": {
                        "bucket_selector": {
                            "buckets_path": {
                                "max_creation_date": "max_creation_date"
                            },
                            "script": "params.max_creation_date >= 1556536369000L && params.max_creation_date <= 1557069590000L "
                        }
                    }
                }
            }
        }
    }
}
```

这里有个明显的坑， ES中存储是使用UTC格式，需要将我们当前时间 加上8小时(28800秒)转化为UTC时间进行比较。 这里需要使用`Painless`的long类型


## 高版本ES实现

从6.1开始，支持 `Bucket Sort Aggregation`聚合， 即支持桶筛选后， 对指定桶进行排序操作。 即我们筛选桶方案1， 然后再用`bucket-sort`进行排序限制（未实践）

## 参考文章

1. https://www.elastic.co/guide/en/elasticsearch/reference/current/search-aggregations-bucket-terms-aggregation.html

2. https://www.elastic.co/guide/en/elasticsearch/reference/current/search-aggregations-pipeline-bucket-sort-aggregation.html

3. https://elasticsearch.cn/article/629

4. https://stackoverflow.com/questions/46908360/sql-like-group-by-and-having
5. http://cwiki.apachecn.org/pages/viewpage.action?pageId=10030397






















 




