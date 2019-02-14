### Elasticsearch  Mapping parameters章节fields翻译

主要在用到一个字段需要多种字段类型字段存在， 因此需要设置此字段。因此翻译官方文档，用于加深理解


我们在很多情况下， 针对同一个字段，有不同的应用场景。 比如， `string`字符串可以用作`text`字段，用于全文本检索， 也可以作为`keyword`字段用于排序或聚合

```
PUT my_index
{
  "mappings": {
    "_doc": {
      "properties": {
        "city": {
          "type": "text",
          "fields": {
            "raw": {  ---------------(1)
              "type":  "keyword"
            }
          }
        }
      }
    }
  }
}

PUT my_index/_doc/1
{
  "city": "New York"
}

PUT my_index/_doc/2
{
  "city": "York"
}

GET my_index/_search
{
  "query": {
    "match": {
      "city": "york"  ----------(2)
    }
  },
  "sort": {
    "city.raw": "asc" ----------(3)
  },
  "aggs": {
    "Cities": {
      "terms": {
        "field": "city.raw" -------(4)
      }
    }
  }
}
```

(1) `city.raw`字段作为`city`字段的 `keyword`版本

(2) `city`字段可以被用来全文检索

(3)(4) `city.raw`字段也可以用来排序和聚合


> 需要注意： 多字段不会改变原始`_source`字段
 
> 提示： fields属性被允许在同一个字段中 有多个字段设置， 多字段可以通过put操作， 修改已经存在的字段

### 多字段与多解析器

多字段的另一个使用场景是 使用不同的方式，解析同一个字段，以获取更好的相关性。

比如， 我们可以索引一个字段 使用 `standard analyzer`解析器(解析为文本为单词)
,并且还可以使用`english analyzer`解析器 将单词分割为根词


```
PUT my_index
{
  "mappings": {
    "_doc": {
      "properties": {
        "text": { -----------(1)
          "type": "text",
          "fields": {
            "english": { --------(2)
              "type":     "text",
              "analyzer": "english"
            }
          }
        }
      }
    }
  }
}

PUT my_index/_doc/1
{ "text": "quick brown fox" } ----------(3)

PUT my_index/_doc/2
{ "text": "quick brown foxes" } --------(4)

GET my_index/_search
{
  "query": {
    "multi_match": {
      "query": "quick brown foxes",
      "fields": [ ------------(5)
        "text",
        "text.english"
      ],
      "type": "most_fields"  --------(6)
    }
  }
}
```
(1) `text`字段使用`standard`解析器 

(2) `text.english`字段使用`english`解析器

(3)(4) 两个文档， 一个是`fox` 另一个是`foxes`

(5)(6) 查询使用`text`和`text.english`字段，并且组合分数

在text字段中， 第一个文档包含fox橘子， 第二个文档包含foxes, `text.english`字段在两个文档中都包含`fox`， 因为`foxes`源于 `fox`

对于`text`字段，查询字符串 被 `standard`解析器 解析， 对于`text.english`字段， 被`english`解析器解析，  源字段允许对foxes的查询 可以匹配到包含 `fox`的文档， 尽可能的允许我们匹配更多的文档， 查询`text字段， 我们提升了文档匹配`foxes`的相关性分数。

### 总结

1. 一个字段可以设置多个子字段，主要用于多种场景， 比如text, keyword

2. wildcard 查询需要`not analyzed` 因此如果需要提高准确率， 需要使用keyword模式
3. 多字段不会改变原始_source字段
4. 可以put操作多字段



原始文档地址： https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-wildcard-query.html






