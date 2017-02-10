# Chapter I

In this chapter, let's talk about the service system of Symfony, in more precise way, let's talk about the logic of using services in symfony.


## introduction

_A service is a simple php class who bring new functionality into your code ..._ **Some random dude, 2017**

So, what's a so called Service ?

Simple description, a service is a php class who bring a new functionality into your symfony application, this way, the functionality become available through the whole application.

Let's imagine a simple php class who write something into your logs :

```php
<?php

namespace App;

class LogWriter
{
    private $logger;

    public function __construct(LoggerManager $logger)
    {
        $this->logger = $logger;
    }

    public function writeOnEvent()
    {
        $this->logger->write('hey dude !');
    }
}
```

Well, if you look at this class, the logic behind is pretty simple, we inject some random logger into our class and use
the injected class to write something into the logs.

This class _can_ become a service into a classic Symfony application, in fact, Symfony **has** a logger service who
write into the log, the so-called class 'logger' is available in Symfony via the DependencyInjection component, more in depth,
via the class 'Logger' (who depend on the bridge with Monolog, the famous logger).

Anyway, what the class we wrote is about ? Simply bring the write method into our application, not so much.

So, how can we define this class has a service into Symfony ?

In a simple way using services.yml file (in the app/config folder) :

```yaml
services:

    app.logwriter
        class: App\LogWriter
            arguments:
                - '@logger'
```

In this example, we define the class who contains our service, in the second line, we define an argument, but, what the hell is that ?

In Symfony, everything in services ! Yeah, i know, that can be overwhelming but that the strict truth, even the kernel
is a service, in this way, Symfony use a **huge** amount of class defined as a service into his logic and this classes
use a lot of dependencies, the so-called **_arguments_**, by defining a arguments with the name 'logger', we call the class
who's been registered as logger into the ContainerBuilder class (the heart of DependencyInjection component), if we dump the container :

```bash
./bin/console debug:container logger
```

Here's what said the console :

```bash
 ------------------ -------------------------------------------------------------------
  Option             Value
 ------------------ -------------------------------------------------------------------
  Service ID         logger
  Class              Symfony\Bridge\Monolog\Logger
  Tags               -
  Calls              pushProcessor, useMicrosecondTimestamps, pushHandler, pushHandler
  Public             yes
  Synthetic          no
  Lazy               no
  Shared             yes
  Abstract           no
  Autowired          no
  Autowiring Types   Psr\Log\LoggerInterface
 ------------------ -------------------------------------------------------------------
 ```

 Simple things, the logger is the bridge with Monolog and by calling this service name (also called alias later),
 we call the class, by meaning this, the container (aka ContainerBuilder) create a new instance of the class and make her
 available through our entire application.

 What a cool story but how can we call our service into Symfony ? We save the service into services.yml and this file
 is called and "read" by the heart of Symfony, every file into the config folder is read in fact, but, we gonna talk about this later.

 Once this file is read, Symfony dump every class as a object and save them into the ContainerBuilder using the name, the arguments and even
 the _tags_ (later on this) described into the services.yml file.

 In order to call this service into a Controller (MVC pattern to the rescue !), Symfony use a shortcut :

 ```php
 <?php

 namespace AppBundle\Controllers;

 use Symfony\Bundle\FrameworkBundle\Controller\Controller;

 class AppController extends Controller
 {
    public function indexAction()
    {
        return $this->get('app.logger')->write();
    }
 }
```
This controller call the service saved sooner and especially, the write method, this way, we don't need to allow PHP to
save a variable into memory, this way, better performances !

**_Just for precision, this code isn't completely valid by the way that we don't return a Response, who's mandatory in Symfony
but we simply want to show the process._**

Alright, now that we have a service and we call him, how can we improve this one ?

## Case I - DependencyInjection power !

Now that we know how to build a service, let's go deeper into the logic of bringing service into our application.

For what we said, a service is a simple php class, fact but in the case that we aren't clear (like 200% clear), a service
SHOULD be free of Symfony logic ...

Yes, that can seem strange but that's true, let's imagine a simple case of a service who log (like the introduction) :

```php
<?php

namespace App;

class LogWriter
{
    private $logger;

    public function __construct(LoggerManager $logger)
    {
        $this->logger = $logger;
    }

    public function writeOnEvent()
    {
        $this->logger->write('hey dude !');
    }
}
```

