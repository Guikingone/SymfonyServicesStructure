# Chapter I

In this chapter, let's talk about the service system of Symfony, in more precise way, let's talk about the logic of using services in symfony.


## Introduction

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

# Case II - Entity lifecycle

Alright random citizen, what's up ?

Kidding, just want to know if the last case was clear ?

**_glory cry_**

Ok, that's what I want to ear.

In this case, we gonna manage the entity through his _entire_ lifecycle, I mean,
like a CRUD but in a smaller size, so, let's build our own CRUD and starting by our CREATE method :

```php
<?php

namespace AppBundle\Managers;

use Doctrine\ORM\EntityManager;

// Entity
use AppBundle\Entity\Article;

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

    public function newArticle()
    {
        $article = new Article();

        // Build our first entity !
    }
}
```

So, we instantiate the entity but how can the visitor build a new Entity without form ? Heck ...

Like many developers, you could say that a "manager" shouldn't build a form and manager the form lifecycle
of the entity, lonely soul ...

In the case of the manager, let's think about your first Manager you've ever build, he retrieve the entity
and we can easily do harder things inside but, we don't do it, simply because we don't need it at this time.

Now, time to dive deeper into this ballet of beauty called the FormFactory class !

By default, when you need to build a form inside a controller, you do this :

```php
<?php

namespace AppBundle\Controllers;

use Symfony\Bundle\FrameworkBundle\Controller\Controller;

class AppController extends Controller
{
    public function newArticleAction(Request $request)
    {
        $article = new Article();
        $form = $this->createForm(ArticleType::class, $article);
        // Form processing
    }
}
```

Let's be clear, that a terribly bad idea !

But why ?

Simply because the Controller of Symfony is probably the ugliest idea (sorry Mr Potencier) and probably
the hardest component to unit test, yes, think about testing, if your class depends
or extends multiple class (in the case of a class who extends an other who implements 3 classes),
who's the class responsible for the shortcut you saw earlier ?

By default, that's Controller but how can we know this without looking into the class ?

The biggest problem is about using a shortcut, that's not a bad idea but when it come to testing the
method, who know the class who implements the method ?

In the case of a manager, we don't have any shortcut and that's a fucking great idea, no shortcut, real
development !

So, let's build our own form process inside of our manager :

```php
<?php

namespace AppBundle\Managers;

use Doctrine\ORM\EntityManager;

// Entity
use AppBundle\Entity\Article;

class EntityManager
{
    /** @var EntityManager */
    private $doctrine;

    /** @var FormFactory */
    private $form;

    public function __construct(
        EntityManager $doctrine,
        FormFactory $form
    ) {
        $this->doctrine = $doctrine;
        $this->form = $form;
    }

    // This must return an array of all the articles stored into the BBD.
    public function getArticles()
    {
        return $this->doctrine->getRepository('AppBundle:Article')->getAllArticles();
    }

    public function newArticle()
    {
        $article = new Article();

        $form = $this->form->create(ArticleType::class, $article);
        // Form processing
    }
}
```

Ok, that's the right way to do it, problem, our form need the request (the HTTP request) to be able to
perform the process of conditioning the data passed thought the request into a valid data for the form.
To do this, we can call the Request class (from the HTTPFoundation component) or simply pass the RequestStack
(The package who contains a lot of request) into our manager and grab the currentRequest in order to find
the data, let's do this way :

```php
<?php

namespace AppBundle\Managers;

use Doctrine\ORM\EntityManager;

// Entity
use AppBundle\Entity\Article;

class EntityManager
{
    /** @var EntityManager */
    private $doctrine;

    /** @var FormFactory */
    private $form;

    /** @var RequestStack */
    private $request;

    public function __construct(
        EntityManager $doctrine,
        FormFactory $form,
        RequestStack $request
    ) {
        $this->doctrine = $doctrine;
        $this->form = $form;
        $this->request = $request;
    }

    // This must return an array of all the articles stored into the BBD.
    public function getArticles()
    {
        return $this->doctrine->getRepository('AppBundle:Article')->getAllArticles();
    }

    public function newArticle()
    {
        $request = $this->request->getCurrentRequest->request->all();

        $article = new Article();

        $form = $this->form->create(ArticleType::class, $article);
        $form->handleRequest($request);

        // Validate the form data

    }
}
```
That's way better and for sure, our code is totally dependant on the Symfony component (I mean, not totally)
by the way that we call the request class of Symfony but we can easily inject the PSR-7 bridge of Zend and
use it or even the Request from Laravel, it's a simple class injection !

Now that we have a request and a form, let's manage the data and the submission of the form :

