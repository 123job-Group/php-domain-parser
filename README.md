# PHP Domain Parser

**PHP Domain Parser** is domain parser implemented in PHP.

![Quality Assurance](https://github.com/jeremykendall/php-domain-parser/workflows/Quality%20Assurance/badge.svg)
[![Total Downloads][ico-packagist]][link-packagist]
[![Latest Stable Version][ico-release]][link-release]
[![Software License][ico-license]][link-license]

## Motivation

While there are plenty of excellent URL parsers and builders available, there
are very few projects that can accurately parse a domain into its component
subdomain, registrable domain, second level domain and public suffix parts.

Consider the domain www.pref.okinawa.jp.  In this domain, the
*public suffix* portion is **okinawa.jp**, the *registrable domain* is
**pref.okinawa.jp**, the *subdomain* is **www** and 
the *second level domain* is **pref**.  
You can't regex that.

PHP Domain Parser is compliant around:

- accurate Public Suffix List based parsing.
- accurate Root Zone Database parsing.

## Installation

### Composer

~~~
$ composer require jeremykendall/php-domain-parser:^6.0
~~~

### System Requirements

You need:

- **PHP >= 7.4** but the latest stable version of PHP is recommended
- the `intl` extension

## Usage

**If you are upgrading from version 5 please check the [upgrading guide](UPGRADING.md) for known issues.**

### Resolving Domains

This library can resolve a domain against:
 
- The [Public Suffix List](http://publicsuffix.org/)
- The [IANA Root Zone Database](https://data.iana.org/TLD/tlds-alpha-by-domain.txt)

In both cases this is done using the `resolve` method implemented on the resource 
instance. The method returns a `Pdp\ResolvedDomain` object which represents the 
result of that process.

For the [Public Suffix List](http://publicsuffix.org/) you need to use the
`Pdp\Rules` class as shown below:

~~~php
<?php 
use Pdp\Rules;

$publicSuffixList = Rules::fromPath('/path/to/cache/public-suffix-list.dat');

$result = $publicSuffixList->resolve('www.PreF.OkiNawA.jP');
echo $result->domain()->toString();            //display 'www.pref.okinawa.jp';
echo $result->subDomain()->toString();         //display 'www';
echo $result->secondLevelDomain();             //display 'pref';
echo $result->registrableDomain()->toString(); //display 'pref.okinawa.jp';
echo $result->publicSuffix()->toString();      //display 'okinawa.jp';
$result->publicSuffix()->isICANN();            //return true;
~~~

For the [IANA Root Zone Database](https://data.iana.org/TLD/tlds-alpha-by-domain.txt),
the `Pdp\TopLevelDomains` class is use instead:

~~~php
use Pdp\TopLevelDomains;

$rootZoneDatabase = TopLevelDomains::fromPath('/path/to/cache/tlds-alpha-by-domain.txt');

$result = $rootZoneDatabase->resolve('www.PreF.OkiNawA.jP');
echo $result->domain()->toString();            //display 'www.pref.okinawa.jp';
echo $result->publicSuffix()->toString();      //display 'jp';
echo $result->secondLevelDomain();             //display 'okinawa';
echo $result->registrableDomain()->toString(); //display 'okinawa.jp';
echo $result->subDomain()->toString();         //display 'www.pref';
echo $result->publicSuffix()->isIANA();        //return true
~~~

In case of an error an exception which extends `Pdp\CannotProcessHost` is thrown.

The `resolve` method will always return a `ResolvedDomain` even if the domain
syntax is invalid or if there is no match found in the resource data. 
To work around this limitation, the library exposes more strict methods,
namely:

- `Rules::getCookieDomain`
- `Rules::getICANNDomain`
- `Rules::getPrivateDomain`

for the Public Suffix List and the following method for the Root Zone
Database:

- `TopLevelDomains::getTopLevelDomain`

These methods resolve the domain against their respective data source using
the same rules as the `resolve` method but will instead throw an exception 
if no valid effective TLD is found or if the submitted domain is invalid.

~~~php
<?php 
use Pdp\Rules;
use Pdp\TopLevelDomains;

$publicSuffixList = Rules::fromPath('/path/to/cache/public-suffix-list.dat');

$publicSuffixList->getICANNDomain('qfdsf.unknownTLD');
// will throw because `.unknownTLD` is not part of the ICANN section

$result = $publicSuffixList->getCookieDomain('qfdsf.unknownTLD');
$result->publicSuffix()->value();   // returns 'unknownTLD'
$result->publicSuffix()->isKnown(); // returns false
// will not throw because the domain syntax is correct.

$publicSuffixList->getCookieDomain('com');
// will not throw because the domain syntax is invalid (ie: does not support public suffix)

$result = $publicSuffixList->resolve('com');
$result->publicSuffix()->value();   // returns null
$result->publicSuffix()->isKnown(); // returns false
// will not throw but its public suffix value equal to NULL

$rootZoneDatabase = TopLevelDomains::fromPath('/path/to/cache/public-suffix-list.dat');
$rootZoneDatabase->getTopLevelDomain('com');
// will not throw because the domain syntax is invalid (ie: does not support public suffix)
~~~

To instantiate each domain resolver you can use the following named constructor:

- `fromString`: instantiate the resolver from a inline string representing the data source;
- `fromJsonString`: instantiate the resolver from a json string representing the data source;
- `fromPath`: instantiate the resolver from a local path or online URL by relying on `fopen`;

**If the instantiation does not work an exception will be thrown.**

Once instantiated, you can always convert the data source in a Json Encoded friendly schema
if needed.

**WARNING:**

**You should never use resolve domain name this way in production, without, at 
least, a caching mechanism to reduce PSL downloads.**

**Using the Public Suffix List to determine what is a valid domain name and what 
isn't is dangerous, particularly in these days when new gTLDs are arriving at a 
rapid pace.**

**If you are looking to know the validity of a Top Level Domain, the 
IANA Root Zone Database is the proper source for this information or alternatively
consider using directly the DNS.** 

**If you must use this library for any of the above purposes, please consider 
integrating an updating mechanism into your software.**

**For more information go to the [Managing external data source section](#managing-the-package-external-resources)** 

### Resolved domain information.

Whichever methods chosen to resolve the domain on success, the package will
return a `Pdp\ResolvedDomain` instance.

The `Pdp\ResolvedDomain` decorates the `Pdp\Domain` class resolved  but also 
gives access as separate methods to the domain different components. 

~~~php
use Pdp\RootZoneDatabase;

/** @var RootZoneDatabase $rootZoneDatabase */
$result = $rootZoneDatabase->resolve('www.PreF.OkiNawA.jP');
echo $result->domain()->toString();            //display 'www.pref.okinawa.jp';
echo $result->publicSuffix()->toString();      //display 'jp';
echo $result->secondLevelDomain();             //display 'okinawa';
echo $result->registrableDomain()->toString(); //display 'okinawa.jp';
echo $result->subDomain()->toString();         //display 'www.pref';
echo $result->publicSuffix()->isIANA();        //return true
~~~
 
You can modify the returned `Pdp\ResolvedDomain` instance using the following methods:

~~~php
<?php 

use Pdp\PublicSuffixList;

/** @var PublicSuffixList $publicSuffixList */
$result = $publicSuffixList->resolve('shop.example.com');
$altResult = $result
    ->withSubDomain('foo.bar')
    ->withSecondLevelDomain('test')
    ->withPublicSuffix('example');

echo $result->domain()->toString(); //display 'shop.example.com';
$result->publicSuffix()->isKnown(); //returns true;

echo $altResult->domain()->toString(); //display 'foo.bar.test.example';
$altResult->publicSuffix()->isKnown(); //returns false;
~~~

**TIP: Always favor submitting a `Pdp\PublicSuffix` object rather that any other
supported type to avoid unexpected results. By default, if the input is not a
`Pdp\PublicSuffix` instance, the resulting public suffix will be labelled as
being unknown. For more information go to the [Public Suffix section](#public-suffix)**

### Public Suffix

The domain effective TLD is represented using the `Pdp\PublicSuffix`. Depending
 on the data source the object exposes different information regarding its
origin.

~~~php
<?php 
use Pdp\PublicSuffixList;

/** @var PublicSuffixList $publicSuffixList */
$publicSuffix = $publicSuffixList->resolve('example.github.io')->publicSuffix();

echo $publicSuffix->domain()->toString(); //display 'github.io';
$publicSuffix->isICANN();                 //will return false
$publicSuffix->isPrivate();               //will return true
$publicSuffix->isIANA();                  //will return false
$publicSuffix->isKnown();                 //will return true
~~~

The public suffix state depends on its origin:
 
- `isKnown` returns `true` if the value is present in the data resource.
- `isIANA` returns `true` if the value is present in the Root Zone Database.
- `isICANN` returns `true` if the value is present in the PSL ICANN section.
- `isPrivate` returns `true` if the value is present in the PSL private section.
 
The same information is used when `Pdp\PublicSuffix` object is 
instantiate via its named constructors:
 
 ~~~php
 <?php 
 use Pdp\PublicSuffix;

$iana = PublicSuffix::fromIANA('ac.be');
$icann = PublicSuffix::fromICANN('ac.be');
$private = PublicSuffix::fromPrivate('ac.be');
$unknown = PublicSuffix::fromUnknown('ac.be');
~~~

Using a `PublicSuffix` object instead of a string or `null` with 
`ResolvedDomain::withPublicSuffix` will ensure that the returned value will
always contain the correct information regarding the public suffix resolution.
 
Using a `Domain` object instead of a string or `null` with the named 
constructor ensure a better instantiation of the Public Suffix object for
more information go to the [ASCII and Unicode format section](#ascii-and-unicode-formats) 
 
### Accessing and processing Domain labels

If you are interested into manipulating the domain labels without taking into 
account the Effective TLD, the library provides a `Domain` object tailored for
manipulating domain labels. You can access the object using the following methods:
 
- the `ResolvedDomain::domain` method 
- the `PublicSuffix::domain` method
- from `ResolvedDomain::subDomain` method
- the `ResolvedDomain::registrableDomain` returns a `ResolvedDomain`

`Domain` objects usage are explain in the next section.

~~~php
<?php 
use Pdp\PublicSuffixList;

/** @var PublicSuffixList $publicSuffixList */
$result = $publicSuffixList->resolve('www.bbc.co.uk');
$domain = $result->domain();
echo $domain->toString(); // display 'www.bbc.co.uk'
count($domain);           // returns 4
$domain->labels();        // returns ['uk', 'co', 'bbc', 'www'];
$domain->label(-1);       // returns 'www'
$domain->label(0);        // returns 'uk'
foreach ($domain as $label) {
   echo $label, PHP_EOL;
}
// display 
// uk
// co
// bbc
// www

$publicSuffixDomain = $result->publicSuffix()->domain();
$publicSuffixDomain->labels(); // returns ['uk', 'co']
~~~ 

You can also add or remove labels according to their key index using the 
following methods:

~~~php
<?php 
use Pdp\PublicSuffixList;

/** @var PublicSuffixList $publicSuffixList */
$domain = $publicSuffixList->resolve('www.ExAmpLE.cOM')->domain();

$newDomain = $domain
    ->withLabel(1, 'com')  //replace 'example' by 'com'
    ->withoutLabel(0, -1)  //remove the first and last labels
    ->append('www')
    ->prepend('docs.example');

echo $domain->toString();    // display 'www.example.com'
echo $newDomain->toString(); // display 'docs.example.com.www'
~~~ 

**WARNING: Because of its definition, a domain name can be `null` or a string.**

To distinguish this possibility the object exposes two (2) formatting methods 
`Domain::value` which can be `null` or a `string` and `Domain::toString` which 
will always cast the domain value to a string.

 ~~~php
use Pdp\Domain;
 
$nullDomain = Domain::fromIDNA2008(null);
$nullDomain->value();    // returns null;
$nullDomain->toString(); // returns '';
 
$emptyDomain = Domain::fromIDNA2008('');
$emptyDomain->value();    // returns '';
$emptyDomain->toString(); // returns '';
 ~~~ 

### ASCII and Unicode formats.

Domain names originally only supported ASCII characters. Nowadays,
they can also be presented under a UNICODE representation. The conversion
between both formats is done using the compliant implementation of 
[UTS#46](https://www.unicode.org/reports/tr46/), otherwise known as Unicode 
IDNA Compatibility Processing. Domain objects expose a `toAscii` and a 
`toUnicode` methods which returns a new instance in the converted format.

~~~php
<?php 
use Pdp\PublicSuffixList;

/** @var PublicSuffixList $publicSuffixList */
$unicodeDomain = $publicSuffixList->resolve('bébé.be')->domain();
echo $unicodeDomain->toString(); // returns 'bébé.be'

$asciiDomain = $publicSuffixList->resolve('xn--bb-bjab.be')->domain();
echo $asciiDomain->toString();  // returns 'xn--bb-bjab.be'

$asciiDomain->toUnicode()->toString() === $unicodeDomain->toString(); //returns true
$unicodeDomain->toAscii()->toString() === $asciiDomain->toString();   //returns true
~~~

By default, the library uses IDNA2008 algorithm to convert domain name between 
both formats. It is still possible to use the legacy conversion algorithm known
as IDNA2003.

Since direct conversion between both algorithms is not possible you need 
to explicitly specific on construction which algorithm you will use
when creating a new domain instance via the `Pdp\Domain` object. This 
is done via two (2) named constructors:

- `Pdp\Domain::fromIDNA2008`
- `Pdp\Domain::fromIDNA2003`

At any given moment the `Pdp\Domain` instance can tell you whether:

- the algorithm used for converting the domain is `IDNA2008`;
- if the domain value is in `ASCII` mode or not;

~~~php
use Pdp\Domain;

$domain = Domain::fromIDNA2008('faß.de');
echo $domain->value(); // display 'faß.de'
$domain->isIdna2008(); // returns true
$domain->isAscii();    // return false

$asciiDomain = $domain->toAscii(); 
echo $asciiDomain->value(); // display 'xn--fa-hia.de'
$asciiDomain->isIdna2008(); // returns true
$asciiDomain->isAscii();    // returns true

$domain = Domain::fromIDNA2003('faß.de');
echo $domain->value(); // display 'fass.de'
$domain->isIdna2008(); // returns false
$domain->isAscii();    // returns true

$asciiDomain = $domain->toAscii();
echo $asciiDomain->value(); // display 'fass.de'
$asciiDomain->isIdna2008(); // returns false
$asciiDomain->isAscii();    // returns true
~~~

**TIP: Always favor submitting a `Pdp\Domain` object for resolution rather that a 
string or an object that can be cast to a string to avoid unexpected format 
conversion errors/results. By default, and with lack of information conversion
is done using IDNA 2008 rules.**

### Managing the package external resources

Depending on your application, the mechanism to store your resources may differ, 
nevertheless, the library comes bundle with a **optional service** which 
enables resolving domain name without the constant network overhead of 
continuously downloading the remote databases.

The interfaces defined under the `Pdp\Storage` namespace enable 
integrating a database managing system and provide an implementation example 
using PHP-FIG PSR interfaces.

The `Pdp\Storage\PsrStorageFactory` enables returning storage instances that
retrieve, convert and cache the Public Suffix List and the IANA Root 
Zone Database using standard interfaces published by the PHP-FIG.

#### Instantiate `Pdp\Storage\PsrStorageFactory`

To work as intended, the `Pdp\Storage\PsrStorageFactory` constructor requires:

- a [PSR-16](http://www.php-fig.org/psr/psr-16/) Simple Cache implementing library.
- a [PSR-17](http://www.php-fig.org/psr/psr-17/) HTTP Factory implementing library.
- a [PSR-18](http://www.php-fig.org/psr/psr-18/) HTTP Client implementing library.

When creating a new storage instance you will require:

- a `$cachePrefix` argument to optionally add a prefix to your cache index, 
default to the empty string;
- a `$ttl` argument if you need to set the default `$ttl`, default to `null` 
to use the underlying caching default TTL;

The `$ttl` argument can be:

- an `int` representing time in second (see PSR-16);
- a `DateInterval` object (see PSR-16);
- a `DateTimeInterface` object representing the date and time when the item 
will expire;

The package does not provide any implementation of such interfaces as you can
find robust and battle tested implementations on packagist.

#### Refreshing the cached PSL and RZD data

**THIS IS THE RECOMMENDED WAY OF USING THE LIBRARY**

For the purpose of this example we will use our PSR powered solution with:
 
- *Guzzle HTTP Client* as our PSR-18 HTTP client;
- *Guzzle PSR-7 package* which provide factories to create a PSR-7 objects using PSR-17 interfaces;
- *Symfony Cache Component* as our PSR-16 cache implementation provider;

We will cache both external sources for 24 hours in a PostgreSQL database.

You are free to use other libraries/solutions/settings as long as they 
implement the required PSR interfaces. 

~~~php
<?php 

use GuzzleHttp\Psr7\Request;
use Pdp\Storage\PsrStorageFactory;
use Psr\Http\Message\RequestFactoryInterface;
use Psr\Http\Message\RequestInterface;
use Symfony\Component\Cache\Adapter\PdoAdapter;
use Symfony\Component\Cache\Psr16Cache;

$pdo = new PDO(
    'pgsql:host=localhost;port:5432;dbname=testdb', 
    'user', 
    'password', 
    [PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION]
);
$cache = new Psr16Cache(new PdoAdapter($pdo, 'pdp', 43200));
$client = new GuzzleHttp\Client();
$requestFactory = new class implements RequestFactoryInterface {
    public function createRequest(string $method, $uri): RequestInterface
    {
        return new Request($method, $uri);
    }
};

$cachePrefix = 'pdp_';
$cacheTtl = new DateInterval('P1D');
$factory = new PsrStorageFactory($cache, $client, $requestFactory);
$pslStorage = $factory->createPublicSuffixListStorage($cachePrefix, $cacheTtl);
$rzdStorage = $factory->createRootZoneDatabaseStorage($cachePrefix, $cacheTtl);

// if you need to force refreshing the rules 
// before calling them (to use in a refresh script)
// uncomment this part or adapt it to you script logic
// $pslStorage->delete(PsrStorageFactory::PUBLIC_SUFFIX_LIST_URI);
$publicSuffixList = $pslStorage->get(PsrStorageFactory::PUBLIC_SUFFIX_LIST_URI);

// if you need to force refreshing the rules 
// before calling them (to use in a refresh script)
// uncomment this part or adapt it to you script logic
// $rzdStorage->delete(PsrStorageFactory::ROOT_ZONE_DATABASE_URI);
$rootZoneDatabase = $rzdStorage->get(PsrStorageFactory::ROOT_ZONE_DATABASE_URI);
~~~

**Be sure to adapt the following code to your own application.
The following code is an example given without warranty of it working 
out of the box.**

**You should use your dependency injection container to avoid repeating this
code in your application.**

### Automatic Updates

It is important to always have an up to date Public Suffix List and Root Zone
Database.  
This library no longer provide an out of the box script to do so as implementing
such a job heavily depends on your application setup.
You can use the above example script as a starting point to implement such a job.

Changelog
-------

Please see [CHANGELOG](CHANGELOG.md) for more information about what has been
changed since version **5.0.0** was released.

Contributing
-------

Contributions are welcome and will be fully credited. Please see 
[CONTRIBUTING](.github/CONTRIBUTING.md) for details.

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

If you discover any security related issues, please email nyamsprod@gmail.com
instead of using the issue tracker.

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
