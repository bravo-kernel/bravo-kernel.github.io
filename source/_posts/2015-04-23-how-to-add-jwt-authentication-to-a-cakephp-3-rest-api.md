title: "How to add JWT Authentication to a CakePHP 3 REST API"
date: 2015-04-20 07:22:32
categories:
  - CakePHP
tags:
- cakephp
- cakephp3
- rest
- api
- security
- jwt
---
In this follow-up post to
[How to prefix route a CakePHP 3 REST API](/2015/04/how-to-prefix-route-a-cakephp-3-rest-api/)
we will implement [JSON Web Token](http://jwt.io/) (JWT) authentication.

[![Run in Postman](https://run.pstmn.io/button.svg)](https://app.getpostman.com/run-collection/197398a609a6d233a8c2)

To prevent (yet another) partial/pointless JWT tutorial we will provide you with step-by-step
instructions:

- describing a full blown, real world implementation
- usable as drop-in code for (almost) any CakePHP 3 application requiring API authentication
- spiced up with background information to help you understand the JWT concept

> **Important**: please remember to use SSL/TLS encrypted connections for ALL API traffic
> to prevent man in the middle attackers from seeing and stealing the tokens.

During this tutorial you will:

+ add some basic user data to the application
+ enable password hashing
+ add the JWT plugin
+ update your prefix route
+ enable JWT Authentication for API resources
+ create the API UsersController
+ implement user registration using the API
+ sanity check your first JWT token
+ implement JWT token requests
+ test JWT protected API resources

## Requirements

To complete this tutorial you will need a fully configurable REST client like:

+ the [Postman](https://chrome.google.com/webstore/search/postman?hl=en&_category=apps) plugin for Chrome (recommended)
+ the [RESTClient](https://addons.mozilla.org/nl/firefox/addon/restclient/) plugin for Firefox
+ the [Cocoa REST Client](http://mmattozzi.github.io/cocoa-rest-client/) if you are on a Mac

## Before We Begin

This is part four of the CakePHP 3 REST API tutorial series:

1. [How to build a CakePHP 3 REST API in minutes](/2015/04/how-to-build-a-cakephp-3-rest-api-in-minutes/)
2. [How to use a CakePHP 3 REST API](/2015/04/how-to-use-a-cakephp-3-rest-api/)
3. [How to prefix route a CakePHP 3 REST API](/2015/04/how-to-prefix-route-a-cakephp-3-rest-api/)
4. How to add JWT Authentication to a CakePHP 3 REST API
5. [How to make your CakePHP 3 API produce JSON API](/2017/03/how-to-make-your-cakephp-3-api-produce-jsonapi/)
6. [How to use a CakePHP API as the data backend for Ember in 30 minutes](/2017/03/how-to-use-a-cakephp-api-as-the-data-backend-for-ember-in-30-minutes/)

Before starting this tutorial either:

+ complete the previous posts
+ start fresh by using these
[end-state application sources](https://github.com/bravo-kernel/application-examples/tree/master/blog-how-to-prefix-route-a-cakephp3-rest-api),
composer installing and running the database migration


## 1. Introduction

The web is already filled with information about JSON Web Token (JWT) Authentication so we will not
duplicate it here but in a nutshell it allows authenticating users against a single token instead
of the more commonly used username/password.

As a side effect our API will benefit from some (very cool) additional JWT functionality like:

+ No more need for sessions
+ No more need to protect our API against Cross-Site Request Forgery (CSRF)
+ Support for granular security through the use of JWT scopes

## 2. Adding Users To The Application

### Populate the database

Download this CakePHP
[database migration file](https://github.com/bravo-kernel/application-examples/blob/master/blog-how-to-add-jwt-authentication-to-a-cakephp-3-rest-api/config/Migrations/20150420112310_users.php)
to your ``config/Migrations`` directory.

Now run the following command inside your application's root directory to
create the ``users`` table:

```bash
bin/cake migrations migrate
```

### Generate the basic controller, entity, table and views

To prepare for testing "basic" HTML access generate the required controller, entity, table and views
by running the following command inside your application's root directory:

```bash
bin/cake bake all Users
```

### Configure Password Hashing

CakePHP 3 comes with a convenient [PasswordHasher](http://book.cakephp.org/3.0/en/controllers/components/authentication.html#hashing-passwords)
that will automatically encrypt user passwords using the very strong
[bcrypt](http://en.wikipedia.org/wiki/Bcrypt) hashing algorithm.
To enable password hashing for your application make sure to add both the class and the method shown below to ``src/Model/Entity/User.php``:

```php
use Cake\Auth\DefaultPasswordHasher;

    protected function _setPassword($password)
    {
        return (new DefaultPasswordHasher)->hash($password);
    }
```

### Verify

If things went well you should now be able to:

+ browse to ``http://cake3api.app/users``
+ create a new user
+ see the created user with hashed password

<br />

{% asset_img browser-users-index.png 'Users index' %}

## 3. Adding the JWT Plugin

Run the following command inside your application's root directory to
composer install the [JwtAuth plugin](https://github.com/ADmad/cakephp-jwt-auth):

```bash
composer require admad/cakephp-jwt-auth
```

Now run the following command to make your application use the plugin:

```php
bin/cake plugin load ADmad/JwtAuth
```

## 4. Updating The Prefix Route

The API prefix route created during the previous tutorial needs updating:

+ to enable the ``Api\UsersController`` for API usage
+ to connect custom action ``/register`` to standard [CRUD Plugin]() action ``/add``
+ to automatically create routes for the non-standard ``/register`` and ``/token`` actions

**Pro tip:** we connect the ``/register`` action so we can simply extend the CRUD Plugin ``add()``
method and benefit of already available logic like validation and response codes instead of having to
reinvent the wheel.

Make sure to update the ``api`` prefix route in ``config/routes.php`` to resemble:

```php
Router::prefix('api', function ($routes) {
    $routes->extensions(['json', 'xml']);
    $routes->resources('Cocktails');
    $routes->resources('Users');
    Router::connect('/api/users/register', ['controller' => 'Users', 'action' => 'add', 'prefix' => 'api']);
    $routes->fallbacks('InflectedRoute');
});
```

## 5. Enabling JWT Authentication

To enable JWT Authentication for all API resources extend the
``src/Controller/Api/AppController.php`` file created during the previous tutorial with the following
``initialize`` method so the file looks similar to:

```php
<?php
namespace App\Controller\Api;

use Cake\Controller\Controller;
use Cake\Event\Event;

class AppController extends Controller
{

    use \Crud\Controller\ControllerTrait;

    public function initialize()
    {
        parent::initialize();

        $this->loadComponent('RequestHandler');
        $this->loadComponent('Crud.Crud', [
            'actions' => [
                'Crud.Index',
                'Crud.View',
                'Crud.Add',
                'Crud.Edit',
                'Crud.Delete'
            ],
            'listeners' => [
                'Crud.Api',
                'Crud.ApiPagination',
                'Crud.ApiQueryLog'
            ]
        ]);
        $this->loadComponent('Auth', [
            'storage' => 'Memory',
            'authenticate' => [
                'Form' => [
                    'scope' => ['Users.active' => 1]
                ],
                'ADmad/JwtAuth.Jwt' => [
                    'parameter' => 'token',
                    'userModel' => 'Users',
                    'scope' => ['Users.active' => 1],
                    'fields' => [
                        'username' => 'id'
                    ],
                    'queryDatasource' => true
                ]
            ],
            'unauthorizedRedirect' => false,
            'checkAuthIn' => 'Controller.initialize'
        ]);
    }
}
```

**Notes:**

- we use `Memory` based non-persistent storage for the authenticated user
(instead of Cake's session based default)
- FormAuthenticate MUST be included here or AuthComponent will not be able to validate the
posted (non-JWT) JSON credentials during the ``/token`` action
- by enabling `queryDataSource` the `sub` field in the JWT token will be used
to query the database for user information (using the User model)
- `checkAuthIn` makes user information available in all Controller
`beforeFilter()` functions

### Verify Authentication Is Enabled

To verify your API resources now actually require authentication query
``http://cake3api.app/api/cocktails.json``.

Should return Status Code 401 (Unauthorized) with a JSON response body similar to:

```json
{
    "success": false,
    "data": {
        "message": "You are not authorized to access that location.",
        "url": "\/api\/cocktails.json",
        "code": 401
    }
}
```

## 6. Creating the API UsersController

We will now create a UsersController responsible for handling all authentication in the ``Api`` namespace:

+ using standard [AuthComponent allow logic](http://book.cakephp.org/3.0/en/controllers/components/authentication.html#making-actions-public)
to allow non-authenticated access to the ``/add`` and ``/token`` actions
+ already containing all required ``use`` statements required later on

Create new file ``src/Controller/Api/UsersController`` with the following code:

```php
<?php
namespace App\Controller\Api;

use Cake\Event\Event;
use Cake\Network\Exception\UnauthorizedException;
use Cake\Utility\Security;
use Firebase\JWT\JWT;

class UsersController extends AppController
{
    public function initialize()
    {
        parent::initialize();
        $this->Auth->allow(['add', 'token']);
    }
}
```

**Note:** Auth-allowed actions MUST be set inside `initialize()`
because we enabled the `checkAuthIn` configuration option.

## 7. Implementing API User Registration

### How it works

User registration through the API does not require JWT authentication and is basically a matter of
posting valid JSON data to the ``/add`` action in our UsersController so the [CRUD Plugin](xx)
can handle validation and creating the user record.

If the user is created succesfully a JSON 201 response (Created) will be returned with a response
body containing:

+ the id of the new user
+ a ``token`` field containing the new user's JWT token

<br />

{% asset_img jwt-primer-registration.png 'JWT primer: user registration' %}


### Create the /register  action

Because the CRUD plugin normally only returns the id of the new record we will add the JWT token
to the JSON response body by extending the ``add()`` method with some custom CRUD
[afterSave](http://crud.readthedocs.org/en/latest/actions/add.html#crud-aftersave)
and [serialize](http://crud.readthedocs.org/en/latest/actions/add.html#serialize) logic.

To implement user registration add the following ``add()`` method to ``src/Controller/Api/UsersController.php``:

```php
    public function add()
    {
        $this->Crud->on('afterSave', function(Event $event) {
            if ($event->subject->created) {
                $this->set('data', [
                    'id' => $event->subject->entity->id,
                    'token' => JWT::encode(
                        [
                            'sub' => $event->subject->entity->id,
                            'exp' =>  time() + 604800
                        ],
                    Security::salt())
                ]);
                $this->Crud->action()->config('serialize.data', 'data');
            }
        });
        return $this->Crud->execute();
    }
```

**Note:** even though this is not required we are adding the [JWT exp claim](http://self-issued.info/docs/draft-ietf-oauth-json-web-token.html#expDef)
to the token payload so the token will expire after one week, effectively forcing the user to request
a new unique token using the ``/token`` action.

**Important:** your JWT token (and thus the user information) will NOT
contain an `id` field if you choose to  disable the `queryDataSource` option.
This might might break code depending on the presence of an `id` field but
is easily solved  by manually adding the `id` field to the JWT token
(below `exp` in the code above).

### Verify User Registration

To verify your setup register a new user by posting JSON data to your API using:

+ URL ``http://cake3api.app/api/users/register``
+ **HTTP Method** ``POST``
+ **Accept Header** ``application/json``
+ **Content-Type Header** ``application/json``
+ Body data in (absolutely) correct JSON format

{% asset_img api-request-register.png 'API Request Headers for register action' %}

Should return Status Code 201 (Created) with a JSON response body containing
the user id and JWT token similar to:

```json
{
    "success": true,
    "data": {
        "id": 2,
        "token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpZCI6Mn0.q2chPMiKRzwrO3v48fi90HyJPHDLOXtwEKr7EcU3GPk"
    }
}
```

## 8. Sanity Checking JWT Tokens

Now that you have received your first JWT token it might be a good time to verify that your token
is valid by:

+ browsing to [http://jwt.io/](http://jwt.io/)
+ pasting your token in the ``Encoded`` field
+ replacing the secret value with the ``Salt`` value found in your ``config/app.php``

If things went well you should see a **green** success message along with the user id and JWT exp
claim as stored in the token:

<br />

{% asset_img jwt-io-signature-verification.png 'jwt.io successful signature verification' %}


## 9. Implementing JWT Token Requests

### How it works

Users can request their JWT token by JSON posting their username and password to the ``/token`` action
after which AuthComponent will use FormAuthenticate (and thus not JwtAuth) to
validate the credentials.

If validation is successful a JSON 200 response (Success) will be returned with a response
body containing the JWT token.

<br />

{% asset_img jwt-primer-token.png 'JWT primer: token request' %}

### Create the /token action

To implement token requests add the following ``token()`` method to ``src/Controller/Api/UsersController.php``:

```php
    public function token()
    {
        $user = $this->Auth->identify();
        if (!$user) {
            throw new UnauthorizedException('Invalid username or password');
        }

        $this->set([
            'success' => true,
            'data' => [
                'token' => JWT::encode([
                    'sub' => $user['id'],
                    'exp' =>  time() + 604800
                ],
                Security::salt())
            ],
            '_serialize' => ['success', 'data']
        ]);
    }
```

### Verify Token Request

To verify your setup try requesting a token for the newly created user by posting JSON data to your API using:

+ URL ``http://cake3api.app/api/users/token``
+ **HTTP Method** ``POST``
+ **Accept Header** ``application/json``
+ **Content-Type Header** ``application/json``
+ Body data with username and password in (absolutely) correct JSON format

{% asset_img api-request-token.png 'API Request Headers for token action' %}

Should return Status Code 200 (Success) with a JSON response body containing
only the JWT token similar to:

```json
{
    "success": true,
    "data": {
        "token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpZCI6Mn0.q2chPMiKRzwrO3v48fi90HyJPHDLOXtwEKr7EcU3GPk"
    }
}
```

## 10. Testing JWT Authentication

> ** Before you begin testing:** please be aware that some servers (like Apache) don't automatically 
> populate $_SERVER['HTTP_AUTHORIZATION'] even when the Authorization header is set. Make sure to 
> first follow [these instructions](https://github.com/ADmad/cakephp-jwt-auth#working)
> if the tests below are not functioning as described.


### How it works

When accessing an API resource that requires authentication the JWT Plugin will look for a token
in the ``Authorization`` header and will validate it using the ``Salt`` value used by your application.

If validation is successful a JSON 200 response (Success) will be returned with application produced body.


<br />

{% asset_img jwt-primer-authentication.png 'JWT primer: authentication process' %}

**Notes:**

+ there is no need to create extra code, all JWT authentication logic is already present in the plugin
+ the JWT Plugin also supports passing the token as a query string parameter named ``_token`` (not described for brevity)
+ the ``Authorization`` header **MUST** contain a Bearer Token which is part of the OAuth V2 standard and should look like:

```
Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpZCI6Mn0.q2chPMiKRzwrO3v48fi90HyJPHDLOXtwEKr7EcU3GPk
```

### Verify Authenticated Access

To verify successful authentication is processed as expected retrieve the list of protected cocktails from your API by using:

+ URL ``http://cake3api.app/api/cocktails``
+ **HTTP Method** ``GET``
+ **Accept Header** ``application/json``
+ **Authorization Header** containing ``Bearer {YOUR-JWT-TOKEN}``

{% asset_img api-request-authenticate.png 'API Request Headers for resources requiring authentication' %}

Should return Status Code 200 (Success) with the familiar JSON cocktails response body:

```json
{
    "success": true,
    "data": [
        {
            "id": 1,
            "name": "Cosmopolitan",
            "description": "Vodka based"
        },
        {
            "id": 2,
            "name": "Margarita",
            "description": "Tequila based"
        },
        {
            "id": 3,
            "name": "Mojito",
            "description": "Rum based"
        }
    ]
}
```

### Verify Unauthenticated Access

To verify unsuccessful authentication is processed as expected retrieve the list of protected cocktails
by using the exact same query but **this time removing** the ``Authorization`` header.

Should instantly return Status Code 401 (Unauthorized) with a JSON response body similar to:

```json
{
    "success": false,
    "data": {
        "message": "You are not authorized to access that location.",
        "url": "\/api\/cocktails.json",
        "code": 401
    }
}
```

## Additional reading

+ Follow-up tutorial [How to make your CakePHP 3 API produce JSON API](/2017/03/how-to-make-your-cakephp-3-api-produce-jsonapi/)
+ [Git repository](https://github.com/bravo-kernel/application-examples/tree/master/blog-how-to-add-jwt-authentication-to-a-cakephp-3-rest-api) with working end state application as produced by this tutorial
+ The [CakePHP JWT Plugin](https://github.com/ADmad/cakephp-jwt-auth) on Github
+ The [PHP JWT Library](https://github.com/firebase/php-jwt) on Github
+ The [CakePHP 3 Book](http://book.cakephp.org/3.0/en/index.html) and [CakePHP 3 API](http://api.cakephp.org/3.0/) documentation

<em>Hat tip to CakePHP Core Developer and JWT Plugin creator [ADmad](https://github.com/ADmad/) for
helping improve this tutorial.</em>
