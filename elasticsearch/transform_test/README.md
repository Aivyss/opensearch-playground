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
# timestampを設定することで、同期できる。
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
curl -XPUT 'http://localhost:9200/customer/_settings' -H 'Content-Type: application/json' -d '
{
  "index": {
    "default_pipeline": "add_timestamp"
  }
}
'
```

# data insert
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

curl -XPOST 'http://localhost:9200/_data_frame/transforms/_preview?pretty' -H 'Content-Type: application/json' --data-binary @./transform_test/transform.json
curl -XPOST 'http://localhost:9200/_data_frame/transforms/_preview?pretty' -H 'Content-Type: application/json' --data-binary @./transform_test/transform2.json
curl -XPOST 'http://localhost:9200/_data_frame/transforms/_preview?pretty' -H 'Content-Type: application/json' --data-binary @./transform_test/transform3.json
```

# preview result
## transform 1
```json
{
    "preview" : [
        {
            "group_code" : "code-1",
            "phone_number" : [
                "09011112222",
                "09022223333"
            ],
            "customer_id" : "1"
        },
        {
            "group_code" : "code-1",
            "phone_number" : [
                "09022223333"
            ],
            "customer_id" : "2"
        },
        {
            "group_code" : "code-1",
            "phone_number" : [
                "09033334444",
                "09044445555"
            ],
            "customer_id" : "3"
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
                    "creation_date_in_millis" : 1730488304677
                },
                "created_by" : "transform"
            },
            "properties" : {
                "group_code" : {
                    "type" : "keyword"
                },
                "customer_id" : {
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
## transform 2
```json
{
  "preview" : [
    {
      "group_code" : "code-1",
      "phone_number_data" : {
        "09044445555" : [
          "3"
        ],
        "09011112222" : [
          "1"
        ],
        "09022223333" : [
          "1",
          "2"
        ],
        "09033334444" : [
          "3"
        ]
      }
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
          "creation_date_in_millis" : 1730490196981
        },
        "created_by" : "transform"
      },
      "properties" : {
        "group_code" : {
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

# transform
## create
```bash
curl -XPUT 'http://localhost:9200/_transform/test-transform1-6?pretty' -H 'Content-Type: application/json' --data-binary @./transform_test/transform.json
curl -XPUT 'http://localhost:9200/_transform/test-transform2-3?pretty' -H 'Content-Type: application/json' --data-binary @./transform_test/transform2.json
```

## start
```bash
curl -XPOST 'http://localhost:9200/_transform/test-transform1-6/_start'
curl -XPOST 'http://localhost:9200/_transform/test-transform2-3/_start'
```

## transform status
```bash
curl -XGET 'http://localhost:9200/_transform/test-transform1-3/_stats?pretty'
```



# else 
```bash
curl -XDELETE 'http://localhost:9200/customer'
curl -XDELETE 'http://localhost:9200/customer_phone'
curl -XGET 'http://localhost:9200/customer/_search?pretty'
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