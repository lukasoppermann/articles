---
title: Revision files & more with gulp
author: Lukas Oppermann
series: Building a Progressive web app;2
category: code
description: Iteratively turning a PHP base website into a progressive web app
preview: Learn how to start transforming your website into a progressive web app. In part two we improve our gulpfile and add revisioning to it.
---

> Building a version of your app/site should be a matter of seconds and be completely automated, so you can do it often and find issues fast.

In the <a href="170117-building-a-pwa-from-your-existing-website" class="o-link o-link--decorated">previous part of this series</a>
we started converting out php based website into a progressive web app (PWA) and added a `gulpfile` to convert our mustache templates to html. In this part we will revision our files with gulp and simplify our `html` task.

## Revisioning files
To revision our assets we use the [`gulp-rev`](https://github.com/sindresorhus/gulp-rev) package:

```bash
$ yarn add -D gulp-rev
```

`gulp-rev` takes some files and creates revisioned versions by appending a hash based on the files content to it. It also creates a `rev-manifest.json` file with key-value pairs matching the original file names to the revisioned file names. This makes it easy referencing a static name, while still being able to use dynamic filenames for cache busting. Let's look into how we can get this working. First create a gulp `rev` tasks to take care of our revisions.

```js
/* ------------------------------
 *
 * Revision
 *
 */
gulp.task('rev', function (done) {
  const rev = require('gulp-rev')

  return gulp.src([
    'public/css/app.css',
    'public/js/*.js',
    'public/svgs/svg-sprite.svg'
  ], {base: 'public'})
    .pipe(rev())
    .pipe(gulp.dest('public'))
    .pipe(rev.manifest())
    .pipe(gulp.dest('public'))
})
```

We need to get all the files we want to revision. You should probably revision all assets you have, to make cache invalidation easier later on.
Afterwards we pipe it through `rev()` and direct the output to wherever we want via `gulp.dest`. Than we can write the manifest with `rev.manifest()`. This will replace our old manifest with a new one.

### Cleaning up
This is all fine, but we end up with all old revisions and new revisions in out public folder – not good. However, there is an easy fix, we can just read the `rev-manifest.json` and delete all files that are in it. This is preferable to deleting all files in all folders, because we might have some non-revisioned files in those folders as well. Of course we need to do this before we create the revisioned versions.

```js
gulp.task('rev', function (done) {
  // synchronously delete old files
  if (fs.existsSync('public/rev-manifest.json')) {
    var manifest = fs.readFileSync('public/rev-manifest.json', 'utf8')
    del.sync(Object.values(JSON.parse(manifest)), {'cwd': 'public/'})
  }
  // …
```

#### Original files
The code above takes care of old revisioned files, but the transformed files are still in our public folder.

**Note:** I am assuming your source files are in a different directory, e.g. `resourecs/css/app.css` and you have a task within your `gulpfile` which transforms those files, e.g. runs the css files through `postcss` and moves them to the `public` folder. If your files in `public` are the source files, do **NOT** add this step. However, you should always aim to have no files in `public` that are not put there during your build step.

First we need to install another package:

```bash
$ yarn add -D gulp-rev-delete-original
```

Afterwards you need to require it and `pipe` the stream through it after the `rev` step. Thats it, your directory should be nice an clean now.

```js
gulp.task('rev', function (done) {
  const rev = require('gulp-rev')
  const revdel = require('gulp-rev-delete-original')

  // synchronously delete old files
  if (fs.existsSync('public/rev-manifest.json')) {
    var manifest = fs.readFileSync('public/rev-manifest.json', 'utf8')
    del.sync(Object.values(JSON.parse(manifest)), {'cwd': 'public/'})
  }

  return gulp.src([
    'public/css/app.css',
    'public/js/*.js',
    'public/svgs/svg-sprite.svg'
  ], {base: 'public'})
    .pipe(rev())
    .pipe(gulp.dest('public'))
    .pipe(revdel())
    .pipe(rev.manifest())
    .pipe(gulp.dest('public'))
})
```

## Updating the html task
We have our nicely revisioned files, but we have to find a way to get them into out compiled templates, preferably without adding the hash manually.
Luckily we have the `rev-manifest.json`, which we can read into a variable and add to the `data` object. Because most template engines do not allow `/` and `.` in variable names (and out keys have both), we need to do a little renaming when we add them.

```js
gulp.task('html', function () {
  // load portfolio data
  let data = {
    'portfolioItems': JSON.parse(fs.readFileSync('resources/templates/data/portfolio.json'))
  }
  // prepare files
  const files = JSON.parse(fs.readFileSync('public/rev-manifest.json', 'utf8'))
  Object.keys(files).map(function (key, index) {
    data[key.replace('.', '_').replace(/^[a-z]+\//, '')] = files[key]
  })
  //...
```

After we have parsed the portfolio items, we load the `rev-manifest.json` file into a variable. Looping over the keys we remove the path and replace the `.` in the filename with a `_`. Afterwards we can append the entries to the `data` object.
Of course, before this works we need to change the reference to the static filename of the assets with a mustache variable. For example instead of `public/css/app.css` we are now using `/{app_css}` in the `head.mustache`. Once this is done your `gulp default` task should deliver the same result as before.

In the <a href="170403-building-a-pwa-from-your-existing-website-part-3" class="o-link o-link--decorated">next part of Building a Progressive</a> web app we will be setting up a service worker to make our page faster and offline available.
