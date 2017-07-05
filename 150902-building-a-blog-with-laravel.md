---
title: Why you should always write your own blogging software
author: Lukas Oppermann
series: Building a blog with Laravel;1
category: code
description: Learn how to build a markdown file based blog with Laravel.
preview: Learn how to build a markdown file based blog with Laravel, so that you can get into writing as fast as possible.
---

> Okay, maybe I exaggerated. There is no "always" on the internet. But sometimes it might really be a good idea to look past the off-the-shelf products like Wordpress or Medium.

## The important part to get right when blogging
What do you think is the most important part about your blog? The SEO so people find it? Maybeit'sthe social media integration so your articles are easy to share? Maybeit'sthe design, that has to be perfect? No. The only thing that matters is **good content**. Everything else can be added afterwards.

If you have an idea for a post, get typing immediately, lest you lose spirit the of the moment. This is why my blog is powered by markdown files. It can write markdown in any text editor. I can be offline, in a train, on my phone. Markdown is perfect for writing, because all you can write and structure your content, no design options, no fuss. Wordpress in comparison is pretty complicated. Ghost and Medium have a pretty good writing experience, but they are not really available everywhere. You pretty much need the internet (I once lost an entire article, because Medium screwed up the sync when I came back online).

When you write your blog in markdown files, you own your content. While you are able to download your articles from most plattforms, this might change, or the exported content might be hard to use in a different system. Markdown is a universal format.

## The MVP: a barebone blog
What we really need is a listing of all our articles and an the individual article pages. Everything else is a bonus we can add once we have our first posts online. We are using markdown files so there is no need for any database interactions.

