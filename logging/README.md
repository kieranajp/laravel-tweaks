# Logging

By default, Laravel logs to files. This causes a) problems with permissions, and b) logs to not be readily accessible if you're running containers.

For these reasons I much prefer to log to `stderr`. I also prefer to log in a structured format such as that specified by Logstash.

## Problem

While Laravel seems to allow this from its config, it didn't work for me. The following config:

```php
// config/logger.php

return [
    'default' => env('LOG_CHANNEL', 'stderr'),
    'channels' => [
        'stderr' => [
            'driver' => 'monolog',
            'handler' => StreamHandler::class,
            'formatter' => LogstashFormatter::class,
            'with' => [
                'stream' => 'php://stderr',
            ],
            'level' => env('LOG_LEVEL', 'info'),
        ],
    ]
];
```

```php
// routes/web.php

Route::get('/test', function () {
    Log::info('log');
    error_log('error_log');
    return 'nice';
});
```

Produced the following output:

```sh
$ docker-compose logs -f php
Attaching to test_php_1
test_php_1    | [21-Mar-2020 12:36:23] NOTICE: fpm is running, pid 1
test_php_1    | [21-Mar-2020 12:36:23] NOTICE: ready to handle connections
test_php_1    | NOTICE: PHP message: error_log
```

## Solution

Creating a custom Monolog handler and invoking that for logs seems to do the trick.

```php
// config/logging.php

return [
    'default' => 'stderr',

    'channels' => [
        'stderr' => [
            'driver' => 'custom',
            'via' => LogDriverFactory::class,
        ],
    ],
];
```

```php
// LogDriverFactory.php (wherever you choose to put it)

use Monolog\Logger;
use Monolog\Handler\StreamHandler;
use Monolog\Formatter\LogstashFormatter;

class LogDriverFactory
{
    public function __invoke(array $config): Logger
    {
        $log = new Logger('stderr');
        $stdErr = new StreamHandler('php://stderr', Logger::DEBUG);
        $stdErr->setFormatter(new LogstashFormatter(env('APP_NAME')));

        $log->pushHandler($stdErr);

        return $log;
    }
}
```

