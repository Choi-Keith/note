

ES支持的全文索引API主要有：
- match: 匹配查询可以处理全文吧、精确字段(日期，数字等)
- match_phrase: 短语匹配所检索的内容分词，必须全部出现在被检索的内容中，并且顺序一致
- match_phrase_prefix: 与match_phrase类似，但是会将需要检索的最有一个词作为前缀，并且匹配这个词项开头的任何词语
- multi match: 可以在多个字段执行相同的查询语句


# 初始化索引books
```
DELETE books

PUT books
{
  "mappings": {
    "properties": {
        "book_id": {
          "type": "keyword"
        },
        "name": {
          "type": "text",
          "analyzer": "standard"
        },
        "author": {
          "type": "keyword"
        },
        "intro": {
          "type": "text"
        },
        "price": {
          "type": "double"
        },
        "date": {
          "type": "date"
        }
      }
  },
  "settings": {
    "number_of_shards": 2,
    "number_of_replicas": 1
  }
}

PUT books/_doc/1
{
  "book_id": "4ee82462",
  "name": "Dive into the Linux kernel architecture",
  "author": "Wolfgang Mauerer",
  "intro": "The content is comprehensive and in-depth, appreciate the infinite scenery of the Linux kernel.",
  "price": 19.9,
  "date": "2010-06-01"
}

PUT books/_doc/2
{
  "book_id": "4ee82463",
  "name": "A Brief History Of Time",
  "author": "Stephen Hawking",
  "intro": "A fascinating story that explores the secrets at the heart of time and space.",
  "price": 9.9,
  "date": "1988-01-01"
}

PUT books/_doc/3
{
  "book_id": "4ee82464",
  "name": "Beginning Linux Programming 4th Edition",
  "author": "Neil Matthew、Richard Stones",
  "intro": "Describes the Linux system and other UNIX-style operating system on the program development",
  "price": 12.9,
  "date": "2010-06-01"
}

GET /books/_search
```

# 匹配索引books所有文档
POST books/_search
{
  "query": {
    "match_all": {}
  }
}


## match
```
POST books/_search
{
  "query": {
    "match": {
      "name": "linux architecture"
    }
  }
}
```

## match_phrase

```
POST books/_search
{
  "query": {
    "match_phrase": {
      "name": {
        "query": "linux architecture",
        "slop": 1
      }
    }
  }
}
```
## match_phrase_prefix
```
POST books/_search
{
  "query": {
    "match_phrase_prefix": {
      "name": {
        "query": "linux kerne",
        "max_expansions": 2
      }
      
    }
  }
}
```


## multi match

```
GET /books/_search
{
  "query": {
    "multi_match": {
      "query": "linux architecture",
      "fields": ["nam*", "intro^2"]
    }
  }
}



```


















































































































