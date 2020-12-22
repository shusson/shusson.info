# Elasticsearch range query datemath and timezones

**22/12/2020**

TLDR: if you use the `time_zone` parameter in a range query, do not set date values with a timezone offset.

When using range queries in elasticsearch, the [API](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-range-query.html#range-query-field-params) allows a `time_zone` to be set. Elasticsearch will ignore the `time_zone` parameter if the value of the range query (gt, lt etc) contains a timezone offset.

For example the following `time_zone` `+02:00` is ignored because the `gte` value contains a zulu time offset (`Z`).

```json
"query": {
  "bool": {
    "must": [
      {
        "range": {
          "date_of_birth": {
            "gte": "2020-01-01T00:00:00Z",
            "time_zone": "+02:00"
          }
        }
      }
    ]
  }
}
```

As far as I can tell, this undocumented in the API, but I think it's understandable given that queries shouldn't be passing multiple timezones. However the situation becomes weirder if you use [datemath](https://www.elastic.co/guide/en/elasticsearch/reference/7.x/common-options.html#date-math) in combination with an offset and a `time_zone`.

```json
"range": {
  "date_of_birth": {
    "gte": "2020-01-01T00:00:00+02:00||/d",
    "time_zone": "+02:00"
  }
}
```

There is a small [note](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-range-query.html#range-query-field-params) in the documentation which says:

> The time_zone parameter does not affect the date math value of now. now is always the current system time in UTC.

> However, the time_zone parameter does convert dates calculated using now and date math rounding. For example, the time_zone parameter will convert a value of now/d.

So there's a hint what ES will do, but we can determine exactly what ES will do using the [Validation API](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-validate.html#search-validate):

Example validation query:

```
curl -XGET "https://<ES_INSTANCE>/<INDEX>/_validate/query?explain=true" -H 'Content-Type: application/json' -d'
{
  "query": {
    "bool": {
      "must": [
        {
          "range": {
            "date_of_birth": {
              "gte": "2020-01-01T00:00:00+02:00||/d",
              "time_zone": "+02:00"
            }
          }
        }
      ]
    }
  }
}'
-------------
{
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "failed" : 0
  },
  "valid" : true,
  "explanations" : [
    {
      "index" : "xxx",
      "valid" : true,
      "explanation" : "+date_of_birth:[1577829600000 TO 9223372036854775807]"
    }
  ]
}
```

ES converts the range query to a lower-bound of `1577916000000` and an upper-bound of `9223372036854775807`. We can ignore the upper-bound since it is a hard limit, the lower-bound converts to `2019-12-31T22:00:00Z`.

With some experimentation we can guess what ES is up to. First ES will convert `2020-01-01T00:00:00+0200` to UTC, `2019-12-31T22:00:00Z`. It will then apply the [rounding logic](https://www.elastic.co/guide/en/elasticsearch/reference/7.x/query-dsl-range-query.html#range-query-date-math-rounding) which, since we have `gte`, rounds down to the first millisecond, `2019-12-31T00:00:00Z`. Finally, and most surprisingly to me, ES will use the `time_zone` value to convert the date to `2019-12-31T22:00:00Z`.

So if you are going to send the `time_zone` parameter, save yourself a lot of confusion and send date strings without timezone offsets:

```json
"query": {
  "bool": {
    "must": [
      {
        "range": {
          "date_of_birth": {
            "gte": "2020-01-01T00:00:00",
            "time_zone": "+02:00"
          }
        }
      }
    ]
  }
}
```

```json
"query": {
  "bool": {
    "must": [
      {
        "range": {
          "date_of_birth": {
            "gte": "2020-01-01T00:00:00||/d",
            "time_zone": "+02:00"
          }
        }
      }
    ]
  }
}
```

That way `time_zone` is never ignored and when using datemath, ES will not convert your dates twice.
