---
series: Building a blog with Laravel; 2
title: Read time, descriptions and meta information
tags: tag1, tag2
author: Lukas Oppermann
category: code
description: Learn how to add a read time estimation, author, meta description and other features to your very own blogging system
preview: In part 2 we are enhancing our blogging system with read time estimations & meta information while refactoring the code to prepare for extracting everything into a service.
---

> Our blog is up and the first articles are published, perfect! Our goal was to get publishing as fast as possible. Now it is time to revisit the code and add some "nice to have" features.

## Adding meta information to markdown
In the [previous part of how to build a blog with Laravel](150902-building-a-blog-with-laravel) we discussed how to quickly get a markdown based blog up an running, now we need to improve it. The idea is to add meta information in a simple way, without destroying our markdown-and-done publishing method. We also want to make sure that all meta information is optional and our blog does not break when we don't add it. The best way is to add the meta info to the very beginning of our  markdown file, like this:

{data-label-file="your-post.md"}
```markdown
---
title: Our title for the post listing\: Colons need to be escaped
author: Lukas Oppermann
description: A description for the description tag
---
# Title of the post, that is displayed when the article is opened
{ $meta} <- without the space in front
```

The `$meta` part will later be replaces with some meta information that are supposed to be displayed in the text. We are using a variable instead of just adding it to the top, so that we can add an image or headline above the meta information, if we want to.

