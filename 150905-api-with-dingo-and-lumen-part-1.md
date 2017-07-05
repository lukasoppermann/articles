---
series: Building APIs with dingo & lumen;1
title: The basics of setting up dingo api with the lumen framework
tags: tag1, tag2
author: Lukas Oppermann
category: code
description: Learn how to build a php API with dingo & lumen\: setup and configuration.
preview: In the introduction to this series we will look into setting up dingo api with lumen on a homestead server, as well as setting up a testing environment with phpunit.
---

> In this series we will write a well tested api using [lumen](http://lumen.laravel.com/) and the [dingo/api](https://github.com/dingo/api) package.

## Install lumen & dingo/api

I am assuming you have composer installed on your machine and a basic knowledge of how to use the command line.
So lets start installing lumen via composers `create-project` command. This command creates a new folder with the name of your project, so make sure to `cd` into your `code` directory (or wherever you store all your projects) first. Once the installation is done we can `cd` into the newly created project folder and require the `dingo/api` package using composer.

```bash
cd ~/code
composer create-project laravel/lumen --prefer-dist myLumenApi
cd myLumenApi
composer require dingo/api:1.0.x@dev --prefer-dist
```

Before we can start to build our api we need to do a little configuration, so open `bootstrap/app.php` in your text editor of choice and uncomment line 5. This will make it possible to use an `.env` file, which you can use in your development environment to simulate environment variables. You can read more about `.env` files in the [Lumen docs](http://lumen.laravel.com/docs/installation#environment-configuration).

```php
Dotenv::load(__DIR__.'/../');
```

Now uncomment line 24 to enable [Eloquent](http://laravel.com/docs/5.1/eloquent), Laravels and Lumens excellent ORM. You could instead implement a repository pattern to abstract your database layer, but I would only recommend it if you are at least 50% sure you will need this, or if your api is really huge. Otherwise you just add an overhead you might never need, and if you do need it,it'snot such a big pain to refactor a couple controllers to use a repository.

```php
$app->withEloquent();
```

Now we only need to load dingos service provider, which I would add at line 80. This pulls in the magic of the `dingo/api` packages for us.

```php
$app->register(Dingo\Api\Provider\LumenServiceProvider::class);
```

## Setting up homestead
I expect you to have a working version of homestead on your machine, if not, I wrote an article explaining [how to setup homestead](/blog/150917-easy-development-using-homestead). Once you are done we need to add a new domain to it, so open `/etc/hosts` by running `open /etc/hosts` in your command line and add a dev domain to it.

```bash
192.168.10.10  api.mylumenapi.app
```

Afterwards we need to add this to homestead so run `homestead edit` from the command line and add the following entry under the `sites` section.

```bash
sites:
    - map: api.mylumenapi.app
      to: /home/vagrant/Code/mylumenapi/public
```

You might need to destroy your vm and restart it, to get it to pick up the new domain. Do this by running the commands below in your command line. If everything went according to plan, you should be able to see the lumen welcome page when accessing your domain `api.mylumenapi.app`.

```bash
homestead destroy
homestead up
```

## Configure dingo/api
The `dingo/api` package lets you change much ofit'sbahaviour by changing the settings via the environment variables, so open your `.env` file and add the variables described below. The [dingo/api documentation](https://github.com/dingo/api/wiki/Configuration) is actually pretty good if you want to read more about those configurations.

**API_STANDARDS_TREE=vnd**   
If your api is publicly available or at any point will be, use the vendor tree `vnd`. The personal tree `prs`, is meant for not distributed projects only. I recommend to always use the `vnd` tree, since it does not hurt you if your api stays private.

**API_SUBTYPE=yourVendorName**
This is the name of your project or application. Github for e.g. uses `github`. It will be part of the accept header, which will look like `vnd.yourVendorName.v1+json`.

**API_PREFIX=/**
For dingo to work you need to provide either a prefix or a subdomain, but subdomain routing is only supported by laravel, not lumen. Since our app is a standalone api we can use a prefix of `/` which means our base url for the api is our urls (api.mylumenapi.app).

**API_VERSION=v1**
The version option specifies our default version, which is used whenever a request does not specify a version. This should always track your most recent version. Make sure to advise your api user to always specify an api version so they are not suprised by breaking changes that may be introduced with a new version.

**API_NAME=YourApiName API**
This option is used as your api name when generating your api documentation via the `api:docs` command.

**API_STRICT=false**
When strict mode is enabled, requests need to specify an `Accept` header. If none is provided an exception will be thrown, instead of using the specified default version. However, this means you will not able to view your api in the browser. You would possibly turn this on, but for developing it is quite handy to quickly view your api in the browser, so we set it to `false` for now.

**API_DEBUG=true**
Just remember to turn this off in your production app.

There are more options available but we do not need to set them at the moment. If you are interested, head over to the [docs](https://github.com/dingo/api/wiki/Configuration), to read all of them.

## Preparing the test setup with phpunit

Phpunit is just one testing framework you could use, phpspec is another very good one, which a much nicer cli gui. However, since I prefere to stick to the basic functions, like `equals()` as much as possible in any case and phpunit comes preinstalled with lumen, I am not going to bother installing another framework.

We are writing integration tests, which means we need to call the api. The fastest way to do so is to install `guzzle` via composer. We will also install the [http-status package](https://github.com/lukasoppermann/http-status) which lets us use strings like `HTTP_OK` or `HTTP_CREATED` instead of their magic numbers 200 or 201. This is a good practise because it makes our tests verbose and easy to read. Tests should be a kind of documentation, the easier they are, the better. Phil Sturgeon has [some thoughts about this](https://philsturgeon.uk/http/2015/08/16/avoid-hardcoding-http-status-codes/) as well, if you are interested.

```bash
composer require guzzlehttp/guzzle:~6.0 --prefer-dist
composer require lukasoppermann/http-status --prefer-dist
```

Once everything is installed, open `tests/TestCase.php` which is the base class that all our tests will extend. We need to import the `Httpstatuscodes` interface, initialize a new guzzle client in the `setUp` function ([learn more about setUp in phpunit](https://phpunit.de/manual/current/en/fixtures.html)) and assign it to a `protected` variable, which we name `$client`. With this done, we can interact with guzzle using the `$this->client` variable and reference status codes like this `self::HTTP_CREATED`.

```php
<?php

use Lukasoppermann\Httpstatus\Httpstatuscodes;

class TestCase extends Laravel\Lumen\Testing\TestCase implements Httpstatuscodes {

    protected $client;

    public function setUp()
    {
        parent::setUp();

        $this->client = new GuzzleHttp\Client([
            'base_uri' => 'http://api.mylumenapi.app',
            'exceptions' => false,
        ]);
    }
```

## Writing the first test

We are ready to write our very first test, so lets think about how our api will actually work. Our posts will be grouped in collections, image something like a *travel* collection or a *tutorial* collection. This means our api needs to respond to `/collections/travel` with a list of posts. This is the first thing we will test, create a new file in the `tests` directory named `ColectionTest.php` with the code below.

```php
<?php

class CollectionTest extends TestCase
{
    /**
     * @test
     */
    public function get_a_collection_by_name()

        $response = $this->client->get('/collections/travel');

        $this->assertEquals(
            self::HTTP_OK,
            $response->getStatusCode()
        );

        $this->markTestIncomplete('add expected return data.');
    }
}
```

At the moment our test only checks for a correct status code, but we added a `markTestIncomplete` statment, which will remind as, that we are missing a vital part that we will add in the next article in this series. Don't forget to either prefix your test functions with `test_` or, like in the example above, add the `@test` doc block, otherwise phpunit will not be able to run your tests.

## Adding the route
If we run our test now, we get an error like `Failed asserting that 400 matches expected 200.`. Which was to be expected, since we did not add any routes. We will look into how we can improve this error message using the `http-status` package in a another post.

To add an api route in our `routes.php` we have have to use the `dingo/api` router, instead of lumens default router, to get all the versioning and header magic that dingo comes with. The process it is pretty straight forward, first we create a new instance of dingos api router, than we add a new *version group* to tell the router which api version the following routes are meant for. Within this group we just add a routing statement to return `test` when the route is called using a get request. Lumen automatically sets the status code to `OK` if we return a string, so our test should pass.

```php
$api = app('Dingo\Api\Routing\Router');

$api->version('v1', function($api){
    $api->get('collections/{collection}', function(){
        return 'test';
    });
});
```

That's it. In the [next part: Database, Model & Controller](151119-lumen-dingo-api-part-2) of this series we will seed our database and create the logic for retrieving this data.