This class is pretty simple, we inject a class who's bridged by Symfony, heck no ... bad idea !

Bu where's the problem with this logic ? In fact, if we respect the OOP rules (oh god, i like this part !),
a class shouldn't be linked to a framework-logic, in our 'language', we call this a framework-agnostic class,
this simply mean that our class isn't linked to the framework and can be used outside of this last one.

In this example, the class come from Symfony and this way, if we inject the service into a separate framework like Zend,
Laravel, CakePHP and even Silex, this could not work ... Very sad news.

Bu how can we change this to a better way ?

Simply by using the dependencyInjection pattern !

_I love to say this in front of the world with a bottle of water in my hand, look so cool._

But what's this dependencyInjection pattern is ?

Simply a pattern (a way in normal language) who help you solve one problem, what if a class need an other to perform some actions ?

So, how can we solve our class ?

First, let's create a interface (hope you know what's a interface ?) :

```php
<?php

namespace App;

interface LoggerLoaderInterface
{
    /** @param object $logger */
    public function buildInstance($logger){}

    /**
     * Allow to retrieve a logger using his name
     */
    public static function getLogger($logger){}
}
```

So, what do this interface ?

Simple things in mind, we create a method who's gonna build the instance of the given logger, this way, our class don't
need to know every class by default, we gonna pass a instance and the getLogger gonna return the logger due to his name.

So, how can we use this interface into our class ? First, create a class named LoggerLoader :

```php
<?php

namespace App\Loaders;

class LoggerLoader implements
      LoggerLoaderInterface
{
    /** @var array */
    protected $loggers;

    /** {@inheritdoc} */
    public function buildInstance($logger)
    {
        $logger = new $class();

        if (!$logger) {
            throw new \InvalidArgumentException(
                sprintf(
                    'The class MUST be a logger instance,
                    given "%s"', gettype($logger);
                )
            )
        }

        $this->loggers[$logger] = $logger;
    }

    /** {@inheritdoc} */
    public function getLogger($logger)
    {
        if (!is_string($logger)) {
            throw new \InvalidArgumentException(
                sprintf(
                    'The argument MUST be a string,
                    given "%s"', gettype($logger);
                )
            )
        }

        if (array_key_exists($logger, $this->loggers)) {
            return $this->loggers[$logger];
        }

        return;
    }
}
```

Now, let's update our first class :

```php
<?php

namespace App;

use App\LoggerLoaderInterface;

class LogWriter
{
    private $loader;

    public function __construct(LoggerLoader $loader)
    {
        $this->loader = $loader;
    }

    public function writeOnEvent($logger)
    {
        $this->loader->getLogger($logger)->write('hey dude !');
    }
}
```

Now, that's a little bit cleaner but we have a massive problem, we repeat code and the structure isn't pretty clear,
if we update the services.yml, here's what we gonna need to do :

```yaml
services:

    app.loader:
        class: App\LoggerLoader

    app.logwriter
        class: App\LogWriter
            arguments:
                - '@app.loader'
```

Alright, pretty complex way to do simple things, indeed, we can save this way and stay with this approach
but to be clear, that's clearly not a good idea.

This type of approach is much affordable if you build your application from the ground up, not if you use
a framework like Symfony and to stay consistent, that's not the way you want to code your application.

So, here it's time to use our super magical pattern, DependencyInjection (imagine ground shaking and glory cry) !

So, we saw that Symfony inject dependencies through our application and classes by simply adding them to
our services.yml as an argument but how call it into our classes ?

Let's take a simple example (imagine a manager) :

```php
<?php

namespace AppBundle\Managers;

class EntityManager
{

}
```

