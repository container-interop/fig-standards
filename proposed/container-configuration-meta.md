# Container Configuration Meta Document

## 1. Introduction

TODO

## 2. Why bother?

Modules (aka packages or bundles) are widespread in modern frameworks. Unfortunately each framework has its own convention and tools for writing them. This PSR is a step towards helping developers write modules that can work in any framework.

This PSR focuses on letting modules register container entries. That is necessary so that modules can expose services to users.

## 3. Scope

### 3.1. Goals

The goal of this PSR is really to make sure that containers can use a common API so that module authors can write only one package instead of one package per framework.

### 3.2. Non-goals

- The goal is not to standardize the way containers work. Each container should keep its own file format, API, specific features, etc... Containers implementing this PSR should only be able to support one additional format / API, and the minimum set of features defined in this PSR.  
  Container configuration is shared between the application developer and the packages bringing some configuration. Ultimately, container configuration is the responsibility of the application developer. Packages should however be able to bring a "default" configuration to help the developer getting started with sensible defaults (container configuration can be quite complex sometimes). This configuration should be both optional (possibility to bypass it completely), and overwritable (possibility to changes bits of the configuration).
- The way a framework can automatically **discover the container definitions** of a package (whether they are in PHP files or config files) is **out of scope** of this PSR.


## 4. Chosen approach

The chosen approach is to standardize *service providers*.

Service providers are classes that provide a list of entries to a container.

**Note**: the name "service provider" is a misnomer as the class really provides container **entries** (not only *services*). A container entry can be anything (an object, an array, a scalar value, etc...). However, the term "service provider" is wildly shared and accepted in the PHP world. In the rest of this document, we will use the term "service" and "container entry" interchangeably.

