---
series: Building APIs with dingo & lumen; 5
title: Json API response formats & pagination
tags: tag1, tag2
author: Lukas Oppermann
category: code
description: Learn how to build a php API with dingo & lumen\: Json Api Response Formats & Pagination
preview: Converting content into json api format is not fun, so we will let dingo automatically do it for us. Additionally we will learn how to return our results paginated.
---

We are building a [json api](http://jsonapi.org/format/) conform api, which means we need to somehow get our database data into the json api format. You could do it by hand within your transformers, but I find it pays to do as little work as possible, so we will let fractal do all the heavy lifting. This significantly reduce the amount of code we have to write and maintain. The more of your code that is maintained by an active open source community, the more time you have for your business specific code, which nobody will write for you.

## Fractal Transformer & Serializer

`Dingo/api` ships with fractal and fractal has a serializer system. Serializers automatically convert your data into a specific format, like json api. The included serializers let you convert your data into a json api format, an array or a `data` namespaced array. And if you need something special you can write a [custom serializer](http://fractal.thephpleague.com/serializers/#custom-serializers). For our example we will be using the `JsonApiSerializer`, since it is exactly what we want.

To change the default serializer you need to add this bit of code to your `bootstrap/app.php` below the `Register Service Providers`. We basically call up the transformer factory, which is the basis of all transformers you create. It has a `setAdapter` which we use. An adapter is basically a package that provides the transformation logic.

```php
// set up default serializer
$app['Dingo\Api\Transformer\Factory']->setAdapter(function ($app) {
    // our code will be here
});
```

We need to create a new fractal manager and a new `JsonApiSerializer` with the api domain as an argument. This automatically triggers the addition of relationship links to our api responses. Afterwards we use the `setSerializer` method of the fractal manager to tell it to use our new serializer.

```php
$fractal = new League\Fractal\Manager;
$serializer = new \App\Api\V1\Serializer\JsonApiSerializer($_ENV['API_DOMAIN']);
$fractal->setSerializer($serializer);
```

Finally we need to `return` a new fractal adapter instance, which gets the fractal manager as the first argument. The other arguments are the string that is used for includes in the url, the delimiter of the includes in the url and a boolean to set [eager loading](http://fractal.thephpleague.com/transformers/#eager-loading-vs-lazy-loading) to true or false. The entire code looks like this:

```php
// set up default serializer
$app['Dingo\Api\Transformer\Factory']->setAdapter(function ($app) {
    $fractal = new League\Fractal\Manager;
    $serializer = new \App\Api\V1\Serializer\JsonApiSerializer($_ENV['API_DOMAIN']);
    $fractal->setSerializer($serializer);
    // return a new Fractal instance
    return new Dingo\Api\Transformer\Adapter\Fractal($fractal, 'include', ',', true);
});
```

Done, now whenever you return something you should get a perfectly valid json api response. But wait, what about errors? Well...

## Custom error formats with dingo api

For errors we need to do something similar. Underneath the our new serializer setup in the `bootstrap/app.php` we need to call the `setErrorFormat` method on the `Dingo\Api\Exception\Handler`. The method takes the desired error format asit'sargument. The strings beginning with a colon (`:`) like `:message` will be replaced with the real content of the error. Now if we throw a `NOT_FOUND` exception, dingo will convert it to a valid json api error response.

```php
// set up error format
$app['Dingo\Api\Exception\Handler']->setErrorFormat([
    'error' => [
        'message' => ':message',
        'errors' => ':errors',
        'code' => ':code',
        'status_code' => ':status_code',
        'debug' => ':debug'
    ]
]);
```

## Pagination
Now we got our format in order, but we still have a problem. When a resource is requested we currently return all items, that could eventually mean thousands of items are returned at once. This would firstly lead to potential long download times and also be a stability issue for your system. Pagination can solve this for us. The json response will have a pagination object, like the one shown below, with links to get to the previous and next set of items via an additional request.

```javascript
"meta": {
    "pagination": {
        "total": 96,
        "count": 20,
        "per_page": 20,
        "current_page": 1,
        "total_pages": 5,
        "links": [
            "previous": "http://api.domain.app/posts?page=1",
            "next": "http://api.domain.app/posts?page=3"
        ]
    }
}
```

### Pagination using eloquents paginate method

The pagination logic will be added to the `PostController.php`, we return 20 posts, which we define in the `private $perPage` variable, when the `index` method is called. In the controller we can use eloquent and as always eloquent provides us with a dead simple way to paginate items, the [`paginate` method](https://laravel.com/docs/5.2/pagination#paginating-eloquent-results). When you new up a model, like `Post`, you can immediately call the `paginate` method on it and provide the number of items per page as the only argument.

We got out result paginated, but we are still missing the pagination object in the api response. Luckily `dingo/api` got us covered. Instead of returning a `collection`, we just return a `paginator` using `$this->response->paginator` with the same arguments we used for the collection and the rest happens automatically.

```php
use App\Api\V1\Transformers\PostTransformer;

class PostsController extends ApiController {

    private $perPage = 20;

    public function index(){
        // get the posts model
        $posts = new Posts:paginate($this->perPage);
        // return collection with paginator
        return $this->response->paginator($posts, new CollectionTransformer, ['key' => 'posts']);
    }

}
```

### Relationships with pagination
#### Adding the related posts route
This was pretty easy, but what if we are retrieving the *posts* through a relationship? If we hit `/collections/{uuid-of-collection}/posts` we want the posts paginated just like before. First we need to actually set up the route in `routes.php` within the api group.

```php
$api->get('collections/{collection_id}/posts', 'CollectionsController@getPosts');
```
#### Adding the getPosts method
We are calling the `getPosts` method on the `CollectionsController`, so we need to add it. Also make sure you `use` the `Illuminate\Http\Request;` and `App\Api\V1\Transformers\PostTransformer` at the top of your file. In the method we simply retrieve the `collection` with the current `collection_id` and call the `posts` relationship on it, which we discussed in [part 4: Model Relationships](160103-lumen-dingo-api-part-4). To see if it is working, we will just return a collection response, like we did before. You should get all results when you call this route.

```php
namespace App\Api\V1\Controllers;

use Illuminate\Http\Request;
use App\Api\V1\Transformers\PostTransformer;
// other use statements ...
class CollectionsController extends ApiController {

    // some code ...

    public function getPosts(Request $request, $collection_id){
        // getPosts code ...
    }
}
```

First we have to get the current collection by the provided `$collection_id` and check if this collection actually exists. In case the collection does not exist, we thrown `NotFoundHttpException` which will be automatically converted to a json api error response by dingo/api.

```php
public function getPosts(Request $request, $collection_id){
    // get the collection with the current id
    $collection = Collection::find($collection_id);

    // throw 404 exception if resource does not exists
    // this will be converted to a jsonapi error by dingo
    if ($collection === null) {
        throw new \Symfony\Component\HttpKernel\Exception\NotFoundHttpException;
    }
```

#### Adding the pagination to the related posts
Now all that is left to do is to return a `paginator` response like before. The collection will be a paginated relationship. Make sure to include the `()`, because `$collection->posts` returns a laravel `collection` while `$collection->posts()` returns a `BelongsToMany` object, which is a `Relation` object and thus has the `paginate` method. Thats it, everything else is just like before.

```php
public function getPosts(Request $request, $collection_id){
    // ... code from above

    return $this->response->paginator(
        $collection->posts()->paginate($this->perPage),
        new PostTransformer,
        ['key' => 'posts']
    );
}
```

That's it. Posting a `GET` request to `collections/{collection_id}/posts` will return a paginated list of posts with links the api consumer can use to navigate to the next an previous page.
