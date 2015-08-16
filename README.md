# Laravel 5 JSON API Transformer Package


[![Scrutinizer Code Quality](https://scrutinizer-ci.com/g/nilportugues/laravel5-jsonapi-transformer/badges/quality-score.png?b=master)](https://scrutinizer-ci.com/g/nilportugues/laravel5-jsonapi-transformer/?branch=master) [![SensioLabsInsight](https://insight.sensiolabs.com/projects/22db88f5-d061-4b32-bad1-4b806ac07318/mini.png)](https://insight.sensiolabs.com/projects/22db88f5-d061-4b32-bad1-4b806ac07318) 
[![Latest Stable Version](https://poser.pugx.org/nilportugues/laravel5-json-api/v/stable)](https://packagist.org/packages/nilportugues/laravel5-json-api) 
[![Total Downloads](https://poser.pugx.org/nilportugues/laravel5-json-api/downloads)](https://packagist.org/packages/nilportugues/laravel5-json-api) 
[![License](https://poser.pugx.org/nilportugues/laravel5-json-api/license)](https://packagist.org/packages/nilportugues/laravel5-json-api) 



## Installation

Use [Composer](https://getcomposer.org) to install the package:

```
$ composer require nilportugues/laravel5-json-api
```


## Laravel 5 / Lumen Configuration

**Step 1: Add the Service Provider**

Open up `bootstrap/app.php`and add the following lines before the `return $app;` statement:

```php
$app->register('NilPortugues\Laravel5\JsonApiSerializer\Laravel5JsonApiSerializerServiceProvider');
$app['config']->set('jsonapi_mapping', include('jsonapi.php'));
```

**Step 2: Add the mapping**

Create a `jsonapi.php` file in `bootstrap/` directory. This file should return an array returning all the class mappings.

An example as follows:


**Step 3: Usage**

For instance, lets say the following object has been fetched from a Repository , lets say `PostRepository` - this being implemented in Eloquent or whatever your flavour is:

```php
use Acme\Domain\Dummy\Post;
use Acme\Domain\Dummy\ValueObject\PostId;
use Acme\Domain\Dummy\User;
use Acme\Domain\Dummy\ValueObject\UserId;
use Acme\Domain\Dummy\Comment;
use Acme\Domain\Dummy\ValueObject\CommentId;

//$postId = 9;
//PostRepository::findById($postId); 

$post = new Post(
  new PostId(9),
  'Hello World',
  'Your first post',
  new User(
      new UserId(1),
      'Post Author'
  ),
  [
      new Comment(
          new CommentId(1000),
          'Have no fear, sers, your king is safe.',
          new User(new UserId(2), 'Barristan Selmy'),
          [
              'created_at' => (new \DateTime('2015/07/18 12:13:00'))->format('c'),
              'accepted_at' => (new \DateTime('2015/07/19 00:00:00'))->format('c'),
          ]
      ),
  ]
);
```

And a series of mappings, placed in `bootstrap/jsonapi.php`, that require to use *named routes* so we can use the `route()` helper function:

```php
<?php
//bootstrap/jsonapi.php
return [
    [
        'class' => 'Acme\Domain\Dummy\Post',
        'alias' => 'Message',
        'aliased_properties' => [
            'author' => 'author',
            'title' => 'headline',
            'content' => 'body',
        ],
        'hide_properties' => [

        ],
        'id_properties' => [
            'postId',
        ],
        'urls' => [
            'self' => route('get_post'),
            'comments' => route('get_post_comments'),
        ],
        // (Optional)
        'relationships' => [
            'author' => [
                'related' => 'self' => route('get_post_author'),
                'self' => 'self' => route('get_post_author_relationship'),
            ]
        ],
    ],
    [
        'class' => 'Acme\Domain\Dummy\ValueObject\PostId',
        'alias' => '',
        'aliased_properties' => [],
        'hide_properties' => [],
        'id_properties' => [
            'postId',
        ],
        'urls' => [
            'self' => 'self' => route('get_post'),
            'relationships' => [
                'comment' => route('get_comment_author_relationship'),
            ],
        ],
    ],
    [
        'class' => 'Acme\Domain\Dummy\User',
        'alias' => '',
        'aliased_properties' => [],
        'hide_properties' => [],
        'id_properties' => [
            'userId',
        ],
        'urls' => [
            'self' => route('get_user'),
            'friends' => route('get_user_friends'),
            'comments' => route('get_user_comments'),
        ],
    ],
    [
        'class' => 'Acme\Domain\Dummy\ValueObject\UserId',
        'alias' => '',
        'aliased_properties' => [],
        'hide_properties' => [],
        'id_properties' => [
            'userId',
        ],
        'urls' => [
            'self' => route('get_user'),
            'friends' => route('get_user_friends'),
            'comments' => route('get_user_comments'),
        ],
    ],
    [
        'class' => 'Acme\Domain\Dummy\Comment',
        'alias' => '',
        'aliased_properties' => [],
        'hide_properties' => [],
        'id_properties' => [
            'commentId',
        ],
        'urls' => [
            'self' => route('get_comment'),
        ],
        'relationships' => [
            'post' => [
                'self' => route('get_post_comments_relationship'),
            ]
        ],
    ],
    [
        'class' => 'Acme\Domain\Dummy\ValueObject\CommentId',
        'alias' => '',
        'aliased_properties' => [],
        'hide_properties' => [],
        'id_properties' => [
            'commentId',
        ],
        'urls' => [
            'self' => route('get_comment'),
        ],
        'relationships' => [
            'post' => [
                'self' => route('get_post_comments_relationship'),
            ]
        ],
    ],
];

```

The named routes belong to the `app/Http/routes.php`. Here's a sample for the routes provided mapping:

```php
$app->get(
  '/post/{postId}',
  ['as' => 'get_post', 'uses' => 'PostController@getPostAction']
);

$app->get(
  '/post/{postId}/comments',
  ['as' => 'get_post_comments', 'uses' => 'CommentsController@getPostCommentsAction']
);

//...
``` 

All of this set up allows you to easily use the `JsonApiSerializer` service as follows:

```php
<?php

namespace App\Http\Controllers;

use Acme\Domain\Dummy\PostRepository;
use NilPortugues\Api\JsonApi\Http\Message\Response;
use NilPortugues\Serializer\Serializer;
use Symfony\Bridge\PsrHttpMessage\Factory\HttpFoundationFactory;


class PostController extends \Laravel\Lumen\Routing\Controller
{
    /**
     * @var PostRepository
     */
    private $postRepository;

    /**
     * @param PostRepository $postRepository
     * @param Serializer $jsonApiSerializer
     */
    public function __construct(PostRepository $postRepository, Serializer $jsonApiSerializer)
    {
        $this->postRepository = $postRepository;
        $this->jsonApiSerializer = $jsonApiSerializer;
    }

    /**
     * @param int $postId
     *
     * @return \Symfony\Component\HttpFoundation\Response
     */
    public function getPostAction($postId)
    {
        $post = $this->postRepository->findById($postId);
        $json = $this->jsonApiSerializer->serialize($post);

        return (new HttpFoundationFactory())->createResponse(new Response($json));
    }
}
```


**Output:**

```
HTTP/1.1 200 OK
Cache-Control: private, max-age=0, must-revalidate
Content-type: application/vnd.api+json
```

```json
{
    "data": {
        "type": "message",
        "id": "9",
        "attributes": {
            "headline": "Hello World",
            "body": "Your first post"
        },
        "links": {
            "self": {
                "href": "http://example.com/posts/9"
            },
            "comments": {
                "href": "http://example.com/posts/9/comments"
            }
        },
        "relationships": {
            "author": {
                "links": {
                    "self": {
                        "href": "http://example.com/posts/9/relationships/author"
                    },
                    "related": {
                        "href": "http://example.com/posts/9/author"
                    }
                },
                "data": {
                    "type": "user",
                    "id": "1"
                }
            }
        }
    },
    "included": [
        {
            "type": "user",
            "id": "1",
            "attributes": {
                "name": "Post Author"
            },
            "links": {
                "self": {
                    "href": "http://example.com/users/1"
                },
                "friends": {
                    "href": "http://example.com/users/1/friends"
                },
                "comments": {
                    "href": "http://example.com/users/1/comments"
                }
            }
        },
        {
            "type": "user",
            "id": "2",
            "attributes": {
                "name": "Barristan Selmy"
            },
            "links": {
                "self": {
                    "href": "http://example.com/users/2"
                },
                "friends": {
                    "href": "http://example.com/users/2/friends"
                },
                "comments": {
                    "href": "http://example.com/users/2/comments"
                }
            }
        },
        {
            "type": "comment",
            "id": "1000",
            "attributes": {
                "dates": {
                    "created_at": "2015-08-13T21:11:07+02:00",
                    "accepted_at": "2015-08-13T21:46:07+02:00"
                },
                "comment": "Have no fear, sers, your king is safe."
            },
            "links": {
                "self": {
                    "href": "http://example.com/comments/1000"
                }
            }
        }
    ],
    "links": {
        "self": {
            "href": "http://example.com/posts/9"
        },
        "next": {
            "href": "http://example.com/posts/10"
        }
    },
    "meta": {
        "author": [
            {
                "name": "Nil Portugués Calderó",
                "email": "contact@nilportugues.com"
            }
        ]
    },
    "jsonapi": {
        "version": "1.0"
    }
}
```

#### Request objects

JSON API comes with a helper Request class, `NilPortugues\Api\JsonApi\Http\Message\Request(ServerRequestInterface $request)`, implementing the PSR-7 Request Interface. Using this request object will provide you access to all the interactions expected in a JSON API:

##### JSON API Query Parameters:

- &filter[resource]=field1,field2
- &include[resource]
- &include[resource.field1]
- &sort=field1,-field2
- &sort=-field1,field2
- &page[number]
- &page[limit]
- &page[cursor]
- &page[offset]
- &page[size]


##### NilPortugues\Api\JsonApi\Http\Message\Request

Given the query parameters listed above, Request implements helper methods that parse and return data already prepared.

```php
namespace NilPortugues\Api\JsonApi\Http\Message;

final class Request
{
    public function __construct(ServerRequestInterface $request) { ... }
    public function getQueryParam($name, $default = null) { ... }
    public function getIncludedRelationships($baseRelationshipPath) { ... }
    public function getSortFields() { ... }
    public function getAttribute($name, $default = null) { ... }
    public function getSortDirection() { ... }
    public function getPageNumber() { ... }
    public function getPageLimit() { ... }
    public function getPageOffset() { ... }
    public function getPageSize() { ... }
    public function getPageCursor() { ... }
    public function getFilters() { ... }
}
```

#### Response objects

The following PSR-7 Response objects providing the right headers and HTTP status codes are available:

- `NilPortugues\Api\JsonApi\Http\Message\ErrorResponse($json)`
- `NilPortugues\Api\JsonApi\Http\Message\ResourceCreatedResponse($json)`
- `NilPortugues\Api\JsonApi\Http\Message\ResourceDeletedResponse($json)`
- `NilPortugues\Api\JsonApi\Http\Message\ResourceNotFoundResponse($json)`
- `NilPortugues\Api\JsonApi\Http\Message\ResourcePatchErrorResponse($json)`
- `NilPortugues\Api\JsonApi\Http\Message\ResourcePostErrorResponse($json)`
- `NilPortugues\Api\JsonApi\Http\Message\ResourceProcessingResponse($json)`
- `NilPortugues\Api\JsonApi\Http\Message\ResourceUpdatedResponse($json)`
- `NilPortugues\Api\JsonApi\Http\Message\Response($json)`
- `NilPortugues\Api\JsonApi\Http\Message\UnsupportedActionResponse($json)`

Due to the current lack of support for PSR-7 Requests and Responses in Laravel, `symfony/psr-http-message-bridge` will bridge between the PHP standard and the Response object used by Laravel automatically, as seen in the Controller example code provided.


<br>
## Quality

To run the PHPUnit tests at the command line, go to the tests directory and issue phpunit.

This library attempts to comply with [PSR-1](http://www.php-fig.org/psr/psr-1/), [PSR-2](http://www.php-fig.org/psr/psr-2/), [PSR-4](http://www.php-fig.org/psr/psr-4/) and [PSR-7](http://www.php-fig.org/psr/psr-7/).

If you notice compliance oversights, please send a patch via [Pull Request](https://github.com/nilportugues/laravel5-jsonapi-transformer/pulls).


<br>
## Contribute

Contributions to the package are always welcome!

* Report any bugs or issues you find on the [issue tracker](https://github.com/nilportugues/laravel5-jsonapi-transformer/issues/new).
* You can grab the source code at the package's [Git repository](https://github.com/nilportugues/laravel5-jsonapi-transformer).


<br>
## Support

Get in touch with me using one of the following means:

 - Emailing me at <contact@nilportugues.com>
 - Opening an [Issue](https://github.com/nilportugues/laravel5-jsonapi-transformer/issues/new)
 - Using Gitter: [![Gitter](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/nilportugues/laravel5-jsonapi-transformer?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge)


<br>
## Authors

* [Nil Portugués Calderó](http://nilportugues.com)
* [The Community Contributors](https://github.com/nilportugues/laravel5-jsonapi-transformer/graphs/contributors)


## License
The code base is licensed under the [MIT license](LICENSE).
