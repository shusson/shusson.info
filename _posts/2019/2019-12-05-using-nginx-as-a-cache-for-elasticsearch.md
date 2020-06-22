# Using nginx as a cache for elasticsearch

__05/12/2019__

![caching](https://imgs.xkcd.com/comics/the_cloud.png)

Elasticsearch has it's own caching but there are situations where adding a more application aware cache
can speed up response times and reduce the load on elasticsearch.

The search api is a good candidate since complicated queries can take a long time (seconds)
especially if the query has painless scripts which access source nested documents.

Setting up an nginx cache is pretty straightforward, it looks something like this:

```nginx
# this creates a cache in /tmp/cache called escache. 10MB size, inactive after 10m.
proxy_cache_path /tmp/cache levels=1:2 keys_zone=escache:10m inactive=10m;

server {

   # simple regex to only cache requests that end in `_search`
   location ~ /.*\/_search\/*$ {
      # (I'm assuming the configuration already has a reverse proxy)

      # tell nginx which cache to use
      proxy_cache escache;

      # the search api also works with GET, but we only cache POST
      proxy_cache_methods POST;

      proxy_cache_key "$request_uri";

      proxy_cache_valid 10m;

      # prevents ES from being overloaded by same request
      proxy_cache_lock on;

      # lets you know how the response was handled for debugging
      add_header X-Cached $upstream_cache_status;
   }
}
```

The trick with the search api is that we need a good cache key.
We can't just use the URI because the search api almost always contains a body, which determines how elasticsearch will process the request.

We could use the request body directly in the cache key:

```nginx
   proxy_cache_key "$request_uri|$request_body";

   # tuning buffers is required if you want to use request_body in the cache key directly
   proxy_buffers 8 32k;
   proxy_buffer_size 64k;
```

But that requires tuning the buffers, which can be [tricky](https://www.getpagespeed.com/server-setup/nginx/tuning-proxy_buffer_size-in-nginx).
If you expect small request bodies (<100KB), modifying buffers is probably fine.
However if you ever expect large queries, and you can control headers in your requests, we can avoid fiddling with request buffers by using a custom header in the cache key.

```nginx
   proxy_cache_key "$request_uri|$http_custom_search_key";

   # only enable the cache if the header is present
   set $no_cache "true";
   if ($http_custom_search_key) {
      set $no_cache "0";
   }
   proxy_no_cache $no_cache;
   proxy_cache_bypass $no_cache;
```

Now we just need to add the header in our application whenever we want a request cached.

```typescript

   const request = {
      body: {
         query: {
               // your query
            }
         }
      }
   };

   const requestMd5 = crypto
      .createHash("sha256")
      .update(JSON.stringify(request), "utf8")
      .digest("hex");

   const request = {
      ...request,
      headers: {
            "custom-search-key": requestMd5
      }
   }

   const result = await this.elastic.search(request);
```
