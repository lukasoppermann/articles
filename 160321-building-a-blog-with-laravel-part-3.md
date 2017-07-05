---
series: Building a blog with Laravel; 3
title: Performance improvement by adding a cache
tags: tag1, tag2
author: Lukas Oppermann
category: code
description: Learn how to use laravels caching, the carbon date package and more to improve your laravel blog
preview: To avoid the overhead of reading all files and parsing our markdown every time we need any information, we just create a cache with all this information available. This enables us to add many nice features while boosting performance.
---
{.o-article__introduction}
> Our controller is growing and getting crowded, to get further with our blog it's best to extract some logic to a service: the PostsService which will deal with getting, processing and caching our posts.

## Service-oriented architecture: Building a service
All our data is in files, which means we are working with flat file storage instead of a database. As we are using markdown, this adds an overhead of having to parse the files whenever we want to access information. This is especially bad on the post listing page, where many files need to be parsed at once.
To add more features and improve our performance we need to add a cache which can provide us with very fast access to information like the list of posts or meta info for each post. This should also remove the need to parse the markdown every time a post is viewed.

We will refactor all logic to read and parse posts into a new `PostService` class, which will include a cache, based on Laravels cache system. Let's start by creating the `app/Services/PostsService.php` file.

{data-label-file="Services/PostsService.php"}
```php
<?php

namespace App\Services;
// we will be needing all those imports
use Bookworm\Bookworm;
use Cache;
use Carbon\Carbon;
use League\CommonMark\Converter;
use League\CommonMark\DocParser;
use League\CommonMark\Environment;
use League\CommonMark\HtmlRenderer;
use Storage;
use Webuni\CommonMark\AttributesExtension\AttributesExtension;

Class PostsService {
    /**
     * @var array
     */
    protected $posts;

    /**
     * @method __construct
     */
    public function __construct(){
        // if cache does not exist (or we are working locally) we (re)build the cache
        if (!Cache::has('posts') || env('APP_ENV') === 'local') {
            $this->buildCache();
        }
        // store posts from cache in class variable
        $this->posts = Cache::get('posts');
    }

}
```

The `__construct` of our new class `PostsService` checks if a cache exists, if it does not (or if we are working locally) we call `buildCache`, which will rebuild our cache for us. Afterwards we get the cached posts via `Cache::get('posts')` and store the result in the class variable `$this->posts`. This way, if we ever want to update our caching logic we just need to change it in one place as we will be using `$this->posts` everywhere within the class.

## Caching posts using the Laravel cache system
The `buildCache` method from we used within `__construct` deletes the `posts` cache and afterwards recreate it, so updates and new articles will be included. We are using `rememberForever` which adds a cache item that never expires. This means we will need to manually clear the cache using `php artisan cache:clear` which we will automate in the next article.

{data-label-file="Services/PostsService.php"}
```php
/*
 * builds up the cache with all articles
 */
private function buildCache(){
    // clear post cache
    Cache::forget('posts');
    // renew post cache
    Cache::rememberForever('posts', function() {
        return $this->getPosts();
    });
}
```

Within the `rememberForever` method we are calling `getPosts`, which is in charge of retrieving the articles and parsing them. This functionality is currently implemented in the `BlogController` and will be removed once the `PostsService` is finished.

### Private methods
The `getPosts` method and most other methods are implemented as `private` methods, because they are internal. `Private` methods can not be cal from outside of their parent class. When writing a class you need to define a *public API*, meaning a way to access your class from the outside. This can be easily done through the use of `private` and `public` methods, where only `public` ones can be used from the outside. With this apporach you are free to change any `private` method without breaking anybody's code, as long as the `public` methods return the same data & structure.

{data-label-file="Services/PostsService.php"}
```php
    /*
     * reads all files and returns posts as array
     */
    private function getPosts(){

        foreach(Storage::files('articles') as $file)
        {
            if (pathinfo($file)['extension'] === 'md') {
                $articles[] = array_merge([
                    'link' => $this->getLink($file),
                    'date' => $this->getDate($file, $this->date_format),
                    'machine_date' => $this->getDate($file, 'Y-m-d'),
                ], $this->getDataFromFile($file));
            }
        }

        return $this->sortArticles($articles);
    }
```

The `getPosts` method grabs all files from our `articles` directory using Laravels `Storage` facade. We loop through the results, check if they are markdown files, and if so, add them to the `$articles` array once we parsed and formatted them. The array is returned sorted by the date in the filename in reverse chronological order.

### Formatting date & link
The `getLink` method we use within `getPosts` returns the file name without the extension by using phps `pathinfo` function.

