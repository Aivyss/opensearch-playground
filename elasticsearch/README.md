# cheatsheet
## create index
```bash
curl -H "Content-Type: application/json" -XPUT http://localhost:9200/shakespeare --data-binary @./practice/shakes-mapping.json
```

## list up indices
```bash
curl -XGET http://localhost:9200/_cat/indices
```

## bulk input
```bash
curl -H "Content-Type: application/json" -XPOST http://localhost:9200/shakespeare/_bulk --data-binary @./practice/shakespeare_7.0.json
```

## search
```bash
curl -H "Content-Type: application/json" -XGET 'http://localhost:9200/shakespeare/_search?pretty' -d '
{
  "query": {
    "match_phrase": {
      "text_entry": "to be or not to be"
    }
  }
}
'
```