The chosen approach has been extensively tested in [container-interop/service-provider](https://github.com/container-interop/service-provider).

### 4.1. The basics

There are many ways to design service providers. Most service providers in existing framework are actively putting services in the container.

For instance in Pimple:

```php
namespace Pimple;

interface ServiceProviderInterface
{
    /**
     * Registers services on the given container.
     *
     * @param Container $pimple A container instance
     * @return void
     */
    public function register(Container $pimple);
}
```

Notice how the `register` method returns nothing in Pimple.


To be able to do the same thing in a cross-framework way, this would mean standardizing how entries can be put in a container (through a `set` method), which is notably hard given the differences between existing containers.

Instead, in the proposed approach, the service provider returns an associative array of factory methods.

```php
use Psr\Container\ServiceProvider;

class MyServiceProvider implements ServiceProvider
{
    public function getServices()
    {
        return [
            'my_service' => function() {
                return new MyService();
            }
        ];
    }
}
```

This offers a number of advantages:

- No need to specify a *set* method on containers
- By analyzing the keys of the returned array, containers know what entries are provided by a given service provider. They can cache this to call the `getServices` method "on demand".

If a factory needs a dependency, it can fetch it from the container:

```php
use Psr\Container\ServiceProvider;
use Psr\Container\ContainerInterface;

class MyServiceProvider implements ServiceProvider
{
    public function getServices()
    {
        return [
            'my_service' => function(ContainerInterface $container) {
                $dependency = $container->get('my_other_service');
                return new MyService($dependency);
            }
        ];
    }
}
```

Because the dependency is fetched from the container, we need a standardized interface to fetch those dependencies. Luckily, PSR-11 and the `ContainerInterface` is just done for that. The container where the service provider is registered must therefore implement the `ContainerInterface` (or provide an adapter to it that implements this interface).

It follows that this PSR has a strong dependency on PSR-11: this PSR cannot be accepted unless **PSR-11 is accepted first**.

In case a delegate container is registered in the container (see PSR-11), it is the delegate container that must be passed to the factory.

### 4.2. Extending services

Extending an entry before it is returned by the container is possible, via the second argument of the factory.

Factories accept the following parameters:

- the container (instance of `Interop\Container\ContainerInterface`)
- a callable that returns the previous entry if overriding/extending a previous entry, or `null` if not (note: this is still in discussion, see chapter 5.3)

Module A:

```php
class A implements ServiceProvider
{
    public function getServices()
    {
        return [
            'logger' => [ A::class, 'getLogger' ],
        ];
    }
    
    public static function getLogger()
    {
        return new Logger;
    }
}
```

Module B:

```php
class B implements ServiceProvider
{
    public function getServices()
    {
        return [
            'logger' => [ B::class, 'getLogger' ],
        ];
    }
    
    public static function getLogger(ContainerInterface $container, callable $getPrevious = null)
    {
        // Get the previous entry
        $previous = $getPrevious();

        // Register a new log handler
        $previous->addHandler(new SyslogHandler());
    
        // Return the object that we modified
        return $previous;
    }
}
```

### 4.3. Advantages of the chosen approach


- Using service providers leaves a lot of freedom when it comes to declaring instances: Entries are declared using a factory method is plain old PHP code. Anything is possible.
- Service providers are fairly easy to write
- The ServiceProvider interface is easy to agree upon (way more easy than other alternatives presented in chapter 6)
- Service providers are simple enough to implement in most existing containers. They can be implemented by almost all containers out there without requiring a major rewrite. They don't require containers to adopt a build stage (preprocessing/compiling/caching things) and yet, it is still possible to cache/compile service providers for maximum performance.


## 5. Design decisions

There are many ways to design a service provider. In this section, we detail the choices that were made in the design of the service provider interface.

### 5.1. Static versus non-static service providers

The initial version of [container-interop/service-provider](https://github.com/container-interop/service-provider/) was using static methods:

- the `getServices` method was static,
- factories could only be public static methods.

The rationale behind using static method was simple:

A factory should be a ["pure" method](https://en.wikipedia.org/wiki/Pure_function). Given a set of parameters (in our case: the container and an optional `$previous` argument), a pure method should always return the same result.
Since the whole configuration is stored in the container, there is no need for a state in the service provider. Static methods convey the idea that a method should be pure.

Furthermore, using public static methods only allows an easy optimisation for compiled or cache containers.

However, [a number of developers from the community at large](https://groups.google.com/d/msg/php-fig/xsY8bRG5K0M/RwIj_BswCgAJ) have [voiced their opinion against using only static methods](https://github.com/container-interop/service-provider/issues/5). Here is a summary of the arguments:

- When registering a service provider, you cannot type-hint it (i.e. `$container->register(ServiceProvider $serviceProvider)` vs `$container->register(string $serviceProviderClassName)`)
- Using only public static methods is more verbose / feels less natural
- Prevents the use of invokable objects as a factory. A set of well-known invokable objects can be used by compiled containers as a mean of optimisation.
- Static methods convey the idea of a pure method but there is no way to ensure it in PHP, as PHP is not a functional language (anyone could use a global or a static property of the service provider to carry some state)

### 5.2. One VS two methods

During the design of container-interop/service-provider, the following issue was raised: should we have only one `getServices` method that is used for both entries declaration and extension, or should we provide 2 different methods (`getServices` and `getServicesExtensions`), one for declaring a service and the other one for extending an already declared service.

WORK IN PROGRESS - This issue is [currently being discussed](https://github.com/container-interop/service-provider/issues/28)

### 5.3. The "previous" service argument

In order to allow extending an existing service, the service to be extended is passed in the second argument of the factory.
The format of this argument is still under discussion.

Possible alternatives are:

- pass a callable that will resolve to the service. Passing a callable has the advantage of not requiring an instantiation of the "previous" service if this previous service is not used (the factory might simply decide to completely override the service)
- simply pass the service to be extended as a second parameter. This is simpler but the previous service will automatically be resolved even when not needed (unless the container applies some techniques like looking at the signature of the factory...) 

The first version of container-interop/service provider did inject the service instance. [v0.2 switched to callables](https://github.com/container-interop/service-provider/issues/9). v0.4 might switch back to injecting the service instance.

WORK IN PROGRESS - This issue is [currently being discussed](https://github.com/container-interop/service-provider/issues/21)

### 5.4. Passing the container as an argument of `getServices`

WORK IN PROGRESS - This issue is [currently being discussed here](https://github.com/container-interop/service-provider/issues/27).

## 6. Alternatives to service providers

Using *service providers* is not the only possible way to register entries in a container.

The following alternatives have been considered to fulfil our goal:

- a standard static format, for example based on XML, JSON, YAML...
- standard PHP objects/interfaces representing container definitions.
- standard service providers (the chosen approach).

### 6.1. Standardizing a common file format

This solution is viable. This is what is done in Java with [CDI](http://www.cdi-spec.org/) or formerly with the [Spring framework](http://docs.spring.io/spring/docs/3.0.x/spring-framework-reference/html/xsd-config.html). This is also to some extend what Symfony 2/3 does with its [services.yml](http://symfony.com/doc/current/service_container.html) file.

The main issue with defining a common file format to describe container entries is to find the format. There are many to choose from.

`container-interop` members has a preference for XML (mainly because it has comments support and XSD can be used to check the validity of the XML and it is wildly supported by IDEs).

On the PHP-FIG mailing list, a number of other suggestions where done, ranging from widespread JSON or YAML file formats to more specialized formats (like NEON).

Because of the cost of parsing the configuration files, it requires some form of compilation or caching to be efficient, so can be tedious to implement. In particular, there is a risk that it may force simpler containers to switch to a "compiled" or "cached" approach.

**Strengths:**

- This approach is the only one that **allows an easy static analysis by external tools** (think about a support in an IDE, or a search tool for entries in Packagist or a third party website).
- Implementation is relatively easy for compiled containers.

**Weaknesses:**

- For each way to create a service, a particular syntax needs to be specified:
    - A syntax to create a service with the new keyword
    - A syntax to create a service using a static factory (`Factory::getMyService()`)
    - A syntax to create a service using a factory instance that is itself a service (`$container->get('myFactory')::getMyService()`)
    - A syntax to create arrays of services, string parameters, etc...
    - A syntax to call setters / methods
    - ...
- This is necessarily complex and a **cognitive burden** for the developers (yet another file format they need to master)
- Furthermore, even with a very feature-rich format, we will always reach a point where we cannot do something. For instance, what if we want to concatenate 2 configuration strings and pass those as a parameter to a constructor? (Symfony solves this with the Expression Language, but this becomes clearly out of reach of any standard).
- The more features we add, the harder the implementation for containers. We have to find a correct balance between available features and easiness of implementation. This might be a difficult balance to strike.
- A configuration format is a performance hit for "runtime containers". A container cannot parse all configuration files on every request. The only possible solutions are doing **heavy caching** or **preprocessing the configuration files and generating a PHP class** (i.e. « compiling » the container). De-facto, this means adding the notion of a "build stage" to all applications. This is a big step for small/light frameworks. Some projects on the PHP-FIG mailing list have said this is a show-stopper for them.
- There is a non-trivial choice to make for the **common file format**. This is a huge challenge. There are many formats out there: JSON / XML / YML / Neon, etc...


### 6.2. Standardizing container definitions

This has been extensively tested in [container-interop/definition-interop](https://github.com/container-interop/definition-interop).

In this approach, we define a set of interfaces that describe container entries.

The whole idea is to have one interface per technique to construct container entries.

- To declare an entry with the new keyword, you would use the `ObjectDefinitionInterface`.
- To declare an entry created by a factory, you would use the `FactoryCallDefinitionInterface`.
- ...

There is one interface per supported way of creating a container entry.

**Prototypes built around this concept:**

[There is a direct implementation of the interface named Assembly.](https://github.com/mnapoli/assembly)

List of containers consuming this interface:
 
- Assembly provides a PSR-11 compliant container consuming container definitions at runtime.
- [Yaco is a compiler that generates a PSR-11 compliant container from container definitions](https://github.com/thecodingmachine/yaco) (i.e. a container builder or container compiler).
- Although not completely ready, a Symfony compiler pass that can consume container definitions was tested. It is definitely possible possible and easy to implement.

Finally, a proof-of-concept that takes [YML service files (in the Symfony format) and converts them into definitions compatible with definition-interop](https://github.com/thecodingmachine/yaml-definition-loader). What is really great with this approach is that we don’t have to choose a file format any more. Anyone can define its own file format (be it JSON / XML / YML / annotations / whatever...) as long as it can be converted to a set of definition objects.

**Strengths:**

- Implementation is relatively easy
- Offers some freedom regarding the best file format

**Weaknesses:**

Most of the weaknesses are shared with the common file format approach:

- For each way to create an service, we need a particular interface:
    - An interface to create a service with the new keyword (`ObjectDefinitionInterface`)
    - An interface to create a service using a static factory (`FactoryCallDefinitionInterface`)
    - A syntax to create a service using a factory instance that is itself a service (`FactoryCallDefinitionInterface`)
    - A syntax to create arrays of services, string parameters  (`ParameterDefinitionInterface`), etc...
    - A syntax to call setters / methods
    - ...
- This is necessarily complex and a **cognitive burden** for the developers. The global architecture (definitions, definition loaders...) is somehow complex to understand for newcomers.
- Furthermore, even with numerous interfaces, we will always reach a point where we cannot do something. For instance, what if we want to concatenate 2 configuration strings and pass those as a parameter to a constructor?
- The more features we add, the harder the implementation for containers. We have to find a correct balance between available features and easiness of implementation. This might be a difficult balance to strike.
- Performance-wise, this is bad for "runtime containers". For each entry we want to create, we need to instantiate several definition objects. If the objects are generated from « loaders », it gets even worse. The only possible solutions are doing heavy caching or preprocessing the configuration files and generating a PHP class (i.e. « compiling » the container). De-facto, this means adding the notion of a "build stage" to all applications.


### 6.3. Rationalizing the choice between service providers, common file format and definition interface.

To choose between service providers over other solutions (like a standardized configuration format or a standardized object describing a container entry), we split the problem in 2 parts:

- **Defining a minimum set of features that each container should have**: what features a container MUST support (for instance: tagging, instantiation using factories, extending existing services, etc...)
- **Testing each proposed solution against the set of proposed features** and deciding which solution solves matches the best

### 6.3.1. Features

Below is a list of all possible features that have been suggested to fulfill the goal of this PSR: packages registering container entries.

They are sorted by relevance.

#### Absolutely needed

1. Ability to create a container entry using the `new` keyword.

1. Ability to call methods (setters or otherwise) on a container entry.

1. Ability to create a container entry using a factory method from a container service.

1. Ability to alias a container entry to another.

1. Ability to modify an entry defined outside of the "module" before it is returned by the container. For instance: ability for a package providing a `Twig_Extension` to modify the `Twig_Environment` by calling `$environment->addExtension($extension)`, even if the `Twig_Environment` entry was not declared by the package itself.

1. Ability to create container entries for scalar values.

1. Ability to create container entries that are numerically indexed arrays. Values of the array can be any valid container entry (i.e. objects, scalars, another array...)

1. Ability to create container entries that are associative arrays (maps). Values of the array can be any valid container entry (i.e. objects, scalars, another array...)

1. Ability for a package to extend those arrays (add elements to the arrays). Think about a package adding a "log handler" to the list of available log handlers.

1. Ability to have several services for the same class or interface (for instance, several services implementing a `LoggerInterface`). In other words: should services have ids, or should they only be bound to a FQCN?

#### Nice to have

11. Ability to set public properties of a container entry.

1. Ability to create a container entry using a static factory method.

1. Ability to create a container entry using a closure.

1. Ability to compile all container entries into a single container for maximum performance.

1. Ability to create container entries from constants (from the `define` keyword or the `const` keyword).

1. Ability to locally declare "anonymous"/"private" services in a package. These are services that can be only used by the package declaring them and are not accessible in other packages or through the container "get" method.

1. Ability to have "optional" references: When binding a service to another service, this is the ability to bypass a reference if the entry does not exist in the container. For instance (using pseudo PHP code):
   ```php
   if ($container->has('dependency')) {
       $dependency = $container->get('dependency');
   } else {
       $dependency = null
   }
   $service = new Service($dependency);
   ```

1. Ability to have static tools analyzing the bindings (for instance, having Packagist analyze the bindings to search for some services...)

1. Ability to perform simple computations on values before injecting them in a container entry (for instance, grab a return value of a function of a service, add it to another value and inject it in another service. This kind of feature comes "out of the box" for closure based services, and can be more complex to implement with definitions. See Symfony's "Expression language")

1. Ability to directly debug the code generating the services (using Xdebug or a similar tool). This is typically a feature available in service providers and not available in configuration files.

1. Ability to get customized error messages in case of misconfiguration. This is the ability, for a package, to throw an error/exception with a detailed custom message if a set of prerequisites is not met. For instance, if an entry expects the container to contain a set of options/dependencies, this is the ability to throw a custom exception message explaining what options are missing, but also how to configure those options.

#### Indifferent

21. Ability for a package to manage priority when inserting a service in an array (add a service at the beginning, at the end, in the middle of an array, or give a priority...).

1. Ability to have fall-back aliases/services: a alias/service is only declared by a package if no other package has provided that service so far.

1. Ability to have static tools help us edit the binding. For instance, a dedicated UI that can be used to create services and drag'n'drop services together (like Mouf does)

#### Not needed

24. Ability to declare "lazy" services (services that are wrapped into proxy objects and instanciated only when needed).

1. Ability to provide a default service that should be used when binding an interface.

#### Highly counterproductive

26. Ability to declare if the service should be instantiated once and reused (singleton) or if the service should be instantiated every time it is injected or fetched from the container.

1. Ability to set private and protected properties of a container entry.



### 6.3.2. Feature support

List of features that can be supported for each configuration format.

Note: only features deemed absolutely needed or nice to have have been kept.

Feature | Static format | Definition interfaces | Service provider
--------|:-------------:|:-------------:|:-------------:
**Absolutely needed features** | | |
Ability to create a container entry using the new keyword. | ++ | ++ | ++
Ability to call methods (setters or otherwise) on a container entry | ++ | ++ | ++
Ability to create a container entry using a factory method from a container service. | ++ | ++ | ++
Ability to alias a container entry to another. | ++ | ++ | ++ 
Ability to modify an entry defined outside of the "module" before it is returned by the container.| + | + | ++
Ability to create container entries for scalar values. | ++ | ++ | ++ 
Ability to create container entries that are numerically indexed arrays. Values of the array can be any valid container entry (i.e. objects, scalars, another array...) | ++ | ++ | ++
Ability to create container entries that are associative arrays (maps). Values of the array can be any valid container entry (i.e. objects, scalars, another array...) | ++ | ++ | ++
Ability for a package to extend those arrays (add elements to the arrays). | + | + | ++
Ability to have several services for the same class or interface (for instance, several services implementing a LoggerInterface). | ++ | ++ | ++
**Nice to have features** | | |
Ability to set public properties of a container entry. | ++ | ++ | ++
Ability to create a container entry using a static factory method. | ++ | ++ | ++
Ability to create a container entry using a closure | -- | -- | ++
Ability to compile all container entries into a single container for maximum performance. | ++ | ++ | + [(1)](#explanation_1)
Ability to create container entries from constants (from the define keyword or the const keyword) | ++ | ++ | ++
Ability to locally declare "anonymous"/"private" services in a package. | + | + | ++
Ability to have "optional" references | + | + | ++
Ability to have static tools analyzing the bindings (for instance, having Packagist analyze the bindings to search for some services...) | ++ | - | --
Ability to perform simple computations on values before injecting them in a container entry | -- | - | ++
Ability to directly debug the code generating the services (using Xdebug or a similar tool). | -- | - | ++
Ability to get customized error messages in case of misconfiguration. | - | - | ++

<a name="explanation_1"></a> (1) For compiled containers, a static format or a definition interface allows to directly put the code generating the services in the container (maximum performance). With service providers, the compiled container will call a static factory method of the service provider. So creating a service from a service provider will generate an additional method call.

The few differences can be summed up as:

- the static file format and definition objects can be used to their full potential by compiled containers
- the service provider approach leaves the most freedom as any PHP code can be used by module authors to create container entries
- with a static file format or a definition object, each new way of instantiating a service makes the standard more complex. With service providers, since instantiation is performed via PHP code, complexity stays the same.

# Relevant links

- The workgroup's project: [container-interop/service-provider](https://github.com/container-interop/service-provider/)
- The workgroup's [Gitter channel](https://gitter.im/container-interop/definition-interop)
- [Discussion](https://github.com/container-interop/fig-standards/issues/9) and [vote](https://github.com/container-interop/fig-standards/wiki/Vote-results-for-the-important-feature-list) regarding the list of relevant features
- [PHP-FIG post](https://groups.google.com/forum/#!topic/php-fig/xsY8bRG5K0M) and [blog article](http://www.thecodingmachine.com/psr-11-an-overview-of-interoperable-php-modules/) regarding the choice of the best approach
- [PHP-FIG post](https://groups.google.com/forum/#!searchin/php-fig/service$20providers%7Csort:relevance/php-fig/5u1HESgvBUQ/GlHw74zoAAAJ) and  [blog article](http://www.thecodingmachine.com/psr-11-get-ready-for-universal-service-providers/) regarding the scope of this PSR