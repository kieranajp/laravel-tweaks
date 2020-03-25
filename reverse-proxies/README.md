# Reverse Proxies and Load Balancers

When Laravel is behind a reverse proxy or load balancer, it can read the incorrect IP address for a request. This can lead to every request seeming as if it's coming from inside your own datacenter.

![](https://i.imgflip.com/338z54.jpg)

## Solution

Simple one, trust reverse proxies to read forwarded IP headers from them.

```php
//app/Http/Middleware/TrustProxies.php

protected $proxies = '*';
```