{#extract-meta}
## Extracting meta information from markdown
Now that we have a way to add it, we need to extract it again so we can work with our meta information. We will be using the meta information in our posts, so we need to get this data within the `show` method of our `BlogController`. The extraction is done in the `getPostContent` method which will return an array. If the method returns `false` the post did no exist, so we redirect to the list of posts.
However, if the array is returned we are all set and can return the view.

{data-label-file="Controllers/BlogController.php"}
```php
public function show($name)
{
    $post = $this->getPostContent($name);
    if( !$post ){
        return redirect()->action('BlogController@index');
    }

    return view('blog.post', $post);
}
```

The next thing to tackle is of course the `getPostContent` method. First we check if the article exists and return false if it does not. With the `$meta_regex` we retrieve everything between the `---` in the very beginning of your file and the next `---`, which will be our meta info. The info is store in the `$meta` variable and pass to the `getMeta` method, which returns the formatted meta info to which we append the date. Afterwards we return an array with the meta info, link, title, which is the result of the `getTitle` method and the content, which is the result from the `getContent` method. Note that we remove the meta info from the markdown content, before we pass it into the `getContent` method.

{data-label-file="Controllers/BlogController.php"}
```php
/*
 * get post by name
 */
private function getPostContent($name){
    // check if valid link
    if( !Storage::exists('articles/'.$name.'.md') ) {
        return false;
    }
    // regex to find meta info
    $meta_regex = '#^---\n(.*?)---\n#is';
    // get file
    $file = Storage::get('articles/'.$name.'.md');
    // get meta info
    preg_match($meta_regex, $file, $meta);
    $meta = $this->getMeta($meta);
    // add date
    $meta['date'] = $this->getDate($name);

    return [
        'link' => $name,
        'title' => $this->getTitle($name, $meta),
        'content' => $this->getContent(preg_replace($meta_regex,'', $file), $meta, $meta_regex),
        'meta' =>  $meta
    ];
}
```

Next we add the `getMeta` method. First we need to check if the current article has meta information. If so, we split it by row and afterwards split at the colon `:`, using the first part as the key and the second as the value in the array we return.

{data-label-file="Controllers/BlogController.php"}
```php
/**
 * add metda info to post content
 */
private function getMeta($meta){

    // if no meta data exists return false
    if(!isset($meta[1])){
        return false;
    }

    // split rows into array
    $data = array_filter(explode("\n", $meta[1]));

    // split at colon into key value pair
    foreach($data as $key => $item){
        unset($data[$key]);
        $item = preg_split('~\\\:(*SKIP)(*FAIL)|:~',$item);
        $data[$item[0]] = str_replace('\:',':',trim($item[1]));
    };

    return $data;
}
```

Moving on, we already wrote the `getDate` method and the `getTitle` in the previous post. The latter one needs a little adjusting though. We set the title to the filename, check if the title was set in the meta section and if so we use the title from the meta section.

{data-label-file="Controllers/BlogController.php"}
```php
/**
 * Get formatted date from filename
 */
private function getTitle($filename, $meta = false)
{
    // extract title from filename
    $title = trim(str_replace('-',' ',substr($filename,7)));

    //replace title if set in meta info
    if( $meta != false && isset($meta['title']) ){
        $title = $meta['title'];
    }

    return $title;
}
```

In the `getContent` method we initialize a new `CommonMarkConverter` and convert our markdown to html which is stored in the `$post` variable. Afterwards we render a view and store it into the `$metaInfo` variable, with which we will replace the `{$meta }` tag in the markdown file. The view gets an array with three variables: `author` which we need to check in case no meta info is provided, `date` and the `readingTime` which will be returned from the `getReadingTime` method. We return the post content, replacing the `{$meta }` tag with our rendered meta template.

{data-label-file="Controllers/BlogController.php"}
```php
/**
 * get content
 */
private function getContent($file, $meta, $meta_regex){
    // convert to html
    $converter = new CommonMarkConverter();
    $post = $converter->convertToHtml($file);

    $metaView = view()->make('blog.meta', [
        'author' => isset($meta['author']) ? $meta['author'] : false,
        'date' => $meta['date'],
        'readingTime' => $this->getReadingTime($post)
    ]);
    $metaInfo = $metaView->render();

    return str_replace('', $metaInfo, $post);
}
```

The blade template is very simple, a div with the author, date and reading time. We only need to add an `@if` statment to take care if the case when no author is set. In this case it will be `Published DD/MM/YY • X min read`.

{data-label-file="blog/meta.blade.php"}
```html
<div class="o-meta publication-info">
    Published
    @if ($author !== false)
        by <a class="author" href="http://vea.re" title="about {!!$author!!}" rel="author">{!!$author!!}</a>,
    @endif
    <time datetime="{!!$date!!}" class="article_time">{!!$date!!}</time> •
    <time datetime="{!!$readingTime!!}m">{!!$readingTime!!} min read</time>
</div>

```

## Reading time estimation
Before we create the `getReadingTime` method, we should get a basic understanding of how reading time estimations work: Basically the words are counted and multiplied with a *words per minute* variable which is typically *200*. Additionally you need to adjust for the time a user needs to look at images as well as deal with code blocks. For code blocks a user will not read every part of the code, for example nobody will read an svgs data attribute `d`, which is why it needs to be removed.

{data-label-file="your-post.md"}
```svg
<svg id="rainsuncloud" xmlns="http://www.w3.org/2000/svg" width="100" height="100" viewBox="0 0 100 100">
    <g id="sun" fill="#FFC300">
        <path d="M28.4 40.2C28.4 34.5 33 30 38.7 30c3.7 0 7 2 8.7 5 .6-.5 1.7-1 3.4-1.5-2.4-4.3-7-7.2-12.2-7.2-7.7 0-14 6.2-14 14 0 1.5.4 3 1 4.5.4-.5 1.4-1.3 ...">
    </g>
</svg>
```

A common implementation is to remove the entire code block, but if you write code-heavy posts, where people actually do look at the code, this will lead to a wrong reading time.

### Implementing a reading time estimation using the bookworm package
You could write this yourself, but luckily, there is a package for this: [worddrop/bookworm](https://github.com/worddrop/bookworm). So go ahead and install it via [*composer*](https://getcomposer.org/).

{data-label-file="command line"}
```bash
$ composer require worddrop/bookworm
```

To use the newly installed package we first need to import it at the very top of our file.

{data-label-file="Controllers/BlogController.php"}
```php
use Bookworm\Bookworm;
```

Now all we need to do is call the `Bookworm::estimate` method with the first argument beeing the `$post` and the second `false`, which means the amount of minutes will be returned as a simple *int*.
This is perfect because we will display the estimation in a `<time>` element, which needs the number of minutes followed by an `m` provided in the `datetime` attribute.

{data-label-file="Controllers/BlogController.php"}
```php
/**
 * get readingtime
 */
private function getReadingTime($post){
    return Bookworm::estimate($post, false);
}
```

You may be wondering why I choose to add a method `getReadingTime` for just one line of code. Well firstly it could happen that we need to add configuration later on and secondly `getReadingTime` gives you a much better idea of what the code does compared to `Bookworm::estimate`. That's it, in the next post we will refactor out controller, add caching and more features like previews.
