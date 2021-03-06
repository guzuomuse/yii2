Caching
=======

Caching is a cheap and effective way to improve the performance of a web application. By storing relatively
static data in cache and serving it from cache when requested, the application saves the time required to generate the data from scratch. Caching is one of the best ways to improve the performance of your application, almost mandatory on any large-scale site.


Base Concepts
-------------

Using cache in Yii involves configuring and accessing a cache application component. The following
application configuration specifies a cache component that uses [memcached](http://memcached.org/) with
two cache servers. Note, this configuration should be done in file located at `@app/config/web.php` alias
in case you're using basic sample application.

```php
'components' => [
	'cache' => [
		'class' => '\yii\caching\MemCache',
		'servers' => [
			[
				'host' => 'server1',
				'port' => 11211,
				'weight' => 100,
			],
			[
				'host' => 'server2',
				'port' => 11211,
				'weight' => 50,
			],
		],
	],
],
```

When the application is running, the cache component can be accessed through `Yii::$app->cache` call.

Yii provides various cache components that can store cached data in different media. The following
is a summary of the available cache components:

* [[\yii\caching\ApcCache]]: uses PHP [APC](http://php.net/manual/en/book.apc.php) extension. This option can be
  considered as the fastest one when dealing with cache for a centralized thick application (e.g. one
  server, no dedicated load balancers, etc.).

* [[\yii\caching\DbCache]]: uses a database table to store cached data. By default, it will create and use a
  [SQLite3](http://sqlite.org/) database under the runtime directory. You can explicitly specify a database for
  it to use by setting its `db` property.

* [[\yii\caching\DummyCache]]: presents dummy cache that does no caching at all. The purpose of this component
  is to simplify the code that needs to check the availability of cache. For example, during development or if
  the server doesn't have actual cache support, we can use this cache component. When an actual cache support
  is enabled, we can switch to use the corresponding cache component. In both cases, we can use the same
  code `Yii::$app->cache->get($key)` to attempt retrieving a piece of data without worrying that
  `Yii::$app->cache` might be `null`.

* [[\yii\caching\FileCache]]: uses standard files to store cached data. This is particular suitable
  to cache large chunk of data (such as pages).

* [[\yii\caching\MemCache]]: uses PHP [memcache](http://php.net/manual/en/book.memcache.php)
  and [memcached](http://php.net/manual/en/book.memcached.php) extensions. This option can be considered as
  the fastest one when dealing with cache in a distributed applications (e.g. with several servers, load
  balancers, etc.)

* [[\yii\caching\RedisCache]]: implements a cache component based on [Redis](http://redis.io/) key-value store
  (redis version 2.6.12 or higher is required).

* [[\yii\caching\WinCache]]: uses PHP [WinCache](http://iis.net/downloads/microsoft/wincache-extension)
  ([see also](http://php.net/manual/en/book.wincache.php)) extension.

* [[\yii\caching\XCache]]: uses PHP [XCache](http://xcache.lighttpd.net/) extension.

* [[\yii\caching\ZendDataCache]]: uses
  [Zend Data Cache](http://files.zend.com/help/Zend-Server-6/zend-server.htm#data_cache_component.htm)
  as the underlying caching medium.

Tip: because all these cache components extend from the same base class [[Cache]], one can switch to use
a different type of cache without modifying the code that uses cache.

Caching can be used at different levels. At the lowest level, we use cache to store a single piece of data,
such as a variable, and we call this data caching. At the next level, we store in cache a page fragment which
is generated by a portion of a view script. And at the highest level, we store a whole page in cache and serve
it from cache as needed.

In the next few subsections, we elaborate how to use cache at these levels.

Note, by definition, cache is a volatile storage medium. It does not ensure the existence of the cached
data even if it does not expire. Therefore, do not use cache as a persistent storage (e.g. do not use cache
to store session data or other valuable information).

Data Caching
------------

Data caching is about storing some PHP variable in cache and retrieving it later from cache. For this purpose,
the cache component base class [[\yii\caching\Cache]] provides two methods that are used most of the time:
[[set()]] and [[get()]]. Note, only serializable variables and objects could be cached successfully.

To store a variable `$value` in cache, we choose a unique `$key` and call [[set()]] to store it:

```php
Yii::$app->cache->set($key, $value);
```

The cached data will remain in the cache forever unless it is removed because of some caching policy
(e.g. caching space is full and the oldest data are removed). To change this behavior, we can also supply
an expiration parameter when calling [[set()]] so that the data will be removed from the cache after
a certain period of time:

```php
// keep the value in cache for at most 45 seconds
Yii::$app->cache->set($key, $value, 45);
```

Later when we need to access this variable (in either the same or a different web request), we call [[get()]]
with the key to retrieve it from cache. If the value returned is `false`, it means the value is not available
in cache and we should regenerate it:

```php
public function getCachedData()
{
	$key = /* generate unique key here */;
	$value = Yii::$app->getCache()->get($key);
	if ($value === false) {
		$value = /* regenerate value because it is not found in cache and then save it in cache for later use */;
		Yii::$app->cache->set($key, $value);
	}
	return $value;
}
```

This is the common pattern of arbitrary data caching for general use.

When choosing the key for a variable to be cached, make sure the key is unique among all other variables that
may be cached in the application. It is **NOT** required that the key is unique across applications because
the cache component is intelligent enough to differentiate keys for different applications.

Some cache storages, such as MemCache, APC, support retrieving multiple cached values in a batch mode,
which may reduce the overhead involved in retrieving cached data. A method named [[mget()]] is provided
to exploit this feature. In case the underlying cache storage does not support this feature,
[[mget()]] will still simulate it.

To remove a cached value from cache, call [[delete()]]; and to remove everything from cache, call [[flush()]].
Be very careful when calling [[flush()]] because it also removes cached data that are from other applications.

Note, because [[Cache]] implements `ArrayAccess`, a cache component can be used liked an array. The followings
are some examples:

```php
$cache = Yii::$app->getComponent('cache');
$cache['var1'] = $value1;  // equivalent to: $cache->set('var1', $value1);
$value2 = $cache['var2'];  // equivalent to: $value2 = $cache->get('var2');
```

### Cache Dependency

Besides expiration setting, cached data may also be invalidated according to some dependency changes. For example, if we
are caching the content of some file and the file is changed, we should invalidate the cached copy and read the latest
content from the file instead of the cache.

We represent a dependency as an instance of [[\yii\caching\Dependency]] or its child class. We pass the dependency
instance along with the data to be cached when calling `set()`.

```php
use yii\cache\FileDependency;

// the value will expire in 30 seconds
// it may also be invalidated earlier if the dependent file is changed
Yii::$app->cache->set($id, $value, 30, new FileDependency(['fileName' => 'example.txt']));
```

Now if we retrieve $value from cache by calling `get()`, the dependency will be evaluated and if it is changed, we will
get a false value, indicating the data needs to be regenerated.

Below is a summary of the available cache dependencies:

- [[\yii\cache\FileDependency]]: the dependency is changed if the file's last modification time is changed.
- [[\yii\cache\GroupDependency]]: marks a cached data item with a group name. You may invalidate the cached data items
  with the same group name all at once by calling [[\yii\cache\GroupDependency::invalidate()]].
- [[\yii\cache\DbDependency]]: the dependency is changed if the query result of the specified SQL statement is changed.
- [[\yii\cache\ChainedDependency]]: the dependency is changed if any of the dependencies on the chain is changed.
- [[\yii\cache\ExpressionDependency]]: the dependency is changed if the result of the specified PHP expression is
  changed.

### Query Caching

For caching the result of database queries you can wrap them in calls to [[yii\db\Connection::beginCache()]]
and [[yii\db\Connection::endCache()]]:

```php
$connection->beginCache(60); // cache all query results for 60 seconds.
// your db query code here...
$connection->endCache();
```


Fragment Caching
----------------

TBD: http://www.yiiframework.com/doc/guide/1.1/en/caching.fragment

### Caching Options

TBD: http://www.yiiframework.com/doc/guide/1.1/en/caching.fragment#caching-options

### Nested Caching

TBD: http://www.yiiframework.com/doc/guide/1.1/en/caching.fragment#nested-caching

Dynamic Content
---------------

TBD: http://www.yiiframework.com/doc/guide/1.1/en/caching.dynamic

Page Caching
------------

TBD: http://www.yiiframework.com/doc/guide/1.1/en/caching.page

### Output Caching

TBD: http://www.yiiframework.com/doc/guide/1.1/en/caching.page#output-caching

### HTTP Caching

TBD: http://www.yiiframework.com/doc/guide/1.1/en/caching.page#http-caching
