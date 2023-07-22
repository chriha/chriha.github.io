---
layout: post
date: 2023-07-23 00:00:00 +0200
title: "Creating an In-Memory Cache Driver with Redis as fallback in Laravel"
categories: [Development]
tags: [php, development, laravel, redis, cache]
image: /assets/img/posts/laravel-in-memory.png
---

One of the many challenges of building a scalable application is to ensure that it can handle a large number of concurrent requests without compromising performance. Caching is a popular technique to improve application performance by storing frequently accessed data in memory. In this post, we will explore how to create an in-memory cache driver with Redis as a fallback in Laravel.

## Understanding Laravel Cache Drivers

Laravel supports multiple cache drivers out of the box, including:

- **File**: Stores cached data in files on the disk.
- **Database**: Stores cached data in a database table.
- **Redis**: Stores cached data in a Redis server.
- **Memcached**: Stores cached data in a Memcached server.
- **Array**: Stores cached data in an array in memory.

You can configure the cache driver in the `config/cache.php` file. For example, to use the Redis cache driver, set the `default` option to `redis`:

```php
'default' => env('CACHE_DRIVER', 'redis'),
```

## Creating an In-Memory Cache Driver

To create an in-memory cache driver, follow these steps:

1. Create a new class that implements the `Illuminate\Contracts\Cache\Store` interface:

    {% gist f62f44203bc8f857744ee0f1f365e83e %}

2. Register the new cache driver in the `config/cache.php` file:

    ```php
    'stores' => [
        'in-memory' => [
            'driver' => 'in-memory',
            'limit' => 1000,
        ],
    ],
    ```

3. Add the `in-memory` cache driver to the `config/cache.php` file:

    ```php
    'default' => env('CACHE_DRIVER', 'in-memory'),
    ```

## Using the In-Memory Cache Driver

You can use the in-memory cache driver like any other cache driver:

```php
// app/Http/Controllers/ExampleController.php

<?php

namespace App\Http\Controllers;

use Illuminate\Support\Facades\Cache;

class ExampleController extends Controller
{
    public function index()
    {
        // Store an item in the cache for 60 seconds...
        Cache::put('key', 'value', 60);

        // Retrieve an item from the cache...
        $value = Cache::get('key');

        // Retrieve an item from the cache, or store the default value forever...
        $value = Cache::rememberForever('key', function () {
            return 'value';
        });
    }
}
```

## Conclusion

In this post, we explored how to create an in-memory cache driver with Redis as a fallback in Laravel. This cache driver is useful for high-frequent usage during a single runtime. If you have any questions or feedback, feel free to leave a comment below.

## Further Reading

- [Laravel Cache](https://laravel.com/docs/10.x/cache)
- [Laravel Cache Drivers](https://laravel.com/docs/10.x/cache#driver-prerequisites)
- [Laravel Cache Custom Drivers](https://laravel.com/docs/10.x/cache#adding-custom-cache-drivers)