For now we should be fine with [Laravel](http://laravel.com/) as our php framework of choice and the [league/commonmark](https://github.com/thephpleague/commonmark) package for parsing markdown files.

So let's get started ...

## The setup for our blog
First we need to install Laravel via [composer](https://getcomposer.org/) which you should already have up and running. After the Laravel is done creating the project `cd` into your directory and require the commonmark package. We are using the `--prefer-dist` flag so we don't pull in tests and other development resources.

```bash
composer create-project laravel/laravel --prefer-dist yourBlog
cd yourBlog
composer require league/commonmark --prefer-dist
```

With everything installed we can add the folder articles in `storage/app/` and create our first article, the title will be prefixed by todays date in the format `YYMMDD` and all spaces should be replaced with a dash. The leading date in the order year, month, day has the nice effect of automatically sorting our posts by the latest date, which should be your most recent post.

```bash
mkdir storage/app/articles
echo -e "#First Post\nAll in markdown, nice." >> storage/app/articles/$(date '+%y%m%d')-first-post.md
```

At this point you probably want to `git init` your project and submit it to version control. Once you are done, you will notice that the `git status` does not include our posts. To solve this and track our posts in git, for versioning revisions we open the `storage/app/.gitignore` and edit it to look like the file below. Basically you need to add `!/articles/` to NOT ignore everything in there and change line one `*` to `/*`. Afterwards you should be able to add your posts to the git history.

```bash
/*
!/articles/
!.gitignore
```

## Building the article listing
Now that we have our posts, we need to build the php logic to display them. We will do this very dirty and look at refactoring it into a nice, clean piece of code in another post.

To start with we setup our routes in the `app/Http/routes.php` file. I use `/blog`, because I am actually adding this blog into my existing website. But if the list of articles is supposed to be your homepage, feel free to just use `/`.

```php
Route::get('/blog', 'BlogController@index');
Route::get('/blog/{article}', 'BlogController@show');
```

The `BlogController` we reference in the routes does not exist yet, so we need to create it. Laravel ships with the artisan command line tool, which let's you create a new controller with a simple make command. By default artisan creates a REST-Controller, but by using the `--plain` flag we can get an empty controller.

```bash
php artisan make:controller --plain BlogController
```

The first method we reference is the `index` method, so let's open `app/Http/Controllers/BlogController.php` and add this method. We also need to add two `use` statements at the top, as we will be using the Laravel `Storage` facade, and the commonmark `CommonMarkConverter`.

```php
<?php

namespace App\Http\Controllers;

use App\Http\Controllers\Controller;
use League\CommonMark\CommonMarkConverter;
use Storage;

class BlogController extends Controller
{
    /**
     * Display a listing of the resource.
     *
     * @return view
     */
    public function index()
    {
        $articles = Storage::files('articles');

        foreach($articles as $article)
        {
            if(pathinfo($article)['extension'] === 'md'){
                $article_file = pathinfo($article)['filename'];

                $article_list[] = [
                    'link' => $article_file,
                    'title' => $this->getTitle($article_file),
                    'date' => $this->getDate($article_file),
                ];
            }
        }

        return view('blog.listing', ['articles' => $article_list]);
    }
```

The `index` method is pretty simple. First we use the `Storage` facade to find all articles. By default it searches in `storage/app`, by providing `articles` as a parameter we will get every file within `atorage/app/articles`.

Then we loop through all returned files to build up an `$articles_list` array, which has a link, title and date for every post. Retrieving the date and title from the filename is done in separate private methods, which we will look into in a moment.

Once the array is complete we pass it to the `views/blog/listing.blade.php` which will take care of the rest.

## Extracting date & time
We still need implementing the `getTitle` and `getDate` methods. For the title we just have to remove the first 7 characters from the file name (the date `YYMMDD` and the dash) and replace all other dashes with spaces.

Retrieving the date is nearly as easy, you just need to cut the pieces of the date into the parts day, month, year and add them in the desired format. I opted for the european format `DD.MM.YY`. Just in case you forgot it, `substr` works by providing the string, the position where to start from, and the amount of characters to subtract.

```php
/**
 * Get formatted title from filename
 */
private function getTitle($filename)
{
    return str_replace('-',' ',substr($filename,7));
}
/**
 * Get formatted date from filename
 */
private function getDate($filename)
{
    return substr($filename,4,2).'.'.substr($filename,2,2).'.'.substr($filename,0,2);
}
```

## Building the blade views

To get our post listing on the screen we need two views, a master view with out base page and a listing view. First we create `views/master.blade.php` which has the basic html, head, body, etc. and is loading our css file. The `@yield('content')` is our insertion point for the content.

```html
<!DOCTYPE html>
<html>
    <head>
        <title>{{$title or 'Your default title'}}</title>
        <meta name="viewport" content="width=device-width, initial-scale=1.0, minimum-scale=1,maximum-scale=1">
        <link href='{{asset('app.css')}}' rel='stylesheet' type='text/css'>
        <script>
            // I would probably add google analytics
        </script>
    </head>
    <body>
        <div class="o-container">
            @yield('content')
        </div>
    </body>
</html>
```

The listing view is just a little more complicated. I suggest adding it to a subfolder, because it is only used in the blog part of our website `views/blog/listing.blade.php`. This view `@extends` our master view and defineds a `@section('content')` which will replace the `@yield('content')` statement in the master view. At the top we add a title an a description for the listing page. Afterwards we iterate through the `$articles_list` array which we passed to the view and display the date and title of each entry in a list item.

```html
@extends('master')
@section('content')
    <div class="o-post-list">
        <h1>Title of your blog</h1>
        <p class="c-blog-intro">Some introtext for your blog.</p>
    </div>
    <ul class="o-list o-list--none">
        @foreach ($articles as $article)
        <li class="o-list-item">
            <span class="c-post-preview-time">{{$article['date']}}</span>
            <a class="o-link c-post-preview-title" href="{{url('blog/'.$article['link'])}}">{{$article['title']}}</a>
        </li>
        @endforeach
    </ul>
@endsection
```

## Building the article view

Now that we have a list of posts, let's make the individual posts work. The controller `show` method is very straight forward. First we check if a file with the provided name exists, if not we redirect to the homepage. Otherwise we retrieve the markdown file, create a new instance of the `CommonMarkConverter`, convert the markdown to html and pass the html to a view.

```php
/**
 * Display a specific post.
 *
 * @param  string  $name
 * @return view
 */
public function show($name)
{
    if( !Storage::exists('articles/'.$name.'.md') ) {
        return redirect()->action('BlogController@index');
    }

    $post = Storage::get('articles/'.$name.'.md');

    $converter = new CommonMarkConverter();
    $post = $converter->convertToHtml($post);

    return view('blog.post', ['post' =>  $post]);
}
```

The only thing left is creating the `views/blog/post.blade.php` file. Like the listing file the post file needs to `@extend` master and define the `@section('content')`. We have a back link at the top and bottom to get back to the listing and in between an `<article>` element, which prints the post variable.

```html
@extends('master')
@section('content')
    <a class="o-link o-link--back" href="{{url('blog/')}}">Back to overview</a>
    <article class="o-post__content">
        {!!$post!!}
    </article>
    <div class="o-footer">
        <a class="o-link o-link--back" href="{{url('blog/')}}">Back to overview</a>
    </div>
@endsection
```

That is it, all we need to do is add some minimal css to `public/css/app.css` to make it look readable. We still have much than can be improved, but this can wait until you wrote a couple posts. You will also have a much better idea of what features you would like to implement, after you have been using your blog for a while.

```css
html, body {
    margin: 0;
    height: 100%;
    color: rgb(50, 50, 55);
    font-family: 'Helvetica Neue', 'Helvetica', sans-serif;
    font-size: 18px;
    line-height: 135%;
}
.o-container{
    width: 100%;
    box-sizing: border-box;
    max-width: 800px;
    padding: 20px 10px;
    margin: 0 auto;
}
h1{
    margin-top: 0;
    font-size: 32px;
}
h2{
    margin-top: 40px;
    font-size: 26px;
}
code{
    display: inline-block;
    font-family: monospace;
    background-color: rgba(0, 0, 0, .05);
    padding: 0 3px;
}
pre code{
    background: none;
}
pre{
    background-color: rgba(0, 0, 0, .05);
    border-radius: 3px;
    padding: 10px 5px;
    overflow-x: scroll;
}
a{
    color: inherit;
    text-decoration: underline;
}
a:hover{
    text-decoration: none;
}
```

## Heed this warning
For brevity, this code is reduced to the bare minimum. You can use it for your own blog, but it is extremely unstable. If an article has a wrong name format, everything might break down. We will look into making this system more stable, secure and useful, but this is enough to get you started. Since you control the system, you can make sure to not break it.

{#a-blog-in-progress}
## Sidenote: a blog in progress ...
am actually build this blog by exactly the same model. I started with a very basic version, just like outlined above and I am adding to it all the time. I got frustrated with not publishing anything because of all the hassle associated with it. Now I can just writing markdown files and only concentrate on the content, which works very well for me.
