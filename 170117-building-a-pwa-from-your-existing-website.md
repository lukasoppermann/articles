---
title: Building a progressive web app (PWA) from your existing website
author: Lukas Oppermann
series: Building a Progressive web app;1
category: code
description: Iteratively turning a PHP base website into a progressive web app
preview: Learn how to start transforming your website into a progressive web app. In part one we use gulp to build fast and static html files from mustache templates.
---

> Starting from scratch is easy, but how do you start when you already have a website running?

Progressive web app (PWA) describes a web site/app that facilitates modern technologies like service workers and offline storage to create a fast and reliable user experience, that might even be available offline. You can <a href="https://developers.google.com/web/progressive-web-apps/" class="o-link o-link--decorated">read more about it here</a> if you want.

It has long been time for a refresh of my portfolio so I figured I will take the chance to explore this new and exciting idea. Currently my website sits on top of a Laravel php backend. While you can use this without a problem, I decided to transfer my website to [Nodejs](https://nodejs.org/en/). Since I will be using a lot of ajax to send incremental updates from the server, I feel node is a perfect fit. That being said, feel free to stick to PHP or any other backend technology you prefer.

To start getting​ the benefits of the update as fast as possible, we will not rewrite everything from the ground up, but add updates in small iterations. Each part of this series will end with a working version that we can publish.

## Moving to mustache templates

My website consist of three main sections: homepage, portfolio and blog. The blog is actually a separate Laravel app, which I will leave as is in the first iterations, as it does some heavy work, like converting markdown to html, which we can deal with later.
Splitting complicated sections into a separate app is perfect for us, as it allows for small changes at a time, without needing to solve the complicated issues in the beginning. If you have parts you do not want to tackle yet, just point some paths to a new separate app that slowly takes over more and more as you progress with the PWA.

My first iteration will consist of moving from the Laravel templating engine to mustache. Those templates will be converted to html files using `gulp` so that they can be served as static files.

The fastest way to get the content into mustache is to view the rendered pages in the browser and copy the source. This way you don't need to collect the content from all the different template files.

I like to separate my app directory into two folders. A `resources` folder with all templates and assets in their _raw_ state, before compilation. And a `public` folder with the complied files. Anything within public should be create via a build script, so that if deleted you can recreate the public folder by just running this build script.

<img class="o-article__image" src="/blog/media/pwa-part-1-directory.svg" alt="my directory structure" />

Lets start by creating our main templates like `/resources/templates/index.mustache` and copying the source from the live page into those files.

For the individual portfolio pages I added a `/resources/templates/portfolio` subfolder.
You might need to tweak some links, to actually call the `.html` files, if you where using `mod_rewrite` before, or make sure that your `mod_rewrite` is still working with the new page setup.

### Converting mustache to html using gulp

Since we are dealing with static content, we don't need the mustache templates to be converted at runtime. Converting them in our build step and serving static files will give us a better performance.

Assuming you have node installed on your machine, you should have `npm` available to install `gulp`, our build tool, as well as the `gulp-mustache` package to convert our templates.

```bash
# cd to your project
$ cd ~/website
# install gulp & gulp-mustache
$ npm i --save-dev gulp gulp-mustache
# create an empty gulpfile.js
$ touch gulpfile.js
```

Within the `gulpfile.js` we need to load `gulp` and the `gulp-mustache` package. Afterwards we can create a new `html` task that takes our `.mustache` files from `resources/templates/` and any subdirectory and converts those into `.html` files. The converted html files are now stored within the `public/` directory.

Once done, you simply need to type `gulp html` into your terminal to convert your files, woot (well, currently we are more or less only renaming the files 🙃).

```js
const gulp = require('gulp')
const mustache = require('gulp-mustache')

gulp.task('html', function () {
  return gulp.src([
      'resources/templates/*.mustache',
      'resources/templates/**/*.mustache'
    ])
    .pipe(mustache({
      extension: '.html'
    }))
    .on('error', console.log)
    .pipe(gulp.dest('public'))
  })
})
```

### Extracting duplicated content into partials

To improve the maintainability of our website further and keep our code DRY, we can extract reusable parts into partials – small bits that are included in our main templates. The head and footer sections are the first that come to mind, so lets create `resources/templates/partials/head.mustache` and `resources/templates/partials/footer.mustache`. The `head.mustache` holds everything from the `<!DOCTYPE html>` to the opening `<body>` tag. The footer will only have the closing `</body>` and `</html>` tag.

In our main templates we can now replace the parts that we just moved to the partials with `{> partials/head}` and `{> partials/footer}`. The `>` indicated that we are referencing a partial. If we run `gulp html` again everything should still be working as before, but now if you want to change something in the head you only need to update one file.

However, currently, all files from the `partials` directory are converted to html files as well. To avoid this, we update the `gulp.src` in our `html` task by adding `!resources/templates/partials/*.mustache` (note the leading `!`). This excludes all files within the `partials` directory from the `glob` of our task.

```js
const gulp = require('gulp')
const mustache = require('gulp-mustache')

gulp.task('html', function () {
  return gulp.src([
    'resources/templates/*.mustache',
    'resources/templates/**/*.mustache',
    '!resources/templates/partials/*.mustache'
  ])
  // ...
```

### Repeating objects with JSON data

On most websites you have some elements with the exact same html, just with different content. For me those are the items on my portfolio overview page. To improve this can extract the html into a template and the content into a JSON file. Afterwards adding a new item will be as easy as adding a couple lines to our JSON file.

Lets start with the JSON file `resources/templates/data/portfolio.json` and just add the content to it. If you need to, you can get creative and add something like a `class` property to toggle certain behaviours by adding or removing classes.

```js
{
  "portfolioItems": [
    {
        "title": "Open Everything",
        "description": "This magazine for the re:publica conference 2013 combines different articles on the topic Open Everything.",
        "src": "/media/card-open-everything.png",
        "link": "/portfolio/open-everything",
    },
    {
        "title": "swift / we are fast",
        "description": "Swift is a case study of a corporate identity for an airline focusing on speed and comfort.",
        "src": "/media/card-swift.png",
        "link": "/portfolio/swift"
    },
    // … repeat for every portfolio item
  ]
}
```

To have this data available within out mustache templates we need to load the file and supply it to the `mustache()` function in our gulp `html` task. The file is with nodes `fs.readFileSync` and parsed into the `data` variable using `JSON.parse` so that we get a JSON object. This object can be supplied as the first argument to the `mustache` function.

```js
gulp.task('html', function () {
  // load portfolio data
  let data = {
    'portfolioItems': JSON.parse(fs.readFileSync('resources/templates/data/portfolio.json'))
  }

  return gulp.src([
      'resources/templates/*.mustache',
      'resources/templates/**/*.mustache',
      '!resources/templates/partials/*.mustache'
    ])
    .pipe(mustache(data, {
      extension: '.html'
    }))
    .on('error', console.log)
    .pipe(gulp.dest('public'))
  })
})
```

To reference a variable we just use double curly braces e.g. `{{link}`. Now add the blueprint of your portfolio item to  `resources/templates/partials/portfolio-item.mustache` and add the `{{variables}}` to the appropriate places.

```html
<a class="o-card__container" href="{{link}}">
    <div class="o-card">
        <div class="o-card__sides">
            <div class="o-card__front">
                <img class="o-image" src="{{src}}">
            </div>
            <div class="o-card__back">
                <div class="o-card__content">
                    <h2 class="o-headline--h3">{{title}}</h2>
                    <p class="o-copy">{{description}}</p>
                    <div class="o-icon--arrow">&rarr;</div>
                </div>
            </div>
        </div>
    </div>
</a>
```

To _loop_ over our `portfolioItems` in mustache we can use [section](https://github.com/janl/mustache.js#sections). In every iteration we load the specified partial, which gets the values injected automatically. The `section` below loads the partial `partials/portfolio-item` for every entry in the `portfolioItems` array, values like `title` are available within the partial directly without the parent `{title}`.

```html
  <div class="o-cards">
    {{#portfolioItems}}
      {{> partials/portfolio-item}}
    {{/portfolioItems}}
  </div>
```

## Final steps
Now you just need to add you `css`, `js` and other tasks that either compile the given assets, or at least copy them into the `public` folder.

That's it, our website should look identical to when we started, but everything in the background was changed. We moved from a PHP based website, to an entirely static html website, using mustache templates which are converted in a build step. This should already give us a small performance boost.

Push your website to production and move on with the <a href="170327-building-a-pwa-from-your-existing-website-part-2" class="o-link o-link--decorated">next part of Building a Progressive web app</a> in which we will add asset revisioning.
