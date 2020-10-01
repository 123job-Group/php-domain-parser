# PHP Domain Parser
**PHP Domain Parser** is a [Public Suffix List](http://publicsuffix.org/) based domain parser implemented in PHP.

![Quality Assurance](https://github.com/jeremykendall/php-domain-parser/workflows/Quality%20Assurance/badge.svg)
[![Total Downloads][ico-packagist]][link-packagist]
[![Latest Stable Version][ico-release]][link-release]
[![Software License][ico-license]][link-license]

## Motivation

While there are plenty of excellent URL parsers and builders available, there
are very few projects that can accurately parse a domaine into its component
subdomain, registrable domain, and public suffix parts.

Consider the domain www.pref.okinawa.jp.  In this domain, the
*public suffix* portion is **okinawa.jp**, the *registrable domain* is
**pref.okinawa.jp**, and the *subdomain* is **www**. You can't regex that.

PHP Domain Parser is compliant around:

- accurate Public Suffix List based parsing.
- accurate Root Zone Database parsing.

## Installation

### Composer

~~~
$ composer require jeremykendall/php-domain-parser
~~~

### System Requirements

You need:

- **PHP >= 7.4** but the latest stable version of PHP is recommended
- the `intl` extension

## Usage

### Public Suffix List Resolution

The first objective of the library is to use the [Public Suffix List](http://publicsuffix.org/) to easily resolve a 
domain as a `Pdp\ResolvedDomain` object using the following methods:

~~~php
<?php 
use Pdp\Rules;

$rules = Rules::fromPath('/path/to/cache/public-suffix-list.dat');

$resolvedDomain = $rules->resolve('www.PreF.OkiNawA.jP');
echo $resolvedDomain->getDomain();            //display 'www.pref.okinawa.jp';
echo $resolvedDomain->getPublicSuffix();      //display 'okinawa.jp';
echo $resolvedDomain->getSecondLevelDomain(); //display 'pref';
echo $resolvedDomain->getRegistrableDomain(); //display 'pref.okinawa.jp';
echo $resolvedDomain->getSubDomain();         //display 'www';
~~~

In case of an error an exception which extends `Pdp\ExceptionInterface` is thrown.

### Top Level Domains resolution

While the [Public Suffix List](http://publicsuffix.org/) is a community based list, the package provides access to 
the Top Level domain information given by the [IANA website](https://data.iana.org/TLD/tlds-alpha-by-domain.txt) to always resolve
top domain against all registered TLD even the new ones.

~~~php
use Pdp\TopLevelDomains;

$iana = TopLevelDomains::fromPath('/path/to/cache/tlds-alpha-by-domain.txt');

$resolvedDomain = $iana->resolve('www.PreF.OkiNawA.jP');
echo $resolvedDomain->getDomain();            //display 'www.pref.okinawa.jp';
echo $resolvedDomain->getPublicSuffix();      //display 'jp';
echo $resolvedDomain->getSecondLevelDomain(); //display 'okinawa';
echo $resolvedDomain->getRegistrableDomain(); //display 'okinawa.jp';
echo $resolvedDomain->getSubDomain();         //display 'www.pref';
~~~

In case of an error an exception which extends `Pdp\ExceptionInterface` is thrown.

**WARNING:**

**You should never use the library this way in production, without, at least, a caching mechanism to reduce HTTP calls.**

**Using the PSL to determine what is a valid domain name and what isn't is dangerous, particularly in these days when new gTLDs are arriving at a rapid pace.  
The IANA Root Zone Database is the proper source for this information.  
Either way, if you must use this library for this purpose, please consider integrating an automatic update mechanism into your software.**

## Managing the package databases

The library comes bundle with a **optional** service which enables resolving domain name without the constant network overhead of continuously downloading the remote databases.
The `Pdp\Storage\PsrStorageFactory` enables returning storage instances that retrieve, convert and cache the Public Suffix List as well as the IANA Root Zone Database.

### Instantiate `Pdp\Storage\PsrStorageFactory`

To work as intended, the `Pdp\Storage\PsrStorageFactory` constructor requires:

- a [PSR-16](http://www.php-fig.org/psr/psr-16/) A Cache object to store the rules locally.
- a [PSR-18](http://www.php-fig.org/psr/psr-18/) A PSR-18 HTTP Client.
- a [PSR-3](http://www.php-fig.org/psr/psr-3/) A Logger object to log storage usage.

When creating a new storage instance you will require:

- a `$cachePrefix` argument to optionally add a prefix to your cache index, default to the empty string `'''`;
- a `$ttl` argument if you need to set the default `$ttl`, default to `null` to use the underlying caching default TTL;

The `$ttl` argument can be:

- an `int` representing time in second (see PSR-16);
- a `DateInterval` object (see PSR-16);
- a `DateTimeInterface` object representing the date and time when the item should expire;

However, the package no longer provides any implementation of such interfaces are
they are many robust implementations that can easily be found on packagist.org. 

#### Refreshing the cached PSL and RZD data

**THIS IS THE RECOMMENDED WAY OF USING THE LIBRARY**

For the purpose of this example we used:
 
- Guzzle as a PSR-18 implementation HTTP client
- The Symfony Cache Component to use a PSR-16 cache implementation

You could easily use other packages as long as they implement the required PSR interfaces. 

~~~php
<?php 

use GuzzleHttp\Client;
use GuzzleHttp\Psr7\Request;
use Pdp\Storage\PsrStorageFactory;
use Psr\Http\Message\RequestFactoryInterface;
use Psr\Http\Message\RequestInterface;
use Symfony\Component\Cache\Adapter\FilesystemAdapter;
use Symfony\Component\Cache\Psr16Cache;

$cache = new Psr16Cache(new FilesystemAdapter('pdp', 3600, __DIR__.'/data'));
$requestFactory = new class implements RequestFactoryInterface {
    public function createRequest(string $method, $uri): RequestInterface
    {
        return new Request($method, $uri);
    }
};

$cachePrefix = 'pdp_';
$cacheTtl = new DateInterval('P1D');
$factory = new PsrStorageFactory(new Client(), $requestFactory, $cache);
$pslStorage = $factory->createPublicSuffixListStorage($cachePrefix, $cacheTtl);
$rzdStorage = $factory->createRootZoneDatabaseStorage($cachePrefix, $cacheTtl);

$rules = $pslStorage->get(PsrStorageFactory::PSL_URL);
$tldDomains = $rzdStorage->get(PsrStorageFactory::RZD_URL);
~~~

### Automatic Updates

It is important to always have an up to date Public Suffix List and Root Zone Database.
This library no longer provide an out of the box script to do so as implementing such a job heavily depends on your application setup. 

Changelog
-------

Please see [CHANGELOG](CHANGELOG.md) for more information about what has been changed since version **5.0.0** was released.

Contributing
-------

Contributions are welcome and will be fully credited. Please see [CONTRIBUTING](.github/CONTRIBUTING.md) for details.

Testing
-------

`pdp-domain-parser` has:

- a [PHPUnit](https://phpunit.de) test suite
- a coding style compliance test suite using [PHP CS Fixer](http://cs.sensiolabs.org/).
- a code analysis compliance test suite using [PHPStan](https://github.com/phpstan/phpstan).

To run the tests, run the following command from the project folder.

``` bash
$ composer test
```

Security
-------

If you discover any security related issues, please email nyamsprod@gmail.com instead of using the issue tracker.

Credits
-------

- [Jeremy Kendall](https://github.com/jeremykendall)
- [Ignace Nyamagana Butera](https://github.com/nyamsprod)
- [All Contributors](https://github.com/jeremykendall/php-domain-parser/contributors)

License
-------

The MIT License (MIT). Please see [License File](LICENSE) for more information.

Attribution
-------

Portions of the `Pdp\Converter` and `Pdp\Rules` are derivative works of the PHP
[registered-domain-libs](https://github.com/usrflo/registered-domain-libs).
Those parts of this codebase are heavily commented, and I've included a copy of
the Apache Software Foundation License 2.0 in this project.

[ico-packagist]: https://img.shields.io/packagist/dt/jeremykendall/php-domain-parser.svg?style=flat-square
[ico-release]: https://img.shields.io/github/release/jeremykendall/php-domain-parser.svg?style=flat-square
[ico-license]: https://img.shields.io/badge/license-MIT-brightgreen.svg?style=flat-square

[link-packagist]: https://packagist.org/packages/jeremykendall/php-domain-parser
[link-release]: https://github.com/jeremykendall/php-domain-parser/releases
[link-license]: https://github.com/jeremykendall/php-domain-parser/blob/master/LICENSE
