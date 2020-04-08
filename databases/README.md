# Databases

Please delete Eloquent entirely.

## Redis

### Problem

Okay, specifically onto Redis, whether that's for caching / sessions / as a KV store / as a message bus, for some godforsaken reason (though please use _literally anything_ that was actually designed for this purpose - SQS, RabbitMQ, IronMQ, 0MQ, whatever).

If your Redis server requires a TLS connection (denoted by either `tls://` or `rediss://` with a double-s in the DSN), then it's not going to work with stock Laravel. For example, if you're trying to use a DigitalOcean managed Redis instance, then TLS will be required, and the connection will fail.

### Solution

Luckily, you can get round this very simply by overriding the scheme that's passed to the underlying Redis driver. Laravel doesn't allow overriding this by default, but there're workarounds.

#### Using Predis

If you're using [Predis](https://github.com/nrk/predis), Laravel [passes the whole config array](https://github.com/laravel/framework/blob/7.x/src/Illuminate/Redis/Connectors/PredisConnector.php#L29) down, so with a little tweaking we can instruct Predis to connect with TLS.

Here's how Predis accepts connection details:

```php
$client = new Predis\Client([
    'scheme' => 'tcp',
    'host'   => '10.0.0.1',
    'port'   => 6379,
]);
```

So you can add the scheme parameter to your config:

```php
// config/database.php

return [
    // ...

    'redis' => [
        // ...

        'default' => [
            'scheme' => env('REDIS_SCHEME', 'tcp'), // Add this!
            'url' => env('REDIS_URL'),
            'host' => env('REDIS_HOST', '127.0.0.1'),
            'password' => env('REDIS_PASSWORD', null),
            'port' => env('REDIS_PORT', '6379'),
            'database' => env('REDIS_DB', '0'),
        ],

        // ...
    ],

    // ...
];

```

You can then set the `REDIS_SCHEME` environment variable to `tls` or `rediss` to (hopefully) successfully connect to your secure Redis instance!

#### Using PhpRedis

[PhpRedis](https://github.com/phpredis/phpredis) accepts the connection string slighly differently, so we can't modify the config array and expect it to work. The parameters we want it to accept look something like this:

```php
$redis->pconnect('tls://127.0.0.1', 6379);
```

Laravel actually [spreads part of the config array into this method](https://github.com/laravel/framework/blob/7.x/src/Illuminate/Redis/Connectors/PhpRedisConnector.php#L118-L130) - so we can actually get this working with a well-formed `REDIS_HOST` environment variable. Setting this to have the `tls://` or `rediss://` prefix seems to do the job!