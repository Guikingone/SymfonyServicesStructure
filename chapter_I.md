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
write into the log, the so-called class 'logger' is available in Symfony via le DependencyInjection component, more in depth,
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
        return $this->get('app.logger'->write();
    }
  }
```
This controller call the service saved sooner and especially, the write method, this way, we don't need to allow PHP to
save a variable into memory, this way, better performances !

**_Just for precision, this code isn't completely valid by the way that we don't return a Response, who's mandatory in Symfony
but we simply want to show the process._**

Alright, now that we have a service and we call him, how can we improve this one ?

## Case I - DependencyInjection power !