{data-label-file="Services/PostsService.php"}
```php
/**
     * Get link from filename
     */
    private function getLink($filename)
    {
        return pathinfo($filename)['filename'];
    }
```

`getDate` gets the date by passing the first 6 characters from `getLink` to `formatDate`, which will return a formatted date or `false`. As we name our files `YYMMDD-title-of-post.md` the first 6 characters represent the date for a published article.

We return the date if it is *not* `false`. If it is `false` we return `false`, which will make the post not show up in the list of posts. However in case we are in our local environment we return `draft` which will make the post show up, so you see your drafts locally, but not on your server.

{data-label-file="Services/PostsService.php"}
```php
    /**
     * Get formatted date from filename
     */
    private function getDate($filename, $format)
    {
        $date = $this->formatDate(substr($this->getLink($filename),0,6), $format);

        if ($date !== false) {
            return $date;
        }

        return env('APP_ENV') === 'local' ? 'draft' : false;
    }
```

### Formatting the date using Carbon
The less code we maintain, the better, so we are going to use the [`Carbon` package](http://carbon.nesbot.com/docs/) to deal with formatting our dates:

{data-label-file="Services/PostsService.php"}
```php
/**
     * format date
     */
    private function formatDate($date, $format){
        try {
            $date = Carbon::createFromDate('20'.substr($date,0,2), substr($date,2,2), substr($date,4,2));
        }
        catch(\InvalidArgumentException $e)
        {
            return false;
        }

        return $date->format($format);
    }
```

The `Carbon::createFromDate` needs year, month and day, which we can `substr` from the `$date` argument. We need to prefix the year with `20` because we formatted the date as `160321` without the leading `20`.

The `createFromDate` method throws an exception if the provided date is invalid, which we catch (our file is a draft) and returning `false`.

If the date is valid we use `Carbon`s format method, with the `$format` string that we passed through. For the normal date we store the string in `$this->date_format` so it is easily adjustable.
For a simple european date you can add the string below to the very top of your file. The [php datetime formats](http://php.net/manual/en/datetime.formats.date.php) page has an overview of all available formatting options.

However, we also need to create a `machine_date` which we will use in the html5 `<time>` tags. We do not need to store this argument in a class variable, as the format is fixed and does not need changing.

{data-label-file="Services/PostsService.php"}
```php
/**
 * the format the dates will be converted to
 *
 * @var string
 */
protected $date_format = 'd/m/Y';
```

### Oh my, I am getting a headache from all those methods, why so many?
As a quick interlude let me address the *fear of too many methods*. Somehow there is the idea in some peoples heads, that we need to have as few functions as possible. Nothing could be further from the truth, you should always have as many functions as make sense. The two main reasons for extracting logic into a method are:

- logic is used in more than one instance (two already count!)
- an entire chunk of logic can be extracted

You only want to extract logic, if it is complete in itself though. The goal is to make your code easier to understand, by extracting logic to a method with a name that makes it evident what the method does. For example `formatDate` is easy to understand, while `ifDateInvalidReturnFalse` does not help much.

### Extracting file data
Now lets get to `getDataFromFile`, the method which deals with everything that is in the actual markdown file. This method should return an `array` with the *content*, the *title* and the *meta information*.

```php
[
    'title'   => 'Post title',
    'content' => 'Post content in html',
    'meta'    => [
        'readingTime' => '7',
        'tags'        => [
            'tag-one',
            'tag-two'
        ],
        'author'      => 'Lukas Oppermann',
        'description' => 'Meta description for the post',
    ]
]
```

To make this happen, we need to get the content of our markdown file using Laravels `Storage` facade. We pass it to `getContent`, which returns the converted html without the meta information at the top. The meta data will be returned by `getMetaData`. But to add the reading time, we need to wrap the result from `getReadingTime` (copied from `BlogController`) in a `meta` array so that it will be correctly merged with the rest of the meta data.

{data-label-file="Services/PostsService.php"}
```php
/**
 * Get data from file
 */
private function getDataFromFile($file)
{
    $fileContent = Storage::get($file);
    $htmlContent = $this->getContent($fileContent);

    return array_merge_recursive([
        'content' => $htmlContent,
        'meta' => [
            'readingTime' => $this->getReadingTime($htmlContent),
        ],
    ], $this->getMetaData($fileContent, $this->getTitle($file)));
}
```
### Converting markdown to html using CommonMark
Before we convert the markdown we need to remove the the meta information. The *regex* for grabbing the meta info is stored in the `$meta_regex` class variable.

{data-label-file="Services/PostsService.php"}
```php
/**
 * regex to retrieve meta-info from content
 *
 * @var string
 */
protected $meta_regex = '#^---\n(.*?)---\n#is';
```

Within `getContent` we now `preg_replace` the meta info using `$this->meta_regex` as the *regex* string. Afterwards you can either use the simple CommonMark conversion we used in the `BlogController` or go for the more advanced version if you fancy installing some [CommonMark extensions](https://github.com/thephpleague/commonmark#community-extensions). I am using the [CommonMark attribute extension](https://github.com/webuni/commonmark-attributes-extension), so I need to do the advanced setup:

1. Create a new CommonMark environment
2. Add the extension using `addExtension`
3. Create a new converter with the environment
4. Convert to html

{data-label-file="Services/PostsService.php"}
```php
/**
 * Get content from string
 */
private function getContent($fileContent)
{
    $content = preg_replace($this->meta_regex, "", $fileContent);

    // 1. new CommonMark environment
    $environment = Environment::createCommonMarkEnvironment();
    // 2. Add extension
    $environment->addExtension(new AttributesExtension());

    // 3. Create a new converter with the environment
    $converter = new Converter(
        new DocParser($environment),
        new HtmlRenderer($environment)
    );
    // 4. Convert to html
    return $converter->convertToHtml($content);
}
```

### Extracting meta data from markdown
The `getMetaData` method takes two arguments, the `$fileContent` from which to extract the meta information and a `$title` (result from `getTitle`) in case there was no title provided in the meta information. `getTitle` uses the `getLink` to retrieve the filename and checks if the first 6 characters are numeric, if they are, we remove them. Than we replace all `-` with spaces and `trim`.

{data-label-file="Services/PostsService.php"}
```php
/**
 * Get formatted title from filename
 */
private function getTitle($filename)
{
    $title = $this->getLink($filename);

    if( is_numeric(substr($title, 0, 6)) ){
        $title = substr($title, 6);
    }

    return trim(str_replace('-',' ',$title));
}
```

In `getMetaData` we extract the meta info using `$this->meta_regex` with `preg_match`. We get everything between the two `---`, which we pass to the `extractMetaData` method, our old [`getMeta` method](160316-building-a-blog-with-laravel-part-2#extract-meta) from the `BlogController` (I renamed it to avoid confusion).
If the title was set in the meta section, we replace `$title` with our title from the meta section.

{data-label-file="Services/PostsService.php"}
```php
/**
 * Get metadata from string
 */
private function getMetaData($fileContent, $title = "No title provided")
{
    preg_match($this->meta_regex, $fileContent, $data);

    $data = $this->extractMetaData($data);

    // set title if in meta info, otherwise use fallback from argument
    if( isset($data['title']) ) {
        $title = $data['title'];
    }

    // loop through meta and run every item through defined function
    foreach($this->available_meta as $key => $function){
        $meta[$key] = $this->$function( $data, $key );
    }

    return [
        'meta' => $meta,
        'title' => $title
    ];
}
```

Afterwards we need to deal with the rest of the meta `$data`. For this we add a class variable `$available_meta` which is an array with the *keys* being the meta items (e.g. *author*) and the value being a *function* name which is used to *parse* this item.

{data-label-file="Services/PostsService.php"}
```php
/**
 * data that will be in meta element
 *
 * @var string
 */
protected $available_meta = [
    'tags'          => 'meta_tags',
    'author'        => 'meta_default',
    'description'   => 'meta_default',
];
```

We loop through `$this->available_meta` in a `foreach`, run the specified function on the data and store the result in the `$meta` variable using the `$key` as the array key.

{data-label-file="Services/PostsService.php"}
```php
// loop through meta and run every item through defined function
foreach($this->available_meta as $key => $function){
    $meta[$key] = $this->$function( $data, $key );
}
```

For most items we will be using `meta_default`, which returns the item if it exists or returns `false` if it does not.

{data-label-file="Services/PostsService.php"}
```php
/**
 * Check for existence and return
 */
private function meta_default($data, $key){
    if (isset($data[$key])){
        return $data[$key];
    }

    return false;
}
```

Then only other meta function we currently need is `meta_tags` (but you might have other meta information, like categories, etc.). We pipe everything through `meta_default` and return `false` if we get a `false` back. Otherwise, the meta item must exist and we `explode` the tags at the comma `,` and `trim` every entry using `array_map`. Using `array_filter` we remove all empty array items and then return the result.

{data-label-file="Services/PostsService.php"}
```php
/**
 *	prepare tags
 */
private function meta_tags($data, $key){
    if( $item = $this->meta_default($data, $key)) {
        $tags = array_map('trim', explode(',',$item));
        return array_filter($tags);
    }

    return false;
}
```

## Public API
All our methods have been `private`, now we build the *public API* to access posts from outside of the `PostsService`. For now we need two `public` method: `all` and `get`, which will be place just after the `__construct`.

The `all` method needs to return an array of all posts, which is already stored in the `$this->posts` variable, so we just return it.

{data-label-file="Services/PostsService.php"}
```php
/**
 * get list of all posts as array
 *
 * @method all
 *
 * @return array
 */
public function all(){
    return $this->posts;
}
```

The `get` method gets a `$link` string and needs to return the matching post, if it exists. To check if a post exists in `$this->posts` we do an `array_search` for `$link`. Since `$this->posts` is a multidimensional array we need to use [`array_column`](http://php.net/manual/en/function.array-column.php) to actually compare the sub-entry. If the link exists we get the `$key` which we use to return the given entry from `$this->posts`, otherwise we return `false`. That is it for the `PostsService`.

{data-label-file="Services/PostsService.php"}
```php
/**
 * get individual post
 *
 * @method get
 *
 * @param  string $link
 *
 * @return array
 */
public function get($link){
    $key = array_search($link, array_column($this->posts, 'link'));

    if($key !== false){
        return $this->posts[$key];
    }

    return false;
}
```

## Refactoring the BlogController to use the post service

A controller is basically a management layer that handles incoming requests and returns a result, like a view. Currently we only get two kinds of a requests, one for a list of articles (`index`) and one for a single article (`show`). Both will receive an instance of the `PostsService` so we need to `use` it at the top.

All that `index` will do is return a view, which gets the array of posts from `$posts->all()` as the argument.


{data-label-file="Controllers/BlogController.php"}
```php
<?php

namespace App\Http\Controllers;

use App\Http\Controllers\Controller;
use App\Services\PostsService as Posts;

class BlogController extends Controller
{
    private $metadata = [];
    /**
     * Display a listing of the resource.
     */
    public function index(Posts $posts)
    {
        return view('blog.listing', [
            'posts' => $posts->all()
        ]);
    }
    // ... show method
}
```

The `show` method, gets a `$link` as the second argument which is used in `$posts->get($link)` to retrieve the individual post and store it in the `$post` variable. We check if `$post` exists and redirect to the listing of articles, if it does not. However, if we get a valid post we return the `blog.post` view and pass the `$post` variable as the additional argument.

{data-label-file="Controllers/BlogController.php"}
```php
    /**
     * Display the specified resource.
     */
    public function show(Posts $posts, $link)
    {
        $post = $posts->get($link);

        if( $post === false ) {
            return redirect()->action('BlogController@index');
        }

        return view('blog.post', $post);
    }
```

## Adjusting the templates
All that is left to do is to adjust our views for the new data structure. In `listing.blade.php` we replace
```php
<ul class="o-list o-list--none">
    @foreach ($articles as $article)
        ...
    @endforeach
</ul>
```
with an `@each` that loops through `$posts` and passes every item as `$post` to the view `blog.preview`.

{data-label-file="blog/listing.blade.php"}
```html
@extends('master')
@section('content')
<div class="o-post-list">
    <h1>Title of your blog</h1>
    <p class="c-blog-intro">Some introtext for your blog.</p>
</div>
<ul class="o-list o-list--none">
    @each('blog.preview', $posts, 'post')
</ul>
@endsection
```

Of course, now we need to create the `blog.preview` view. In here, we only show anything if the date is not `false`. *Remember, that's why we set the date for unpublished articles to "draft" when working locally and to* `false` *on the server*.

{data-label-file="blog/preview.blade.php"}
```html
@if ($post['date'] !== false)
    <li class="o-list-item">
        <time class="c-post-preview-time">{{$post['date']}}</time>
        <a class="o-link c-post-preview-title" href="{{url('blog/'.$post['link'])}}">
            {{$post['title'] or 'No title provided'}}
        </a>
    </li>
@endif
```

The last tiny change is in the `blog.post` view, which also gets `$post` passed as an argument. The *content*, which we want to show can be accessed via the `$content` variable in the view, instead of the former `$post`. After this change we are all done.

{data-label-file="blog/post.blade.php"}
```html
@extends('master')
@section('content')
    <a class="o-link o-link--back" href="{{url('blog/')}}">Back to overview</a>
    <article class="o-post__content">
        {!!$content!!}
    </article>
    <div class="o-footer">
        <a class="o-link o-link--back" href="{{url('blog/')}}">Back to overview</a>
    </div>
@endsection
```

In this part we introduced a cache to speed up the blog and enable features like showing the `title` from the meta info in the post listing, without a performance hit. Make sure to `php artisan cache:clear` the cache every time you change anything in a post or add a new post. We will automate this in the next part, but until than you need to clear that cache.
