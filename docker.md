![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/wbhiRD1.webp)

```
"registry-mirrors": [ "https://dockerproxy.com" ]
```

Lokal utvikling og feilsøking

```
PG_HOST=pg

REDIS_HOST=redis
```

endre seg til

```
PG_HOST=127.0.0.1

REDIS_HOST=127.0.0.1

```

Og kommenter ut `NODE_ENV=production`