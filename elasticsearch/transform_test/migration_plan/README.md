# mapping
```bash
curl -XPUT http://localhost:9200/customer -H 'Content-Type: application/json' -d '
{
    "mappings": {
        "properties": {
            "customer_id": {
                "type": "keyword"
            },
            "group_code": {
                "type": "keyword"
            },
            "phone_number_list": {
                "type": "nested",
                "properties": {
                    "phone_number": {
                        "type": "text",
                        "fields": {
                            "suggest": {
                                "type": "text",
                                "analyzer": "standard",
                                "search_analyzer": "standard"
                            },
                            "raw": {
                                "type": "keyword"
                            }
                        }
                    }
                }
            }
        }
    }
}
'
```
```bash
curl -XPUT http://localhost:9200/customer-1 -H 'Content-Type: application/json' -d '
{
    "mappings": {
        "properties": {
            "customer_id": {
                "type": "keyword"
            },
            "group_code": {
                "type": "keyword"
            },
            "phone_number_list": {
                "type": "nested",
                "properties": {
                    "phone_number": {
                        "type": "text",
                        "fields": {
                            "suggest": {
                                "type": "text",
                                "analyzer": "standard",
                                "search_analyzer": "standard"
                            },
                            "raw": {
                                "type": "keyword"
                            }
                        }
                    }
                }
            },
            "phone_numbers": {
                "type": "keyword"
            }
        }
    }
}
'
```

# ingest pipeline
## timestampを設定することで、同期できる。
```bash
curl -XPUT 'http://localhost:9200/_ingest/pipeline/add_timestamp' -H 'Content-Type: application/json' -d '
{
  "description": "Adds timestamp",
  "processors": [
    {
      "set": {
        "field": "_source.timestamp",
        "value": "{{_ingest.timestamp}}"
      }
    }
  ]
}
'
```

## flatten field
```bash
curl -XPUT 'http://localhost:9200/_ingest/pipeline/phone_numbers' -H 'Content-Type: application/json' --data-binary @./transform_test/migration_plan/number_pipeline.json
```

## default pipeline
```bash
curl -XPUT 'http://localhost:9200/customer-1/_settings' -H 'Content-Type: application/json' -d '
{
  "index": {
    "default_pipeline": "phone_numbers"
  }
}
'
```

# data insert (customerに)
```bash
curl -XPUT 'http://localhost:9200/customer/_doc/1' -H 'Content-Type: application/json' -d '
{
  "customer_id": "1",
  "group_code": "code-1",
  "phone_number_list": [
    {
      "phone_number": "09011112222" 
    },
    {
      "phone_number": "09022223333"
    }
  ]
}
'
curl -XPUT 'http://localhost:9200/customer/_doc/2' -H 'Content-Type: application/json' -d '
{
  "customer_id": "2",
  "group_code": "code-1",
  "phone_number_list": {
      "phone_number": "09022223333"
  }
}
'
curl -XPUT 'http://localhost:9200/customer/_doc/3' -H 'Content-Type: application/json' -d '
{
  "customer_id": "3",
  "group_code": "code-1",
  "phone_number_list": [
    {
      "phone_number": "09033334444" 
    },
    {
      "phone_number": "09044445555"
    }
  ]
}
'
curl -XPUT 'http://localhost:9200/customer/_doc/4' -H 'Content-Type: application/json' -d '
{
  "customer_id": "4",
  "group_code": "code-2",
  "phone_number_list": [
    {
      "phone_number": "08011112222" 
    },
    {
      "phone_number": "09011112222"
    }
  ]
}
'
curl -XPUT 'http://localhost:9200/customer/_doc/5' -H 'Content-Type: application/json' -d '
{
  "customer_id": "5",
  "group_code": "code-2",
  "phone_number_list": [
    {
      "phone_number": "09022223333" 
    },
    {
      "phone_number": "08011112222"
    }
  ]
}
'
curl -XPUT 'http://localhost:9200/customer/_doc/6' -H 'Content-Type: application/json' -d '
{
  "customer_id": "6",
  "group_code": "code-3",
  "phone_number_list": [
    {
      "phone_number": "09022223333" 
    },
    {
      "phone_number": "08011112222"
    }
  ]
}
'
```

# reindex
```bash
curl -XPOST 'http://localhost:9200/_reindex' -H 'Content-Type: application/json' -d '
{
  "source": {
    "index": "customer"
  },
  "dest": {
    "index": "customer-1"
  }
}
'
```

# transform
## preview
```bash
curl -XPOST 'http://localhost:9200/_data_frame/transforms/_preview?pretty' -H 'Content-Type: application/json' --data-binary @./transform_test/migration_plan/transform.json
```