```php
<?php

namespace AppBundle\Managers;

use Doctrine\ORM\EntityManager;

// Entity
use AppBundle\Entity\Article;

class EntityManager
{
    /** @var EntityManager */
    private $doctrine;

    /** @var FormFactory */
    private $form;

    /** @var RequestStack */
    private $request;

    public function __construct(
        EntityManager $doctrine,
        FormFactory $form,
        RequestStack $request
    ) {
        $this->doctrine = $doctrine;
        $this->form = $form;
        $this->request = $request;
    }

    // This must return an array of all the articles stored into the BBD.
    public function getArticles()
    {
        return $this->doctrine->getRepository('AppBundle:Article')->getAllArticles();
    }

    public function newArticle()
    {
        $request = $this->request->getCurrentRequest->request->all();

        $article = new Article();

        $form = $this->form->create(ArticleType::class, $article);
        $form->handleRequest($request);

        if ($form->isSubmitted() && $form->isValid()) {
            $this->doctrine->persist($article);
            $this->doctrine->flush();
        }

        return $form;
    }
}
```

What the hell is that approach !?

In fact, simple things come with some magic, let's decompose our code.

First, we verify if the form is valid and submitted, simple approach to validate the client values,
then, we call Doctrine (aka the EntityManager) to persist the entity (aka save in a array before she's gonna
be saved into BDD) then we flush (aka save in the BDD) the entity, the cool things here it's that Doctrine
gonna handle all the process for us, some magic !

Last but not least, we return the form ... Why ?

Simply because we gonna show the form to the client (in a Twig view for example) and we need to access to
the form variable to call the {{ form_start() }} and {{ form_end() }} methods, to do this, in the controller
we simply pass the variable to the view array, here, we don't have access to this array so we return the
variable and we gonna grab this last one inside our controller, let's build some magic things !

So, how can we call the form inside our Controller ? First, let's update our services.yml :

```yaml
services:

    app.entity_manager
        class: AppBundle\Managers\EntityManager
            arguments:
                - '@doctrine.orm.default_entity_manager'
                - '@form.factory'
                - '@request_stack'
```

Simple approach of injecting the class, you know how to do this ...

So, let's update our Controller then :

```php
<?php

namespace AppBundle\Controllers;

use Symfony\Bundle\FrameworkBundle\Controller\Controller;

class AppController extends Controller
{
    public function newArticleAction(Request $request)
    {
        $form = $this->get('app.entity_manager')->newArticle();

        return $this->render('yourview.html.twig', [
            'form' => $form
        ]);
    }
}
```

That's what we call a "fucking cleaner" controller (sorry for the joy, we love this approach !), by moving
the form processing into our manager (aka service), we have move the logic outside of the C of MVC, simple,
effective and logic, in fact, we've build what we call a MSCV pattern for :

- Model => Our Entity
- Services => Our Manager
- Controller => Our Controller (magic isn't it ?)
- View => Our Twig View (because Twig is too cute !)

So, what's next ?

Not so much, first, let's talk about this approach, why and what the consequences ?

In fact, moving the "heavy" process from our controller to our service help us to improve performances
(like Mr Potencier, we love having fast web application !) and build what we call "the art of logic",
this approach is build from a simple vision, if a class is linked to a framework,
she's not really simple to test and building something with her could be hard, so,
let's build a service (or a manager) to save logic from him and have a better time testing our logic.

Simple logic, magic approach !

But, there's a single problem, we inject the request inside our service, that's not clearly "clean" ...

In fact, the request is a simple class in Symfony, like ... Well, like a class that you build, so,
as you know the controller "manage" the request ... Wrong !

The controller ISN`T the manager of the request, the kernel is, in fact,
the HTTPKernel manager the request, response, exceptions,
controller and so on by dispatching "events" (we come on this later), this way, the router grab the request
and analyse what's inside, this way, he call the ArgumentResolver to analyse the arguments passed through,
after this, the ControllerResolver is called and find the controller linked to this request
(aka route asked) by the client, once this is done, the request "pass" from the controller to the HTTPKernel
again and this last one send a event call kernel.response (yes, very hard ...) in order to send the response
returned from the controller.

This way, the request isn`t linked to the controller, this last one is a simple 'point' on the road
of the request and he SHOULDN'T manage the request, in fact, it's our logic (aka services here) who need the
request to perform something, this way, the request is passed directly to the services who need her
and our application goes faster (yes, that's life changer isn't it !?) do to the fact that our request
is "manage" and "solved" before even been manage by our Controller, this way, everything goes faster !

So, that was a simply way of explaining the hard process behind our logic, now, you have a manager who can
CREATE and GET (aka find) a simple entity, let's build the UPDATE AND DELETE methods !

# Case III - A manager should manage !
