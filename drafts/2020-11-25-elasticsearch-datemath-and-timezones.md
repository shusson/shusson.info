# Elasticsearch datemath and timezones

__25/11/2020__

Datemath documentation: https://www.elastic.co/guide/en/elasticsearch/reference/7.x/common-options.html#date-math

```json
"query": {
  "bool": {
    "must": [
      {
        "range": {
          "start_date": {
            "gt": "2020-01-01T00:00:00.000Z||/d",
            "time_zone": "Europe/Amsterdam"
          }
        }
      }
    ]
  }
}
```

Interesting scenarios:

- no offset with timezone
- offset with timezone
- offset with timezone and rounding


<!-- https://www.epochconverter.com/ -->

<!-- GET index/_validate/query?explain=true -->

```
{
  "query": {
    "nested": {
      "score_mode": "none",
      "path": "admission",
      "query": {
        "bool": {
          "must": [
            {
              "range": {
                "admission.start_date": {
                  "gt": "2020-01-01T00:00:00.000||/d",
                  "time_zone": "Europe/Amsterdam"
                }
              }
            }
          ]
        }
      }
    }
  }
}
```