```json
{
    "preview" : [
        {
            "group_code" : "code-1",
            "customer_ids" : [
                "1"
            ],
            "phone_number" : "09011112222"
        },
        {
            "group_code" : "code-1",
            "customer_ids" : [
                "1",
                "2"
            ],
            "phone_number" : "09022223333"
        },
        {
            "group_code" : "code-1",
            "customer_ids" : [
                "3"
            ],
            "phone_number" : "09033334444"
        },
        {
            "group_code" : "code-1",
            "customer_ids" : [
                "3"
            ],
            "phone_number" : "09044445555"
        },
        {
            "group_code" : "code-2",
            "customer_ids" : [
                "4",
                "5"
            ],
            "phone_number" : "08011112222"
        },
        {
            "group_code" : "code-2",
            "customer_ids" : [
                "4"
            ],
            "phone_number" : "09011112222"
        },
        {
            "group_code" : "code-2",
            "customer_ids" : [
                "5"
            ],
            "phone_number" : "09022223333"
        },
        {
            "group_code" : "code-3",
            "customer_ids" : [
                "6"
            ],
            "phone_number" : "08011112222"
        },
        {
            "group_code" : "code-3",
            "customer_ids" : [
                "6"
            ],
            "phone_number" : "09022223333"
        }
    ],
    "generated_dest_index" : {
        "mappings" : {
            "_meta" : {
                "_transform" : {
                    "transform" : "transform-preview",
                    "version" : {
                        "created" : "7.7.1"
                    },
                    "creation_date_in_millis" : 1730934808015
                },
                "created_by" : "transform"
            },
            "properties" : {
                "group_code" : {
                    "type" : "keyword"
                },
                "phone_number" : {
                    "type" : "keyword"
                }
            }
        },
        "settings" : {
            "index" : {
                "number_of_shards" : "1",
                "auto_expand_replicas" : "0-1"
            }
        },
        "aliases" : { }
    }
}
```
シャードの設定を変えるためには、事前に同じindexを作成すること。

## create
```bash
curl -XPUT 'http://localhost:9200/_transform/test-transform?pretty' -H 'Content-Type: application/json' --data-binary @./transform_test/migration_plan/transform.json
```

## start
```bash
curl -XPOST 'http://localhost:9200/_transform/test-transform/_start'
```

## transform status
```bash
curl -XGET 'http://localhost:9200/_transform/test-transform1/_stats?pretty'
```

## result
```bash
curl -XGET 'http://localhost:9200/customer_phone/_search?pretty'
```

## insert data to customer-1
```bash
curl -XPUT 'http://localhost:9200/customer-1/_doc/7' -H 'Content-Type: application/json' -d '
{
  "customer_id": "7",
  "group_code": "code-1",
  "phone_number_list": [
    {
      "phone_number": "09011112222" 
    },
    {
      "phone_number": "09022223333"
    }
  ]
}
'
```

## search

|group_code|phone_number|customer_ids|
|:--|:--|:--|
|**code-1**|**09011112222**|**1, 7**|
|**code-1**|**09022223333**|**1, 2, 7**|
|**code-1**|**09033334444**|**3**|
|**code-1**|**09044445555**|**3**|
|code-2|08011112222|4, 5|
|code-2|09011112222|4|
|code-2|09022223333|5|
|code-3|08011112222|6|
|code-3|09022223333|6|

```bash
curl -XGET 'http://localhost:9200/customer_phone/_search?pretty' -H 'Content-Type: application/json' --data-binary @./transform_test/migration_plan/search_from_transform_index.json
```

# else 
```bash
curl -XDELETE 'http://localhost:9200/customer'
curl -XDELETE 'http://localhost:9200/customer-1'
curl -XDELETE 'http://localhost:9200/customer_phone'
curl -XDELETE 'http://localhost:9200/_transform/test-transform'
curl -XGET 'http://localhost:9200/customer/_search?pretty'
curl -XGET 'http://localhost:9200/customer-1/_search?pretty'
curl -XGET 'http://localhost:9200/customer_phone/_search?pretty'
curl -XGET 'http://localhost:9200/customer_phone/_search?pretty' -H 'Content-Type: application/json' -d '
{
  "query": {
    "match": {
      "phone_number": "090"
    }
  }
}
'
curl -XGET 'http://localhost:9200/customer/_mapping?pretty'
curl -XGET 'http://localhost:9200/customer_phone/_mapping?pretty'
curl -XGET 'http://localhost:9200/_cat/indices?v'
```