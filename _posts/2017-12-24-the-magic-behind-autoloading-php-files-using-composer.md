---
layout: post
title: "The magic behind autoloading php files using Composer"
description: "Do you still remember how we used load PHP Classes using require_once? Learn how frameworks like 
Symfony dynamically resolve Namespaces."
tagline: Best PHP Dependency Manager
tags: [php, composer, namespaces, autoloading, require_once]
date: 2017-12-25 21:11
image: composer_autoloading.jpg
---

#### Foreword

It's been several years I didn't start a project from scratch. I mean totally from scratch. All my projects are by 
default fresh Symfony installations. 2 weeks ago I was recording one of the lectures of this course and when I was 
preparing a super simple example I run into a `Class not found` error. This reminded me the good old days
 of using 
PHP `require_once` method so I decided to write a short article about the way Composer is making our life 
easier and how it actually autoloads files behind the scenes.

Thank you [Nils](https://twitter.com/naderman) and [Jordi](https://twitter.com/seldaek) for the awesome dependency 
manager and happy birthday Jordi!

#### Base line

Don't overthink what a **Namespace** is. 

**Namespace** is basically just a **Class** prefix (like directory in Operating System) to 
ensure the Class path uniqueness. 

Also just to make things clear, the **use** statement is not doing anything 
only aliasing your **Namespaces** so you can use shortcuts or include **Classes** with the same name but different 
**Namespace** in the same file.

E.g:

```php
// You can do this at the top of your Class
use Symfony\Component\Debug\Debug;

if ($_SERVER['APP_DEBUG']) {
    // So you can utilize the Debug class it in an elegant way
    Debug::enable();
    // Instead of this ugly one
    // \Symfony\Component\Debug\Debug::enable();
}
```

#### Problem

I want to call a public method of a Class encapsulated in a specific Namespace from a different directory.

```php
<?php

// /ComposerAutoloading/src/Christmas/Santa.php

namespace Christmas;

class Santa
{
    /**
     * @return void
     */
    public function sayIt(): void
    {
        echo "Merry Christmas && Write HQ code!";
    }
}
```

```php
<?php

// /ComposerAutoloading/src/index.php

$santa = new \Christmas\Santa();
$santa->sayIt();
```

```bash
Fatal error:  Uncaught Error: Class 'Santa' not found
```

#### Old fashion solution

[![Old fashion James Bond movie](http://img.youtube.com/vi/ceTR67EjeyU/0.jpg)](http://www.youtube
.com/watch?v=ceTR67EjeyU)
(click to play the movie scene)

```bash
php index.php

Fatal error:  Uncaught Error: Class 'Santa' not found
```

So, to be able to initialize the Santa Class, you have to import it to your **Global Namespace** of **index.php** by 
running
```require_once($filePath)``` which behind the scenes will execute the [include](http://php.net/manual/en/function
.include.php) statement.

```php
<?php

// /ComposerAutoloading/src/index.php

require_once 'Christmas/Santa.php';

$santa = new \Christmas\Santa();
$santa->sayIt(); // Merry Christmas && Write HQ code!
```

This is of course a good solution but un-scalable one as the project size would grow.

#### Composer solution

Composer, PHP Composer ;)

Instead of manually typing ```require_once``` every single time you want to include code from a different file you 
just import an auto-generated, self explanatory Composer file called: **autoload.php**.

Like Symfony does in it's frontend controllers: [index.php/app.php/app_dev.php/console.php](https://github.com/symfony/demo/blob/master/public/index.php#L17)

```php
<?php

// /ComposerAutoloading/src/index.php

// require_once 'Christmas/Santa.php';
require __DIR__.'/../vendor/autoload.php';

$santa = new \Christmas\Santa();
$santa->sayIt(); // Merry Christmas && Write HQ code!
```

Ehm Lukas but... I don't have any **autoload.php** file nor **vendor** folder.

Well, that's your problem.

Naaah, is Christmas. Let me show you how to generate them.

Everything starts with **composer.json** file. Run:

```
/ComposerAutoloading/
```

```bash
composer init
```

and wuala, **composer.json** without any dependencies, as we don't need them to get autoloading working:

```JSON
{
    "name": "enterprise-php/composer-autoloading",
    "description": "An example how Composer works behind the scenes.",
    "type": "project",
    "authors": [
        {
            "name": "Lukas Lukac",
            "email": "services@trki.sk"
        }
    ],
    "require": {}
}
```

Now you can generate the **autoload.php** by running:

```bash
composer install

    Loading composer repositories with package information
    Updating dependencies (including require-dev)
    Nothing to install or update
    Generating autoload files
```

The moment you included **autoload.php** in your **index.php**

```php
require __DIR__.'/../vendor/autoload.php';
```

you triggered Standard PHP Library (SPL) [function](http://php.net/manual/en/function.spl-autoload-register.php) ```spl_autoload_register(callable $autoloadFunction)``` the 
composer is using to register itself to take over the responsibility of autoloading PHP files on runtime (common 
Classes can be pre-cached).

```php
// /ComposerAutoloading/vendor/composer/ClassLoader.php

/**
 * Registers this instance as an autoloader.
 *
 * @param bool $prepend Whether to prepend the autoloader or not
 */
public function register($prepend = false)
{
    spl_autoload_register(array($this, 'loadClass'), true, $prepend);
}
```

Ha! Interesting. So, all my Classes across different Namespaces will be now autoloaded automatically? Let's see.

```bash
php index.php

Fatal error:  Uncaught Error: Class 'Santa' not found
```

Damn, Santa is still not found. That's pretty bad. Why?

Well, the registered **ClassLoader** executes the following ```loadClass($class)``` method (shortened, adjusted for 
simplicity):

```php

// /ComposerAutoloading/vendor/composer/ClassLoader.php

/**
 * Loads the given class or interface.
 *
 * @param  string    $class The name of the class
 * @return bool|null True if loaded, null otherwise
 */
public function loadClass($class)
{
    if ($file = $this->findFile($class)) {
        includeFile($file);

        return true;
    }
}

/**
 * Finds the path to the file where the class is defined.
 *
 * @param string $class The name of the class
 *
 * @return string|false The path if found, false otherwise
 */
public function findFile($class)
{
    // class map lookup
    if (isset($this->classMap[$class])) {
        return $this->classMap[$class];
    }
    
    if (null !== $this->apcuPrefix) {
        $file = apcu_fetch($this->apcuPrefix.$class, $hit);
        if ($hit) {
            return $file;
        }
    }

    // PSR-4 lookup
    $logicalPathPsr4 = strtr($class, '\\', DIRECTORY_SEPARATOR) . $ext;

    $first = $class[0];
    if (isset($this->prefixLengthsPsr4[$first])) {
        ...
    }

    // PSR-4 fallback dirs
    foreach ($this->fallbackDirsPsr4 as $dir) {
        if (file_exists($file = $dir . DIRECTORY_SEPARATOR . $logicalPathPsr4)) {
            return $file;
        }
    }

    // PSR-0 lookup
    if (false !== $pos = strrpos($class, '\\')) {
        // namespaced class name
        $logicalPathPsr0 = substr($logicalPathPsr4, 0, $pos + 1)
            . strtr(substr($logicalPathPsr4, $pos + 1), '_', DIRECTORY_SEPARATOR);
    } else {
        // PEAR-like class name
        $logicalPathPsr0 = strtr($class, '_', DIRECTORY_SEPARATOR) . $ext;
    }

    if (isset($this->prefixesPsr0[$first])) {
        foreach ($this->prefixesPsr0[$first] as $prefix => $dirs) {
            if (0 === strpos($class, $prefix)) {
                foreach ($dirs as $dir) {
                    if (file_exists($file = $dir . DIRECTORY_SEPARATOR . $logicalPathPsr0)) {
                        return $file;
                    }
                }
            }
        }
    }

    // PSR-0 fallback dirs
    foreach ($this->fallbackDirsPsr0 as $dir) {
        if (file_exists($file = $dir . DIRECTORY_SEPARATOR . $logicalPathPsr0)) {
            return $file;
        }
    }

    // PSR-0 include paths.
    if ($this->useIncludePath && $file = stream_resolve_include_path($logicalPathPsr0)) {
        return $file;
    }

    return $file;
}
```
Long story short. Composer checks different types of storage in quest of trying to find your Santa Class ordered
by the fastest access for performance reasons obviously.

1. in memory **classMap array**
2. APCU cache
3. disk using PSR-0 and PSR-4 standards
4. disk using "PEAR-like class name"

So if it's going through all this trouble why it can't find my Santa class? I mean, how hard can it be to find Santa...

Because it's not Kaspersky antivirus! It checks ONLY the places you configured it to check. 

Where can I configure it? Correct, where the journey has began, in **composer.json**.

You can either directly specify particular Classes in the classMap attribute (useful when no Namespace, clear 
directory structure is defined):

```JSON
{
    "name": "enterprise-php/composer-autoloading",
    "description": "An example how Composer works behind the scenes.",
    "type": "project",
    "authors": [
        {
            "name": "Lukas Lukac",
            "email": "services@trki.sk"
        }
    ],
    "require": {},
    "autoload": {
        "classmap": [
            "src/Christmas/Santa.php"
        ]
    }
}
```

which would result after running ```composer install``` to a new Hash file:

```php
<?php

// /ComposerAutoloading/vendor/composer/autoload_classmap.php
// autoload_classmap.php @generated by Composer

$vendorDir = dirname(dirname(__FILE__));
$baseDir = dirname($vendorDir);

return array(
    'Christmas\\Santa' => $baseDir . '/src/Christmas/Santa.php',
);
```

Therefore the Composer Class lookup will be straight forward:

```php
public function findFile($class)
{
    // class map lookup
    if (isset($this->classMap[$class])) {
        return $this->classMap[$class];
    }
```

OR. You can define a more broad **PSR-4** standard rule readable as: 

_Every time you will try to find a **Class** with **Namespace** starting with "Christmas", look into the 
"src/Christmas" directory._

```JSON
{
    "name": "enterprise-php/composer-autoloading",
    "description": "An example how Composer works behind the scenes.",
    "type": "project",
    "authors": [
        {
            "name": "Lukas Lukac",
            "email": "services@trki.sk"
        }
    ],
    "require": {},
    "autoload": {
        "psr-4": {
            "Christmas\\": "src/Christmas"
        }
    }
}
```

Once again, after running ```composer install``` you will get a new Hash like structured file **autoload_psr4.php**:

```php
<?php

// /ComposerAutoloading/vendor/composer/autoload_psr4.php

// autoload_psr4.php @generated by Composer

$vendorDir = dirname(dirname(__FILE__));
$baseDir = dirname($vendorDir);

return array(
    'Christmas\\' => array($baseDir . '/src/Christmas'),
);
```

One way or the other, the output of running ```php index.php``` will be, and I wish you too:

> Merry Christmas && Write HQ code!

#### Summary

If you want to play further with the code, you can find it all in this GitHub [directory](https://github
.com/EnchanterIO/enterprise-level-php/tree/landing-page/examples/ComposerAutoloading).

Thank you for your time reading my first technical article on **Enterprise Level PHP**.

I am also happy to announce the launch of my course for Junior, Advanced PHP Developers dedicated to software quality
and maintainable code!!!

It's my first project launch and I have very few Twitter followers therefore if you could re-tweet it and spread the 
word I would be EXTREMELY HAPPY.

[Read more about the course!](https://enterprise-level-php.com/)