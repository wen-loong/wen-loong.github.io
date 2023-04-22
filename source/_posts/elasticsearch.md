---
title: elasticsearch
comments: true
aplayer: false
date: 2023-04-17 21:55:16
tags:
 - Elasticsearch
 - 全文搜索引擎
categories:
keywords:
description:
top_img:
cover:
sticky:
---
## 一、ES查询
---
 ES查询分为两种，一种是查询（query），另一种是过滤（filter）。
 - query默认会计算返回文档的得分，然后根据得分进行排序
 - filter只会筛选出符合条件的文档，不会计算得分，查询的结果可以被缓存。
  > 单从性能的角度来看，过滤优于查询。如果只是大范围筛选数据不考虑相关度得分，使用filter，如果对排序结果又要求，需要将匹配的文档排在最前面，使用query

### 1、查询（query）的使用
查询包括 `match`、`match_phrase`、`match_phrase_prefix`、`multi_match `、`range `、`term `、`terms `、`exists`、`missing`
|查询关键字|描述|
|  ----  | ----  |
|match|会将搜索词（输入条件）做分词，然后与目标字段做匹配，若分词中任意一个词与目标字段匹配上，那么这个文档满足条件|
|match_phrase|精确匹配查询的短语，需要全部单词和顺序要完全一样，标点符号除外|
|match_phrase_prefix|和 match_phrase 用法是一样的，区别就在于它允许对最后一个词条前缀匹配|
|multi_match |可以在多个字段上执行相同的 match 查询|
|range |出那些落在指定区间内的数字或者时间|
|term|精确值匹配，这些精确值可能是数字、时间、布尔或者 not_analyzed 的字符串|
|terms|和 term 查询一样，区别在于可以传入多个值。如果目标字段包含了指定值中的任何一个值，那么这个文档满足条件|
|exists |用于查找指定字段中有值的文档|
|missing|用于查找指定字段中没有值的文档(ES 5之后版本已经移除)|

+ `match`查询使用
`GET test-index/_search`
```json
{
  "query": {
    "match": {
      "name": "zhanghao"
    }
  }
}
```
+ `match_phrase` 使用
`GET test-index/_search`
```json
{
  "query": {
    "match_phrase": {
      "name": "zhanghao"
    }
  }
}
```
+ `match_phrase_prefix`使用
```json
{
  "query": {
    "match_phrase_prefix": {
      "name": "zhanghao"
    }
  }
}
```

+ `multi_match`查询
```json
{
  "query": {
    "multi_match": {
      "query": "zhangsan",
      "fields": ["name", "description"]
    }
  }
}
```

+ `range`查询

```json
{
  "query": {
    "range": {
      "age": {
        "gte": 10,
        "lte": 20
      }
    }
  }
}
```

+ `term` 查询
```json
{
  "query": {
    "term": {
      "name": {
        "value": "lisi"
      }
    }
  }
}
```

+ `terms` 查询
```json
{
  "query": {
    "terms": {
      "name": [
        "zhangsan",
        "lisi"
      ]
    }
  }
}
```

+ `exists` 查询
```json
{
  "query": {
    "exists": {
      "field": "name"
    }
  }
}
```

+ `missing`查询（ES 5之前版本支持）
```json
{
  "query": {
    "bool": {
      "filter": [
        {
          "missing": {
            "field": "desc"
          }
        }
      ]
    }
  }
}
```


### 2、过滤（filter）的使用
+ filter 和 `term` 使用
`GET test-index/_search`
```json
{
  "query": {
    "bool": {
      "filter": {
        "term": {
          "name":"zhanghao"
        }
      }
    }
  }
}
```
+ filter 和 `terms` 使用
`GET test-index/_search`
```json
GET test-index/_search
{
  "query": {
    "bool": {
      "filter": [
        {
          "terms": {
            "name": [
              "zhanghao",
              "lisi"
            ]
          }
        }
      ]
    }
  }
}
```
+ filter和bool使用
`GET test-index/_search`
```json 
{
  "query": {
    "bool": {
      "filter": {
        "bool": {
          "must": {
            "match": {
              "name": "zhanghao"
            }
          }
        }
      }
    }
  }
}
```
+ 多个过滤条件
```json
{
  "query": {
    "bool": {
      "filter": [
        {
          "term": {
            "name": "zhangsan"
          }
        },
        {
          "term": {
            "age": "18"
          }
        }
      ]
    }
  }
}
```

> 还有ids、ranage、exists 等过滤器，使用方法基本一致。

