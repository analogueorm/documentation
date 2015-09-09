Add this line to your project's composer.json file : 

```
"Analogue/ORM": "~2.1"
```

If you're using Laravel 4, there is a specific branch for it.

Laravel 4 : 
```
"Analogue/ORM": "~1.*"
```
> Important : The L4 Branch is currently not up-to-date, improvement/bugfixes will be merged back as soon as the internal refactoring happening on dev branch is done.

Then run : 

```
composer update
```

##Usage

Although Analogue is built on top of Laravel components, the package itself is framework-agnostic and can be used on its own.

```php

use Analogue\ORM\Analogue;

require 'vendor/autoload.php';

$connection = [
    'driver'    => 'mysql',
    'host'      => 'localhost',
    'database'  => 'database',
    'username'  => 'root',
    'password'  => 'password',
    'charset'   => 'utf8',
    'collation' => 'utf8_unicode_ci',
    'prefix'    => '',
];

$analogue = new Analogue($connection);

```
This will initialize the default connection for Analogue.

You can use multiple database connections, by specifing a connection string. For example, you can add a sqlite database.

```php

$sqlite = [
    'driver'   => 'sqlite',
    'database' => __DIR__.'/database.sqlite',
    'prefix'   => '',
];

$analogue->addConnection($sqlite, 'sqlite');

```

## Laravel 4/5 Integration

If you're using Laravel, Analogue will naturally use your application's database configuration. Just add the Service Provider to config/app.php :

```
'Analogue\ORM\AnalogueServiceProvider',
```

If you're using facades, add this line to the aliases :

```
'Analogue' => 'Analogue\ORM\AnalogueFacade',
```

You can access the Analogue object from the IoC Container :

```php

app('analogue'); // L5

$app->make('analogue'); // L4 | L5

```

### Lumen 

To add Analogue support to Lumen, register the service provider in `bootstrap/app.php` :

```php
    
$app->register('Analogue\ORM\AnalogueServiceProvider');

```

### Authentication Driver

Authentication driver for Laravel is provided in a [separate package](https://github.com/analogueorm/laravel-auth).

## Silex

This [project](https://github.com/anthonysterling/silex-provider-analogue-orm) has a Service Provider that integrates analogue into the Silex framework.


