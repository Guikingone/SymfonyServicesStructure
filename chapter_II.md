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
like JSON, XML and even HTML (yes, that could happen), in this example,
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

use Symfony\Component\HttpFoundation\Response;
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

In fact, nothing too dangerous, we simply find the article according to
the id send through the request, once the article is found, we return
a JsonResponse who contains the proper headers and the resource
(aka the article) and a simple message to validate the research.

If the resource isn't found, we return a JsonResponse again
but with the 404 headers code.

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

Ok, now, time to get serious about entity management in a API,
let's build a form process in order to save some data.

## Case II - Processing is everything, process everywhere !

Ok, let's be serious, we build a 'GET' method,
we can find a single article but how can we add a new one ?
In our manager, it was simple, we call the form and the client send the
data but here, the client gonna fly around a frontend application
(like VueJS, Angular ou even React ... Heck, don't like this one) and
he don't gonna click on our form directly ... Hum, hard time.

In fact, not so much, the main principal here is ... _logic sharing_

Yeah, i know, sound strange but that's the truth,
we gonna receive the data, don't show any form and cry if the data isn't
what we expected (not so much on the last part), so, how build this thing ?

Simply by using the same logic as earlier :

```php
<?php

namespace AppBundle\Managers\Api;

use Symfony\Component\HttpFoundation\Response;
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

    public function postNewArticle()
    {
        $article = new Article();

        // Grab the data passed through the request.
        $data = $this->request->getCurrentRequest()->request->all();

        $form = $this->form->create(ArticleType::class, $article, [
            'csrf_protection' => false,
        ]);
        $form->submit($data);

        if ($form->isSubmitted() && $form->isValid()) {
            // Search if a equivalent resource has been created.
            $data = $form->getData();
            $trick = $this->doctrine->getRepository('AppBundle:Article')
                                    ->findOneBy([
                                        'name' => $data->getName(),
                                    ]);

            if ($article) {
                return new JsonResponse(
                    [
                        'message' => 'Resource already found.',
                        'name' => $article->getName()
                    ],
                    Response::HTTP_SEE_OTHER
                );
            }

            $this->doctrine->persist($article);
            $this->doctrine->flush();

            return new JsonResponse(
                [
                    'message' => 'Resource created',
                    'name' => $article->getName()
                ],
                Response::HTTP_CREATED
            );
        }

        return new JsonResponse(
            [
                'message' => 'Form invalid',
            ],
            Response::HTTP_BAD_REQUEST
        );
    }
}
```

Yay, that's what I like to see in my manager, clear and nice form
processing, so what's the process here ?

In fact, we do more than just creating a new article, for my concern,
i'm not the one to blame, explanation :

If we follow to HTTP protocol (boring and eyes-murderer document called
the RFC ...) to the T, a post method can do|have multiple state, in fact,
event if REST API (Application Programming Interface) has no state or
SHOULDN'T have a state, the HTTP protocol say (and i listen to him) that
we need to pass by X state in X methods to perform true HTTP actions.

For the POST method, the protocol say that once the data is received and
managed, we need to search if a similar resource was created before with
the same attributes, if it's the case, we SHOULD redirect to this last one
and show the attributes found.

If no resources is found, we could create the resource using the data
and return a 201 (CREATED) headers code with the 'representation' of the
resource.

Last case, if the data are invalid then we should return a 400 (BAD_REQUEST)
headers code and a message showing the errors.

Well, that's the theory and what we implement here, just for showing you
the RIGHT way to do this but let's be clear, this example ISN'T complete,
in fact, we don't return ALL the attributes so ... Well, don't take this
for money maker.

So, what the process here ?

Simple things come with simple explanations ...

First, we instantiate a new Entity, normal process, then,
we create a new Form linked to this entity and we turn off the csrf_protection ...

What ?!

Yes, in the process of a API, the CSRF protection could be turned off,
two main reasons, first, the client don't click on our form, he just send
a HTTP request, no matter from where he came from, he need to be
authenticated. Second point, the authentication is probably a abstract
part in Symfony, here, we can use JWT for example and provide
a API Token who's gonna log our user, simple, fast et effective.

By turning off the CSRF protection, we gonna say to Symfony :

_Hey dude, what's up ? I want to send you a POST request, can I ?_

_Yes, for sure, show me your token ..._

_Don't have any, can I ? I'm just here for a simple request ..._

_So get out of my way little guy, i'm not a open API !_

You saw the deal ...

In this chapter, we don't gonna manage the token validation or his presence in the
request, we simply gonna build our manager, nothing more.

Then, we submit the data in order to check if no resource can be found with
the same name, if it's the case, we return the resource.
If nothing is found and the form is valid, we can submit the value
to the database and save them, good point.

In the case of the form isn't valid, we return a response who say that the
form isn't valid, captain obvious !

Simple process here, this process respect the HTTP rules to the T and
the REST principles to the single line, not to mention that every case is
covered !

Ok, let's build a method wo can manager the update of our resource :

```php
<?php

namespace AppBundle\Managers\Api;

use Symfony\Component\HttpFoundation\Response;
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

    public function postNewArticle()
    {
        $article = new Article();

        // Grab the data passed through the request.
        $data = $this->request->getCurrentRequest()->request->all();

        $form = $this->form->create(ArticleType::class, $article, [
            'csrf_protection' => false,
        ]);
        $form->submit($data);

        if ($form->isSubmitted() && $form->isValid()) {
            // Search if a equivalent resource has been created.
            $data = $form->getData();
            $trick = $this->doctrine->getRepository('AppBundle:Article')
                                    ->findOneBy([
                                        'name' => $data->getName(),
                                    ]);

            if ($article) {
                return new JsonResponse(
                    [
                        'message' => 'Resource already found.',
                        'name' => $article->getName()
                    ],
                    Response::HTTP_SEE_OTHER
                );
            }

            $this->doctrine->persist($article);
            $this->doctrine->flush();

            return new JsonResponse(
                [
                    'message' => 'Resource created',
                    'name' => $article->getName()
                ],
                Response::HTTP_CREATED
            );
        }

        return new JsonResponse(
            [
                'message' => 'Form invalid',
            ],
            Response::HTTP_BAD_REQUEST
        );
    }

    public function putSingleTricks()
    {
        $id = $this->request->getCurrentRequest()->get('id');

        $article = $this->doctrine->getRepository('AppBundle:Article')
                                   ->findOneBy([
                                       'id' => $id,
                                   ]);

        if (!$article) {
            $article = new Article();

            $data = $this->request->getCurrentRequest()->request->all();

            $form = $this->form->create(ArticleType::class, $tricks, [
                'csrf_protection' => false,
            ]);
            $form->submit($data);

            if ($form->isSubmitted() && $form->isValid()) {
                // Search if a equivalent resource has been created.
                $object = $form->getData();
                $article = $this->doctrine->getRepository('AppBundle:Article')
                                          ->findOneBy([
                                              'name' => $object->getName(),
                                          ]);

                if ($article) {
                    return new JsonResponse(
                        [
                            'message' => 'Resource already found.',
                            'name' => $article->getName()
                        ],
                        Response::HTTP_SEE_OTHER
                    );
                }

                $this->doctrine->persist($article);
                $this->doctrine->flush();

                return new JsonResponse(
                    [
                        'message' => 'Resource created',
                        'name' => $article->getName()
                    ],
                    Response::HTTP_CREATED
                );
            }
        }

        $data = $this->request->getCurrentRequest()->request->all();

        $form = $this->form->create(ArticleType::class, $article, [
            'csrf_protection' => false,
        ]);
        $form->submit($data);

        if ($form->isSubmitted() && $form->isValid()) {
            $this->doctrine->flush();

            return new JsonResponse(
                [
                    'message' => 'Resource updated',
                    'name' => $article->getName()
                ],
                Response::HTTP_OK
            );
        }

        return new JsonResponse(
            [
                'message' => 'Resource not updated',
            ],
            Response::HTTP_NO_CONTENT
        );
    }
}
```

Ok, that's a little bit more complex than before, first, let's explain the PUT
method :

- First, we need to grab the data passed through the request and find the
resource linked to the data using his id, then, we need to check if a resource
can be found using the id passed.

- In the case that no resource can be found using the id, we need to create a new
one using the informations passed through the request, that a other constraints of
the PUT method, if no resource match, we need to create one.

_As always, we check if the form is valid with the current data._

- In the case of a resource created with the current data, we need to
check if the resource isn't already persisted into the BDD, this point
could be strange but it's always logic, the data passed through the
request could contain 'keys' that already in the BDD (like the name
for example), we need to be sure that only **one** resource contain the
data in the BDD. If the form is valid, we send the resource to the BDD.

- In the case that the resource exist and can't be found in the BDD, we
grab the request data and update the resource with the data, simple story !

_Nota bene : In this case, we take care that every 'input' of the form is send !_

Alright random citizen ! Time to get serious !

As you saw, we can update a whole resource using the data but how can we
_update_ a single part of this resource ? How can we update just the name ?

## Case III - I'm flying like a butterfly !






