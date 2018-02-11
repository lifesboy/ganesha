# Ganesha

Ganesha is PHP implementation of [Circuit Breaker pattern](http://martinfowler.com/bliki/CircuitBreaker.html) which has multi strategies to avoid cascading failures and supports various storages to record statistics.

[![Build Status](https://travis-ci.org/ackintosh/ganesha.svg?branch=master)](https://travis-ci.org/ackintosh/ganesha) [![Coverage Status](https://coveralls.io/repos/github/ackintosh/ganesha/badge.svg?branch=master)](https://coveralls.io/github/ackintosh/ganesha?branch=master)

![ganesha](https://ackintosh.github.io/assets/images/ganesha.png)

It is one of the very active Circuit Breaker in PHP and production ready: well-tested, well-documented. :muscle:  You can integrate Ganesha to your existing code base easily as Ganesha provides just simple interface and [Guzzle Middleware](https://github.com/ackintosh/ganesha#ganesha-heart-guzzle) behaves transparency.

If you have an idea about enhancement, bugfix, etc..., please let me know it via [Issues](https://github.com/ackintosh/ganesha/issues). :sparkles:

## Are you interested?

[Here](./examples) is an example which is easily executable. All you need is Docker.  
You can experience how Ganesha behaves when a failure occurs.

## Unveil Ganesha

```bash
# Install Composer
$ curl -sS https://getcomposer.org/installer | php

# Run the Composer command to install the latest version of Ganesha
$ php composer.phar require ackintosh/ganesha
```

## Usage

Ganesha provides following simple interface. Each method receives a string (named `$service` in example) to identify the service. `$service` will be the service name of the API, the endpoint name or etc. Please remember that Ganesha detects system failure for each `$service`.

```php
$ganesha->isAvailable($service);
$ganesha->success($service);
$ganesha->failure($service);
```

```php
$ganesha = Ackintosh\Ganesha\Builder::build([
    'failureRateThreshold' => 50,
    'adapter'              => new Ackintosh\Ganesha\Storage\Adapter\Redis($redis),
]);

$service = 'external_api';

if (!$ganesha->isAvailable($service)) {
    die('external api is not available');
}

try {
    echo \Api::send($request)->getBody();
    $ganesha->success($service);
} catch (\Api\RequestTimedOutException $e) {
    // If an error occurred, it must be recorded as failure.
    $ganesha->failure($service);
    die($e->getMessage());
}
```

#### Subscribe to events in ganesha

```php
$ganesha->subscribe(function ($event, $service, $message) {
    switch ($event) {
        case Ganesha::EVENT_TRIPPED:
            \YourMonitoringSystem::warn(
                "Ganesha has tripped! It seems that a failure has occurred in {$service}. {$message}."
            );
            break;
        case Ganesha::EVENT_CALMED_DOWN:
            \YourMonitoringSystem::info(
                "The failure in {$service} seems to have calmed down :). {$message}."
            );
            break;
        case Ganesha::EVENT_STORAGE_ERROR:
            \YourMonitoringSystem::error($message);
            break;
        default:
            break;
    }
});
```

#### Disable

Ganesha continue to record success/failure statistics, but Ganesha does not trip even if failure count reached to threshold.

```php
// Ganesha with Count strategy(threshold `3`).
// $ganesha = ...

// Disable
Ackintosh\Ganesha::disable();

// Although the failure is recorded to storage,
$ganesha->failure($service);
$ganesha->failure($service);
$ganesha->failure($service);

// Ganesha does not trip and Ganesha::isAvailable() returns true.
var_dump($ganesha->isAvailable($service);
// bool(true)
```

#### Reset


```php
$ganesha = Ackintosh\Ganesha\Builder::build([
	// ...
]);

$ganesha->reset();

```

## Strategies

Ganesha has two strategies which avoids cascading failures.

### Rate (default)

```php
$ganesha = Ackintosh\Ganesha\Builder::build([
    'timeWindow'            => 30,
    'failureRateThreshold'  => 50,
    'minimumRequests'       => 10,
    'intervalToHalfOpen'    => 5,
    'adapter'               => new Ackintosh\Ganesha\Storage\Adapter\Memcached($memcached),
]);
```

### Count

If you want use Count strategy, use `Builder::buildWithCountStrategy()`.

```php
$ganesha = Ackintosh\Ganesha\Builder::buildWithCountStrategy([
    'failureCountThreshold' => 100,
    'intervalToHalfOpen'    => 5,
    'adapter'               => new Ackintosh\Ganesha\Storage\Adapter\Memcached($memcached),
]);
```

## Adapters

### Redis

Redis adapter requires [phpredis](https://github.com/phpredis/phpredis). So if you don't have it, run `pecl install redis`.

```php
$redis = new \Redis();
$redis->connect('localhost');
$adapter = new Ackintosh\Ganesha\Storage\Adapter\Redis($redis);

$ganesha = Ackintosh\Ganesha\Builder::build([
    'adapter' => $adapter,
]);
```

### Memcached

Memcached adapter requires [memcached](https://github.com/php-memcached-dev/php-memcached/) (NOT memcache) extension.

```php
$memcached = new \Memcached();
$memcached->addServer('localhost', 11211);
$adapter = new Ackintosh\Ganesha\Storage\Adapter\Memcached($memcached);

$ganesha = Ackintosh\Ganesha\Builder::build([
    'adapter' => $adapter,
]);
```

## Ganesha :heart: Guzzle

If you using [Guzzle](https://github.com/guzzle/guzzle) (v6 or higher), [Guzzle Middleware](http://docs.guzzlephp.org/en/stable/handlers-and-middleware.html) powered by Ganesha makes it easy to integrate Circuit Breaker to your existing code base.

```php
use Ackintosh\Ganesha\Builder;
use Ackintosh\Ganesha\GuzzleMiddleware;
use Ackintosh\Ganesha\Exception\RejectedException;
use GuzzleHttp\Client;
use GuzzleHttp\HandlerStack;

$ganesha = Builder::build([
    'timeWindow'           => 30,
    'failureRateThreshold' => 50,
    'minimumRequests'      => 10,
    'intervalToHalfOpen'   => 5,
    'adapter'              => $adapter,
]);
$middleware = new GuzzleMiddleware($ganesha);

$handlers = HandlerStack::create();
$handlers->push($middleware);

$client = new Client(['handler' => $handlers]);

try {
    $client->get('http://api.example.com/awesome_resource');
} catch (RejectedException $e) {
    // If the circuit breaker is open, RejectedException will be thrown.
}
```

### How does Guzzle Middleware determine the `$service`?

As documented in [Usage](https://github.com/ackintosh/ganesha#usage), Ganesha detects failures for each `$service`. Below, We will show you how Guzzle Middleware determine `$service` and how we specify `$service` explicitly.

By default, the host name is used as `$service`.


```php
// In the example above, `api.example.com` is used as `$service`.
$client->get('http://api.example.com/awesome_resource');
```

You can also specify `$service` via a option passed to client, or request header. If both are specified, the option value takes precedence.

```php
// via constructor argument
$client = new Client([
    'handler' => $handlers,
    // 'ganesha.service_name' is defined as ServiceNameExtractor::OPTION_KEY
    'ganesha.service_name' => 'specified_service_name',
]);

// via request method argument
$client->get(
    'http://api.example.com/awesome_resource',
    [
        'ganesha.service_name' => 'specified_service_name',
    ]
);

// via request header
$request = new Request(
    'GET',
    'http://api.example.com/awesome_resource',
    [
        // 'X-Ganesha-Service-Name' is defined as ServiceNameExtractor::HEADER_NAME
        'X-Ganesha-Service-Name' => 'specified_service_name'
    ]
);
$client->send($request);
```

Alternatively, you can apply your own rules by implementing a class that implements the `ServiceNameExtractorInterface`.

```php
use Ackintosh\Ganesha\GuzzleMiddleware\ServiceNameExtractorInterface;
use Psr\Http\Message\RequestInterface;

class SampleExtractor implements ServiceNameExtractorInterface
{
    /**
     * @override
     */
    public function extract(RequestInterface $request, array $requestOptions)
    {
        // We treat the combination of host name and HTTP method name as $service.
        return $request->getUri()->getHost() . '_' . $request->getMethod();
    }
}

// ---

$ganesha = Builder::build([
    // ...
]);
$middleware = new GuzzleMiddleware(
    $ganesha,
    // Pass the extractor as an argument of GuzzleMiddleware constructor.
    new SampleExtractor()
);
```

## Ganesha :heart: Swagger Codegen

PHP client generated by [Swagger Codegen](https://github.com/swagger-api/swagger-codegen) (v2.3~) is using Guzzle as HTTP client and as we mentioned as [Ganesha :heart: Guzzle](https://github.com/ackintosh/ganesha#ganesha-heart-guzzle), Guzzle Middleware powered by Ganesha is ready. So it is possible to integrate Ganesha and the PHP client in a smart way as below.

```php
// For details on how to build middleware please see https://github.com/ackintosh/ganesha#ganesha-heart-guzzle
$middleware = new GuzzleMiddleware($ganesha);

// Set the middleware to HTTP client.
$handlers = HandlerStack::create();
$handlers->push($middleware);
$client = new Client(['handler' => $handlers]);

// Just pass the HTTP client to the constructor of API class.
$api = new PetApi($client);

try {
    // Ganesha is working in the shadows! The result of api call is monitored by Ganesha.
    $api->getPetById(123);
} catch (RejectedException $e) {
    awesomeErrorHandling($e);
}
```

## Run tests

We can run unit tests on Docker container, so it is not necessary to install dependencies in your machine.

```bash
# Start redis, memcached server
$ docker-compose up 

# Run tests in container
$ docker-compose run --rm -w /tmp/ganesha -u ganesha client vendor/bin/phpunit
```

## Build PR site with [Soushi](https://github.com/kentaro/soushi)

https://ackintosh.github.io/ganesha/

```bash
$ docker-compose run --rm client soushi build docs

# The site will be generate into `docs` directory and let's confirm your changes.
$ open docs/index.html
```

## Requirements

Ganesha supports PHP 5.6 or higher.
