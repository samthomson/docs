# Cache

- [Configuration](#configuration)
- [Cache Usage](#cache-usage)
    - [Obtaining A Cache Instance](#obtaining-a-cache-instance)
    - [Retrieving Items From The Cache](#retrieving-items-from-the-cache)
    - [Storing Items In The Cache](#storing-items-in-the-cache)
    - [Removing Items From The Cache](#removing-items-from-the-cache)
- [Adding Custom Cache Drivers](#adding-custom-cache-drivers)
- [Cache Tags](#cache-tags)
    - [Storing Tagged Cache Items](#storing-tagged-cache-items)
    - [Accessing Tagged Cache Items](#accessing-tagged-cache-items)
- [Cache Events](#cache-events)

<a name="configuration"></a>
## Configuration

Laravel provides a unified API for various caching systems. The cache configuration is located at `config/cache.php`. In this file you may specify which cache driver you would like used by default throughout your application. Laravel supports popular caching backends like [Memcached](http://memcached.org) and [Redis](http://redis.io) out of the box.

The cache configuration file also contains various other options, which are documented within the file, so make sure to read over these options. By default, Laravel is configured to use the `file` cache driver, which stores the serialized, cached objects in the filesystem. For larger applications, it is recommended that you use an in-memory cache such as Memcached or APC. You may even configure multiple cache configurations for the same driver.

### Cache Prerequisites

#### Database

When using the `database` cache driver, you will need to setup a table to contain the cache items. You'll find an example `Schema` declaration for the table below:

    Schema::create('cache', function($table) {
        $table->string('key')->unique();
        $table->text('value');
        $table->integer('expiration');
    });

#### Memcached

Using the Memcached cache requires the [Memcached PECL package](http://pecl.php.net/package/memcached) to be installed.

The default [configuration](#configuration) uses TCP/IP based on [Memcached::addServer](http://php.net/manual/en/memcached.addserver.php):

    'memcached' => [
        [
            'host' => '127.0.0.1',
            'port' => 11211,
            'weight' => 100
        ],
    ],

You may also set the `host` option to a UNIX socket path. If you do this, the `port` option should be set to `0`:

    'memcached' => [
        [
            'host' => '/var/run/memcached/memcached.sock',
            'port' => 0,
            'weight' => 100
        ],
    ],

#### Redis

Before using a Redis cache with Laravel, you will need to install the `predis/predis` package (~1.0) via Composer.

For more information on configuring Redis, consult its [Laravel documentation page](/docs/{{version}}/redis#configuration).

<a name="cache-usage"></a>
## Cache Usage

<a name="obtaining-a-cache-instance"></a>
### Obtaining A Cache Instance

The `Illuminate\Contracts\Cache\Factory` and `Illuminate\Contracts\Cache\Repository` [contracts](/docs/{{version}}/contracts) provide access to Laravel's cache services. The `Factory` contract provides access to all cache drivers defined for your application. The `Repository` contract is typically an implementation of the default cache driver for your application as specified by your `cache` configuration file.

However, you may also use the `Cache` facade, which is what we will use throughout this documentation. The `Cache` facade provides convenient, terse access to the underlying implementations of the Laravel cache contracts.

For example, let's import the `Cache` facade into a controller:

    <?php

    namespace App\Http\Controllers;

    use Cache;
    use Illuminate\Routing\Controller;

    class UserController extends Controller
    {
        /**
         * Show a list of all users of the application.
         *
         * @return Response
         */
        public function index()
        {
            $value = Cache::get('key');

            //
        }
    }

#### Accessing Multiple Cache Stores

Using the `Cache` facade, you may access various cache stores via the `store` method. The key passed to the `store` method should correspond to one of the stores listed in the `stores` configuration array in your `cache` configuration file:

    $value = Cache::store('file')->get('foo');

    Cache::store('redis')->put('bar', 'baz', 10);

<a name="retrieving-items-from-the-cache"></a>
### Retrieving Items From The Cache

The `get` method on the `Cache` facade is used to retrieve items from the cache. If the item does not exist in the cache, `null` will be returned. If you wish, you may pass a second argument to the `get` method specifying the custom default value you wish to be returned if the item doesn't exist:

    $value = Cache::get('key');

    $value = Cache::get('key', 'default');


You may even pass a `Closure` as the default value. The result of the `Closure` will be returned if the specified item does not exist in the cache. Passing a Closure allows you to defer the retrieval of default values from a database or other external service:

    $value = Cache::get('key', function() {
        return DB::table(...)->get();
    });

#### Checking For Item Existence

The `has` method may be used to determine if an item exists in the cache:

    if (Cache::has('key')) {
        //
    }

#### Incrementing / Decrementing Values

The `increment` and `decrement` methods may be used to adjust the value of integer items in the cache. Both of these methods optionally accept a second argument indicating the amount by which to increment or decrement the item's value:

    Cache::increment('key');

    Cache::increment('key', $amount);

    Cache::decrement('key');

    Cache::decrement('key', $amount);

#### Retrieve Or Update

Sometimes you may wish to retrieve an item from the cache, but also store a default value if the requested item doesn't exist. For example, you may wish to retrieve all users from the cache or, if they don't exist, retrieve them from the database and add them to the cache. You may do this using the `Cache::remember` method:

    $value = Cache::remember('users', $minutes, function() {
        return DB::table('users')->get();
    });

If the item does not exist in the cache, the `Closure` passed to the `remember` method will be executed and its result will be placed in the cache.

You may also combine the `remember` and `forever` methods:

    $value = Cache::rememberForever('users', function() {
        return DB::table('users')->get();
    });

#### Retrieve And Delete

If you need to retrieve an item from the cache and then delete it, you may use the `pull` method. Like the `get` method, `null` will be returned if the item does not exist in the cache:

    $value = Cache::pull('key');

<a name="storing-items-in-the-cache"></a>
### Storing Items In The Cache

You may use the `put` method on the `Cache` facade to store items in the cache. When you place an item in the cache, you will need to specify the number of minutes for which the value should be cached:

    Cache::put('key', 'value', $minutes);

Instead of passing the number of minutes until the item expires, you may also pass a PHP `DateTime` instance representing the expiration time of the cached item:

    $expiresAt = Carbon::now()->addMinutes(10);

    Cache::put('key', 'value', $expiresAt);

The `add` method will only add the item to the cache if it does not already exist in the cache store. The method will return `true` if the item is actually added to the cache. Otherwise, the method will return `false`:

    Cache::add('key', 'value', $minutes);

The `forever` method may be used to store an item in the cache permanently. These values must be manually removed from the cache using the `forget` method:

    Cache::forever('key', 'value');

<a name="removing-items-from-the-cache"></a>
### Removing Items From The Cache

You may remove items from the cache using the `forget` method on the `Cache` facade:

    Cache::forget('key');

You may clear the entire caching using the `flush` method:

    Cache::flush();

Flushing the cache **does not** respect the cache prefix and will remove all entries from the cache. Consider this carefully when clearing a cache which is shared by other applications.

<a name="adding-custom-cache-drivers"></a>
## Adding Custom Cache Drivers

To extend the Laravel cache with a custom driver, we will use the `extend` method on the `Cache` facade, which is used to bind a custom driver resolver to the manager. Typically, this is done within a [service provider](/docs/{{version}}/providers).

For example, to register a new cache driver named "mongo":

    <?php

    namespace App\Providers;

    use Cache;
    use App\Extensions\MongoStore;
    use Illuminate\Support\ServiceProvider;

    class CacheServiceProvider extends ServiceProvider
    {
        /**
         * Perform post-registration booting of services.
         *
         * @return void
         */
        public function boot()
        {
            Cache::extend('mongo', function($app) {
                return Cache::repository(new MongoStore);
            });
        }

        /**
         * Register bindings in the container.
         *
         * @return void
         */
        public function register()
        {
            //
        }
    }

The first argument passed to the `extend` method is the name of the driver. This will correspond to your `driver` option in the `config/cache.php` configuration file. The second argument is a Closure that should return an `Illuminate\Cache\Repository` instance. The Closure will be passed an `$app` instance, which is an instance of the [service container](/docs/{{version}}/container).

The call to `Cache::extend` could be done in the `boot` method of the default `App\Providers\AppServiceProvider` that ships with fresh Laravel applications, or you may create your own service provider to house the extension - just don't forget to register the provider in the `config/app.php` provider array.

To create our custom cache driver, we first need to implement the `Illuminate\Contracts\Cache\Store` [contract](/docs/{{version}}/contracts) contract. So, our MongoDB cache implementation would look something like this:

    <?php

    namespace App\Extensions;

    class MongoStore implements \Illuminate\Contracts\Cache\Store
    {
        public function get($key) {}
        public function put($key, $value, $minutes) {}
        public function increment($key, $value = 1) {}
        public function decrement($key, $value = 1) {}
        public function forever($key, $value) {}
        public function forget($key) {}
        public function flush() {}
        public function getPrefix() {}
    }

We just need to implement each of these methods using a MongoDB connection. Once our implementation is complete, we can finish our custom driver registration:

    Cache::extend('mongo', function($app) {
        return Cache::repository(new MongoStore);
    });

Once your extension is complete, simply update your `config/cache.php` configuration file's `driver` option to the name of your extension.

If you're wondering where to put your custom cache driver code, consider making it available on Packagist! Or, you could create an `Extensions` namespace within your `app` directory. However, keep in mind that Laravel does not have a rigid application structure and you are free to organize your application according to your preferences.

<a name="cache-tags"></a>
## Cache Tags

> **Note:** Cache tags are not supported when using the `file` or `database` cache drivers. Furthermore, when using multiple tags with caches that are stored "forever", performance will be best with a driver such as `memcached`, which automatically purges stale records.

<a name="storing-tagged-cache-items"></a>
### Storing Tagged Cache Items

Cache tags allow you to tag related items in the cache and then flush all cached values that assigned a given tag. You may access a tagged cache by passing in an ordered array of tag names. For example, let's access a tagged cache and `put` value in the cache:

	Cache::tags(['people', 'artists'])->put('John', $john, $minutes);

	Cache::tags(['people', 'authors'])->put('Anne', $anne, $minutes);

However, you are not limited to the `put` method. You may use any cache storage method while working with tags.

<a name="accessing-tagged-cache-items"></a>
### Accessing Tagged Cache Items

To retrieve a tagged cache item, pass the same ordered list of tags to the `tags` method:

	$john = Cache::tags(['people', 'artists'])->get('John');

    $anne = Cache::tags(['people', 'authors'])->get('Anne');

You may flush all items that are assigned a tag or list of tags. For example, this statement would remove all caches tagged with either `people`, `authors`, or both. So, both `Anne` and `John` would be removed from the cache:

	Cache::tags(['people', 'authors'])->flush();

In contrast, this statement would remove only caches tagged with `authors`, so `Anne` would be removed, but not `John`.

	Cache::tags('authors')->flush();

<a name="cache-events"></a>
## Cache Events

To execute code on every cache operation, you may listen for the [events](/docs/{{version}}/events) fired by the cache. Typically, you should place these event listeners within the `boot` method of your `EventServiceProvider`:

    /**
     * Register any other events for your application.
     *
     * @param  \Illuminate\Contracts\Events\Dispatcher  $events
     * @return void
     */
    public function boot(DispatcherContract $events)
    {
        parent::boot($events);

        $events->listen('cache.hit', function ($key, $value) {
            //
        });

        $events->listen('cache.missed', function ($key) {
            //
        });

        $events->listen('cache.write', function ($key, $value, $minutes) {
            //
        });

        $events->listen('cache.delete', function ($key) {
            //
        });
    }
