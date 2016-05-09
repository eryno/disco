# bitexpert/disco
This package provides a [container-interop](https://github.com/container-interop/container-interop) compatible, 
annotation-based dependency injection container. Have a look at the [adroit-disco-demo](https://github.com/bitExpert/adroit-disco-demo) 
project to find out how to use Disco.

[![Build Status](https://travis-ci.org/bitExpert/disco.svg?branch=master)](https://travis-ci.org/bitExpert/disco)
[![Dependency Status](https://www.versioneye.com/php/bitexpert:disco/dev-master/badge.svg)](https://www.versioneye.com/php/bitexpert:disco/dev-master)

Installation
------------

The preferred way of installing `bitexpert/disco` is through Composer. Simply add `bitexpert/disco` as a dependency:

```
composer.phar require bitexpert/disco
```

Usage
-----

Disco provides a large amount of functionality by utilizing a few simple comment annotations (ex: "@return").

To get started, define a configuration class. It can be named whatever you like, but must be marked with an @Configuration annotation:

```php
use bitExpert\Disco\Annotations\Configuration;

/**
 * @Configuration
 */
class MyConfiguration
{
}
```

Add a method for each object instance you want to let Disco manage for you. Create a public method for each instance that should be publicly available. These instances will be accessible via the has() and get() methods of the \bitExpert\Disco\AnnotationBeanFactory class. 

Use protected/private methods for local dependencies used "inside" the container configuration. 
This is useful for dependencies like database connectors, which you may need to construct objects but most likely do not want to expose as a "real" dependency.

Each method must be marked with an @Bean annotation, as well as the @return annotation defining the type of the 
dependency. (The @return annotation is mainly needed for the lazy proxy magic).

```php
use bitExpert\Disco\Annotations\Bean;
use bitExpert\Disco\Annotations\Configuration;
use bitExpert\Disco\Helper\SampleService;

/**
 * @Configuration
 */
class MyConfiguration
{
    /**
     * @Bean
     * @return SampleService
     */
    public function mySampleService()
    {
        return new SampleService();
    }
}
```

The @Bean annotation accepts a few configuration settings for influencing the lifetime of the instance. You can define the "scope" (either the string "request" or "session"), specify a boolean value for "lazy" (to only load the object once it's needed), or specify a boolean value for "singleton". 

The following configuration creates a lazy-loaded, singleton, request aware instance:

```php
use bitExpert\Disco\Annotations\Bean;
use bitExpert\Disco\Annotations\Configuration;
use bitExpert\Disco\Helper\SampleService;

/**
 * @Configuration
 */
class MyConfiguration
{
    /**
     * @Bean({"singleton"=true, "lazy"=true, "scope"="request"})
     * @return SampleService
     */
    public function mySampleService()
    {
        return new SampleService();
    }
}
```

To inject a dependency simply call the respective method:

```php
use bitExpert\Disco\Annotations\Bean;
use bitExpert\Disco\Annotations\Configuration;
use bitExpert\Disco\Helper\SampleService;
use bitExpert\Disco\Helper\MasterService;

/**
 * @Configuration
 */
class MyConfiguration
{
    /**
     * @Bean
     * @return SampleService
     */
    public function mySampleService()
    {
        return new SampleService();
    }
    
    /**
     * @Bean
     * @return MasterService
     */
    public function myMasterService()
    {
        return new MasterService($this->mySampleService());
    }
}
```

To inject parameters use the @Parameter annotation. Parameters need to be passed as a constructor argument to the 
\bitExpert\Disco\AnnotationBeanFactory instance:

```php
use bitExpert\Disco\Annotations\Bean;
use bitExpert\Disco\Annotations\Configuration;
use bitExpert\Disco\Helper\SampleService;

/**
 * @Configuration
 */
class MyConfiguration
{
    /**
     * @Bean
     * @Parameters({
     *  @Parameter({"name" = "test"})
     * })
     * @return SampleService
     */
    public function mySampleService($test = '')
    {
        $service = new SampleService();
        $service->setTest($test);
        return $service;
    }
}
```

To instantiate the \bitExpert\Disco\AnnotationBeanFactory instance, proceed as follows: 

```php
$parameters = ['test' => 'This is a test.'];

$beanFactory = new \bitExpert\Disco\AnnotationBeanFactory(MyConfiguration::class, $parameters);
\bitExpert\Disco\BeanFactoryRegistry::register($beanFactory);
```

Since Disco implements the [container-interop](https://github.com/container-interop/container-interop) recommendation,
you are able to call the has() and get() method as expected:

```php
$beanFactory->has('mySampleService');

$beanFactory->get('mySampleService');
```

To configure there Disco will store the generated proxy classes or the annotation metadata pass an instance of 
\bitExpert\Disco\BeanFactoryConfiguration as 3rd parameter when creating the \bitExpert\Disco\AnnotationBeanFactory 
instance:

```php
$parameters = ['test' => 'This is a test.'];
$config = new \\bitExpert\Disco\BeanFactoryConfiguration('/tmp/');

$beanFactory = new \bitExpert\Disco\AnnotationBeanFactory(MyConfiguration::class, $parameters, $config);
\bitExpert\Disco\BeanFactoryRegistry::register($beanFactory);
```

To influence the object creation you are able to register post processors by annotating the methods with the 
@BeanPostProcessor annotation. The method needs to return an instance implementing the \bitExpert\Disco\BeanPostProcessor
interface.

```php
use bitExpert\Disco\Annotations\Bean;
use bitExpert\Disco\Annotations\BeanPostProcessor;
use bitExpert\Disco\Annotations\Configuration;
use bitExpert\Disco\Helper\SampleService;
use bitExpert\Disco\Helper\SampleServiceBeanPostProcessor;

/**
 * @Configuration
 */
class MyConfiguration
{
    /**
     * @BeanPostProcessor
     * @return SampleServiceBeanPostProcessor
     */
    public function sampleServiceBeanPostProcessor()
    {
        return new SampleServiceBeanPostProcessor();
    }

    /**
     * @Bean({"singleton"=true, "lazy"=true, "scope"="request"})
     * @return SampleService
     */
    public function mySampleService()
    {
        return new SampleService();
    }
}
```

Performance Tuning
------------------

Since a lot of parsing and reflection logic is involved during the conversion process of the configuration code into
its "final format" bitexpert/disco can be rather slow in production and during development, especially when running 
in a virtual machine, e.g. with Vagrant. 

Make sure to follow the hints on how to improve performance for [Doctrine Annotations](http://doctrine-orm.readthedocs.org/projects/doctrine-common/en/latest/reference/annotations.html)
and pick a \\Doctrine\\Common\\Cache\\Cache implementation that suites your needs. To use a specific cache implementation
pass it to \\bitExpert\\Disco\\BeanFactoryConfiguration::construct() as 3rd parameter.

In addition to that the [ProxyManager](https://github.com/Ocramius/ProxyManager) needs to be configured for faster
performance, read about the details [here](https://ocramius.github.io/ProxyManager/docs/tuning-for-production.html). To
enable the usage of the custom autoloader simply set the 4th parameter of \\bitExpert\\Disco\\BeanFactoryConfiguration::construct()
to true. This will massively increase the overall performance of bitexpert/disco.

When enabling the caching methods make sure you regularly clean your caching folder after changing your configuration code!

License
-------

Disco is released under the Apache 2.0 license.
