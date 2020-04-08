# Databases

Please delete Eloquent entirely.

## Redis

### Problem

Okay, specifically onto Redis, whether that's for caching / sessions / as a KV store / as a message bus, for some godforsaken reason (though please use _literally anything_ that was actually designed for this purpose - SQS, RabbitMQ, IronMQ, 0MQ, whatever).

If your Redis server requires a TLS connection (denoted by either `tls://` or `rediss://` with a double-s in the DSN), then it's not going to work with stock Laravel. For example, if you're trying to use a DigitalOcean managed Redis instance, then TLS will be required, and the connection will fail.

### Solution

Luckily, you can get round this very simply by overriding the scheme that's passed to the underlying Redis driver. Laravel doesn't allow overriding this by default, but if you're using [Predis]() it [passes the whole array](https://github.com/laravel/framework/blob/7.x/src/Illuminate/Redis/Connectors/PredisConnector.php#L29).

Here's how Predis accepts connection details:

```php
$client = new Predis\Client([
    'scheme' => 'tcp',
    'host'   => '10.0.0.1',
    'port'   => 6379,
]);
```

PhpRedis accepts it as a DSN, and Laravel doesn't seem to parse the scheme for that unfortunately, so you're shit out of luck with that driver. Switching to Predis requires installing it (`composer require predis/predis`), and changing the `REDIS_CLIENT` environment variable to `predis`. Then you can add the scheme parameter to your config:

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