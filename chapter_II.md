# Chapter I

In this chapter, let's talk about the evolution of the service, the API
case and the dependencyInjection system in a more precise way.

## Introduction

In the last chapter, we've build a simple CRUD service, our logic is
stored inside a manager and this last one could accept a lot of things.

This approach is good and probably faster that all you have
probably tested but there's a major problem, how can we build the
complete process of request->response inside a manager ?

And first, question, is that his purpose ?

## Case I - A Manager isn't always the answer !

As you may understand, a manager IS for most of the case, the answer but
there's a backdoor, this manager is linked to a single entity and
in a modern application, we can have a thousand entity stored ... Boring
history.

So, let's imagine we have a API running, in order to respect the rules
of a modern API (REST, SOAP ... what's that ?), we need to have a 'structured'
format for the response, today, an API can communicate with multiple format
like JSON, XML and event HTML (yes, that could happen), in this example,
let's use JSON, probably our favorite format.

To start, let's declare a new class inside a AppBundle\Managers\Api
folder and name it ApiArticleManager :

```php
<?php

namespace AppBundle\Managers\Api;

class ApiArticleManager
{

}
```

Simple things short, what do we need in order to build our API manager ?

Let's use Doctrine, the FormFactory and the RequestStack for the moment :

```yaml
services:

    api.article_manager
        class: AppBundle\Managers\Api\ApiArticleManager
            arguments:
                - '@doctrine.orm.entity_manager'
                - '@form.factory'
                - '@request_stack'
```

Alright, nice thing done, let's update our class :

```php
<?php

namespace AppBundle\Managers\Api;

class ApiArticleManager
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
}
```

Time to get serious, what the purpose of an API ?

In fact, the simplest goal is to return a response according to the
request send ...
Not so hard, we know how to return this from the controller ... STOP.

Bad idea, let's talk about your idea and imagine that
you've build an API and you return a response from the controller, that
could lead to a great idea but sadly, that's a bad one, in fact, that's
probably the baddest you've ever had, even if you think you could do more
bad.

To explain faster, a controller SHOULD return a response, that's fact but
in the case of an API, that's NOT the way to do it, why ?

Simply because the data and process are handled by the service|manager,
if you keep your idea in mind, we gonna need to pass the data to the
controller and expect him to send a Response according to the data,
if you know the HTTP protocol, for example, a PUT method COULD do 3 to
4 operations and return as much responses in a single request->response
process !

That's gonna be a massive variable process and the controller could be
overwhelmed by all this process, bad idea.

Our approach is to do the request->response process INSIDE our manager
and pass the different response to the controller, this way, he could
send the response according to the request and be as fast as possible.

Let's build a simple 'get a single article' method :

```php
<?php

namespace AppBundle\Managers\Api;

use Symfony\Component\HttpFoundation\JsonResponse;

class ApiArticleManager
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

    public function getSingleArticle()
    {
        $id = $this->request->getCurrentRequest()->get('id');

        $article = $this->doctrine->getRepository('AppBundle:Article')
                                  ->findOneBy([
                                    'id' => $id
                                  ]);

        if ($article) {
            return new JsonResponse(
                $article,
                ['message' => 'Resource found.'],
                Response::HTTP_OK
            );
        }

        return new JsonResponse(
            ['message' => 'Resource not found.'],
            Response::HTTP_NOT_FOUND
        );
    }
}
```

Alright, what's new here ?

In fact, nothing to dangerous, we simply find the article according to
the id send through the request, once the article is found, we return
a JsonResponse who contains the proper headers and the resource
(aka the article) and a simple message to validate the research.

If the resource isn't found, we return a JsonResponse again bu the 404
headers code.

Let's update our routes and controllers :

```yaml
api_article_single:
     path:    /api/article/{id}
     methods: 'GET'
     defaults: { _controller: AppBundle:Api:Home:getSingleArticle }
     requirements:
          id: \d+
```

Our controller :

```php
<?php

namespace AppBundle\Controllers\Api;

use Symfony\Bundle\FrameworkBundle\Controller\Controller;

class HomeController extends Controller
{
    public function newArticleAction()
    {
        return $this->get('api.article_manager')->getSingleArticle();
    }
}
```

And that's it !

In a single line, we found the article, return him if he's found and
return the proper response !

The key here is to understand that a response is always send and for
the first time, it's not the controller directly who do this process,
plus, we don't need to call a shortcut inside the controller, we simply
inject the "components" that we need into our manager and that's all !