This class could be linked to a Entity (let's say Article, pretty complex example ...) and we gonna manage
this entity with the present class, like ... well, like a manager.

So, let's build our class for retrieving our entity from the BDD :

```php
<?php

namespace AppBundle\Managers;

class EntityManager
{
    // This must return an array of all the articles stored into the BBD.
    public function getArticles()
    {
        return $this->getSomething->getAllArticles();
    }
}
```

In this simple case, we could call a ORM (for Object Relation Mapper ou Mapping, choose your favorite version)
and expect him to have a method who call the entity stored into the BDD.

In this case, we're in Symfony and we gonna call Doctrine (because he rocks), let's update this :

```php
<?php

namespace AppBundle\Managers;

class EntityManager
{
    private $doctrine;

    // This must return an array of all the articles stored into the BBD.
    public function getArticles()
    {
        return $this->doctrine->getAllArticles();
    }
}
```

Pretty simple but definitively not good, indeed, how can we say that Doctrine could call the EntityManager ?

Personally, i know he can but how can we say to a other developer that it's possible ?

Short answer ... By calling the class and not the "package" around the class, explanation :

In a "dependency" like Doctrine, everything is called by the service @doctrine (in Symfony case), this
service is an alias for a simple class :

```bash
 ------------------ -----------------------------------------
  Option             Value
 ------------------ -----------------------------------------
  Service ID         doctrine
  Class              Doctrine\Bundle\DoctrineBundle\Registry
  Tags               -
  Public             yes
  Synthetic          no
  Lazy               no
  Shared             yes
  Abstract           no
  Autowired          no
  Autowiring Types   -
 ------------------ -----------------------------------------
```

Look tempting but this class is a simple "facade" (Laravel people gonna hate me)
for the whole Doctrine package, when you call the EntityManager, you need to say :

```php
    $doctrine->getEntityManager()->getRepository()->SayHello();
```

Well, now that we know this, how can we call directly the EntityManager ?

Let's debug our Container :

```bash
    ./bin/console debug:container entity
```

```bash
  [0] form.type.entity
  [1] doctrine.orm.default_entity_listener_resolver
  [2] doctrine.orm.default_listeners.attach_entity_listeners
  [3] doctrine.orm.default_entity_manager
  [4] doctrine.orm.default_entity_manager.property_info_extractor
  [5] doctrine.orm.entity_manager
```

Lets choose the option five (i can smell it, that's the key !),
oops, that's a alias for the key 3, let's choose the third option :

```bash
 ------------------ -------------------------------------
  Option             Value
 ------------------ -------------------------------------
  Service ID         doctrine.orm.default_entity_manager
  Class              Doctrine\ORM\EntityManager
  Tags               -
  Public             yes
  Synthetic          no
  Lazy               yes
  Shared             yes
  Abstract           no
  Autowired          no
  Autowiring Types   -
  Factory Class      Doctrine\ORM\EntityManager
  Factory Method     create
 ------------------ -------------------------------------
```

That's it, we have our class, the service ID is the name of the service we need to call,
let's update our services.yml :

```yaml
services:

    app.entity_manager
        class: AppBundle\Managers\EntityManager
            arguments:
                - '@doctrine.orm.default_entity_manager'
```

Alright, now that we 'inject' the EntityManager, how can we use him into our class ?

In Symfony, we can call a dependency by multiple way into a service, you can use the __construct,
the setter and even by the attributes, by default, we use the __construct with a simple attributes
typehinted to the dependency.

Let's update our class :

```php
<?php

namespace AppBundle\Managers;

use Doctrine\ORM\EntityManager;

class EntityManager
{
    /** @var EntityManager */
    private $doctrine;

    public function __construct(
        EntityManager $doctrine
    ) {
        $this->doctrine = $doctrine;
    }

    // This must return an array of all the articles stored into the BBD.
    public function getArticles()
    {
        return $this->doctrine->getRepository('AppBundle:Article')->getAllArticles();
    }
}
```

That's way better, in fact, we could name the attributes $em for reading convention but who care,
we're just here to understand and practice.

So, now, we have a class who can manage the entity Article, nice thing with another bad thing, we can
retrieve the entity by a simple method, we don't allow a variable in order to save memory, you can use
one if you prefer but that's clearly a bad idea if you return the result without managing the return.

In this type of class, the key is to understand that we need to be as simple as possible,
a manager (as a true Manager as described in the 12 rules of OOP) need to manage something, in the perfect
world (we're not living in it sadly), you SHOULD have a manager by entity, that's rule, respect it !

As you saw, we can now dive deeper into the service system of Symfony, you have a basic knowledge of how
to inject the dependencies and how to retrieve it, now it's time to manage bigger things.

# Case II - Entity lifecyle


