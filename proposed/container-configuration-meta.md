# Container Configuration Meta Document

## 1. Introduction

TODO

## 2. Why bother?

Modules (aka packages or bundles) are widespread in modern frameworks. Unfortunately each framework has its own convention and tools for writing them. This PSR is a step towards helping developers write modules that can work in any framework.

This PSR focuses on letting modules register container entries. That is necessary so that modules can expose services to users.

## 3. Scope

### 3.1. Goals

The goal of this PSR is really to make sure that containers can read a common format or use a common API so that module authors can write only one package instead of one package per framework.

To reach this goal, the problem can be split into several sub-parts:

- **Defining a minimum set of features that each container should have**: what features a container MUST support (for instance: tagging, instantiation using factories, extending existing services, etc...)
- **Defining a common format**: this PSR must decide if interoperability is performed at the container entry descriptor level (interfaces for objects describing services), or at the file level, in which case a common file format must be decided (JSON / XML / YML / NEON, etc...)

### 3.2. Non-goals

The goal is not to standardize the way containers work. Each container should keep its own file format, API, specific features, etc... Containers implementing this PSR should only be able to support one additional format / API, and the minimum set of features defined in this PSR.

The way a framework can automatically **discover the container definitions** of a package (whether they are in PHP files or config files) is **out of scope** of this PSR.

## 4. Approaches

### 4.1. Features

First of all, a word of caution. Container configuration is shared between the application developer and the packages bringing some configuration. Ultimately, container configuration is the responsibility of the application developer. Packages should however be able to bring a "default" configuration to help the developer getting started with sensible defaults (container configuration can be quite complex sometimes). This configuration should be both optional (possibility to bypass it completely), and overwritable (possibility to changes bits of the configuration).

In this section, we will describe the list of features that we can consider to fulfill the goal of this PSR: packages registering container entries. Let's list all possible features and later, we can vote on the features we want to keep or not.

1. Ability to create a container entry using the `new` keyword.

1. Ability to call methods (setters or otherwise) on a container entry.

1. Ability to set public properties of a container entry.

1. Ability to set private and protected properties of a container entry.

1. Ability to create a container entry using a static factory method.

1. Ability to create a container entry using a factory method from a container service.

1. Ability to create a container entry using a closure.

1. Ability to compile all container entries into a single container for maximum performance.

1. Ability to alias a container entry to another.

1. Ability to modify an entry defined outside of the "module" before it is returned by the container. For instance: ability for a package providing a `Twig_Extension` to modify the `Twig_Environment` by calling `$environment->addExtension($extension)`, even if the `Twig_Environment` entry was not declared by the package itself.

1. Ability to create container entries for scalar values.

1. Ability to create container entries from constants (from the `define` keyword or the `const` keyword).

1. Ability to create container entries that are numerically indexed arrays. Values of the array can be any valid container entry (i.e. objects, scalars, another array...)

1. Ability to create container entries that are associative arrays (maps). Values of the array can be any valid container entry (i.e. objects, scalars, another array...)

1. Ability for a package to extend those arrays (add elements to the arrays). Think about a package adding a "log handler" to the list of available log handlers.

1. Ability for a package to manage priority in those arrays (add a service at the beginning, at the end, in the middle, or give a priority...).

1. Ability to locally declare "anonymous"/"private" services in a package. These are services that can be only used by the package declaring them and are not accessible in other packages or through the container "get" method.

1. Ability to provide a default service that should be used when binding an interface.

1. Ability to declare "lazy" services (services that are wrapped into proxy objects and instanciated only when needed).

1. Ability to have several services for the same class or interface (for instance, several services implementing a `LoggerInterface`). In other words: should services have ids, or should they only be bound to a FQCN?

1. Ability to declare if the service should be instantiated once and reused (singleton) or if the service should be instantiated every time it is injected or fetched from the container.

1. Ability to have "optional" references: When binding a service to another service, this is the ability to bypass a reference if the entry does not exist in the container. For instance (using pseudo PHP code):
   ```php
   if ($container->has('dependency')) {
       $dependency = $container->get('dependency');
   } else {
       $dependency = null
   }
   $service = new Service($dependency);
   ```

1. Ability to have fall-back aliases/services: a alias/service is only declared by a package if no other package has provided that service so far.

1. Ability to have static tools analyzing the bindings (for instance, having Packagist analyze the bindings to search for some services...)

1. Ability to have static tools help us edit the binding. For instance, a dedicated UI that can be used to create services and drag'n'drop services together (like Mouf does)

1. Ability to perform simple computations on values before injecting them in a container entry (for instance, grab a return value of a function of a service, add it to another value and inject it in another service. This kind of feature comes "out of the box" for closure based services, and can be more complex to implement with definitions. See Symfony's "Expression language")

1. Ability to directly debug the code generating the services (using Xdebug or a similar tool). This is typically a feature available in service providers and not available in configuration files.

1. Ability to get customized error messages in case of misconfiguration. This is the ability, for a package, to throw an error/exception with a detailed custom message if a set of prerequisites is not met. For instance, if an entry expects the container to contain a set of options/dependencies, this is the ability to throw a custom exception message explaining what options are missing, but also how to configure those options.

When the list of features is complete, I propose we create a poll on each feature, ranking them from -- to ++:

- -- = highly counterproductive
- - = not needed
- / = indifferent
- + = nice to have
- ++ = absolutely needed


### 4.2. The format

TODO

