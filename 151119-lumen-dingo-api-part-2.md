---
series: Building APIs with dingo & lumen;2
title: Creating the API database, models & controllers
category: code
tags: tag1, tag2
author: Lukas Oppermann
description: Learn how to build a php API with dingo & lumen\: Creating the api database, models & controllers.
preview: With the packages set up we now need to create the database and create our models and controllers to return a response when we receive an actual api call.
---

> In the [introduction](150905-api-with-dingo-and-lumen-part-1) of this series we set up our project environment and installed some packages. We configured the dingo/api package and setup a route as well as a basic phpunit test. In this part we will migrate and seed our database and create the controller and model.

## Creating your sqlite database
An API needs data, or else it is pointless. I recommend working with *sqlite*, especially during development, as it requires no setup outside of your application. To setup *sqlite*, you need to set your DB environment variable, which will be inside your `.env` file during development. Add `DB_CONNECTION=sqlite` to this file and save it.

Now you need to create the `database.sqlite` file, the easiest way is to do it via the terminal. Make sure it has the correct permissions, it should be *644* or *-rw-r--r--* (you can see permissions in the terminal by using `ls -l`). You can change the permissions via the terminal, in case they are not correct.

```bash
$ touch database/database.sqlite
$ chmod 644 database/database.sqlite
```

## Folder structure
Now that we have our database in place we need to create the model, but where should we put it? I like to create a custom folder `Api` to group all files. You can of course use the `Http` folder if you prefer.

Additionally I find it quite helpful to add version folders, I start with `V1` and if I need to create a new version, I will create a new folder `V2`. This lets me maintain separate versions side by side, without messing up my code by adding custom `if` clauses within the files. Of course this has the potential of much code duplication, because you will copy all files to a new folder. But generally I would rather have duplicate code than potentially breaking *V1* when I was changing something in *V2*. So my folder structure looks like this. You should create one like it, to follow the tutorial.

{.o-list .c-file-list}
- myLumenApi/ {.o-list__item .c-file-list__folder}
    - app/ {.o-list__item .c-file-list__folder}
        - V1/ {.o-list__item .c-file-list__folder}
            - Controllers/ {.o-list__item .c-file-list__folder}
            - Models/ {.o-list__item .c-file-list__folder}
            - Transformers/ {.o-list__item .c-file-list__folder}
    - bootstrap/ {.o-list__item .c-file-list__folder}
    - database/ {.o-list__item .c-file-list__folder}
    - ... {.o-list__item .c-file-list__folder}


## Model
According to our new folder structure, our model will be created within `app/Api/V1/Models/`, so lets do this now.

```bash
$ touch app/Api/V1/Models/Collection.php
```

Our `Collection` model is pretty straight forward. We namespace it according to *PSR-4* for the autoloading to work, and `use` eloquent, which provides us with the model base class, which we extend with our own implementation. All we need to do in our model for now is to set the `timestamp` option and the `incrementing` option to `false`.

```php
<?php

namespace App\Api\V1\Models;

use Illuminate\Database\Eloquent\Model;

class Collection extends Model
{
    /**
     * Indicates if the model should be timestamped.
     *
     * @var bool
     */
    public $timestamps = false;
    /**
     * Indicates if the model should force an auto-incrementeing id.
     *
     * @var bool
     */
    public $incrementing = false;
}
```

### Timestamps & soft deletes
The `timestamp` option automatically adds a `created_at` and `modified_at` column to the table. As our collections have no relevance other than to group our content pieces, we are not interested in knowing about the creation and modification date of collections. For content like posts, however those fields can be very important, to show a last modified date, or the creation date of something. Additionally you have the option to include a soft delete. This is an awesome concept for items that you might want to restore once they have been deleted, e.g. articles. Basically it adds a `deleted_at` column to the table. If it is anything other than `NULL` the entry is deleted. To restore it the `deleted_at` field of an entry has to be reset to `NULL`. Eloquent, the ORM that ships with lumen, knows how to work with soft deletes, so by default deleted items are not returned, but you can get them using the `withTrashed()` or `onlyTrashed()` methods described in the [docs](https://laravel.com/docs/5.2/eloquent#soft-deleting).

To add soft deletion to your model simply load and `use` the `SoftDeletes` trait. Eloquent will take care of the rest.

```php
...
use Illuminate\Database\Eloquent\SoftDeletes;

class SomeModel extends Model
{
    use SoftDeletes;
    ...
```

 All that is left to do is to add the fields to your migrations. Within the `up` call simply add the following two lines at the end.

```php
Schema::create('...', function (Blueprint $table) {
    // your normal migration stuff
    $table->string(...);
    // adding timestamps
    $table->timestamps();
    // adding softDeletes
	$table->softDeletes();
});
```

### Incrementing
With the `incrementing` option set to `true` we need to use an *auto incrementing* field as an id. However, we will be using *uuids*, so we need to set it to `false`. The benefit of a *uuid* is that it obfuscates our ids, so nobody can simply guess an id and steal all our content.
Using incrementing ids a script could just query our API for `/collections/1`, `/collections/1`, ... which would be even worse for the actual entries within a collection. This is why I prefer to use uuids even if I am not sure if the ids will be exposed at all. If you want to know more, read [phil's post on uuids](https://philsturgeon.uk/http/2015/09/03/auto-incrementing-to-destruction/). If you are creating models for internal use only, for e.g. a `User` model, for a system where users are not available via the API, you should probably stick with incrementing ids. So you need to decide the correct value for this option for every model.

## Database migration & seeding
Our model is in place so we can move on to our migration. Create a new migration file using the artisan helper in your terminal.

```bash
$ artisan make:migration create_collection_table
```

The new migration file will be placed within `database/migrations`, and look something like this `2015_11_17_183803_create_collection_table.php`. The `down` method just `drops` the table, the interesting stuff happens within the `up` method. The two *ids* are of the type `binary` because this will be faster than using `char` for storing a *uuid*.

```php
/**
 * Run the migrations.
 *
 * @return void
 */
public function up()
{
    Schema::create('collections', function(Blueprint $table)
    {
        $table->binary('id');
        $table->string('type',255);
        $table->binary('page_id');
        $table->integer('position');
    });
}

/**
 * Reverse the migrations.
 *
 * @return void
 */
public function down()
{
    Schema::drop('collections');
}
```

### Modelfactory
Once you changed your migration to resemble the example above, we need to create our seeder, which puts the actual content in out database. We can use a handy artisan helper to create a seeder, similar to how we created the migration.

```bash
$ artisan make:seeder CollectionsTableSeeder
```

Open `database/seeds/CollectionsTableSeeder.php` and change it to look like this:

```php
public function run()
{
    factory('App\Api\V1\Models\Collection', 50)->create();
}
```

This is supposed to create 50 entries within the collection table, but we a using a `factory` function, which needs some configuration as well. So lets jump into `database/factories/ModelFactory.php` and replace the existing code with the code below. Lumen comes with [faker](https://github.com/fzaninotto/Faker) out of the box, faker is the supreme packages for creating random data with a specific type e.g. a random *email* or a random *uuid*, so of course we use this in our factory for the collection model. A [model factory](http://laravel.com/docs/5.1/testing#model-factories) is basically a blueprint to tell lumen how to spin up database entries for a given model.

```php
$factory->define(App\Api\V1\Models\Collection::class, function ($faker) {
    $types = ['travel', 'news'];
    return [
        'id' => $faker->uuid,
        'type' => $faker->randomElement($types),
        'position' => $faker->randomDigit(),
        'page_id' => $faker->uuid,
    ];
});
```
### DatabaseSeeder
The last thing to do before we can migrate & seed our database is to configure the `DatabaseSeeder.php`. While you could certainly just uncomment the `call` method and replace the name of the seeder class, I like to get a tiny bit fancy here. All tables that are supposed to be seeded are stored in the `$tables` array. Those tables are truncated (all data is deleted) first and seeded afterwards. This means if you change anything by hand (which you shouldn't do), it will all be gone, the next time you seed. But if you do not *truncate* your tables, every time you seed, additional rows will be created, instead of replacing the old rows. We do not want this to happen, so go on an truncate!

```php
class DatabaseSeeder extends Seeder
{
    // tables below will be:
    // 1. truncated / deleted
    // 2. seeded
    protected $tables = [
        'collections'
    ];

    /**
     * Run the database seeds.
     *
     * @return void
     */
    public function run()
    {
        Model::unguard();

        foreach($this->truncate as $table){
            // empty table
            DB::table($table)->truncate();
            // seed table
            $this->call(ucfirst($table).'TableSeeder');
        }

        Model::reguard();
    }
}
```

Now that everything is in place you should be able to migrate and seed your database. Just to make sure, I recommend to reset the migration & remigrate as well as to just reseed. This way you  can make sure that it all works fine.

```bash
$ php artisan migrate --seed
# reset & seed again to make sure
$ php artisan migrate:reset
$ php artisan migrate --seed
# re-seed
$ php artisan db:seed
```

## Routing & controller
In [part 1 of this series](150905-api-with-dingo-and-lumen-part-1) we had our route just return the string `test`, but this will no longer suffice. We want to use our future *CRUD* controller so we need to adjust our `route.php`.

```php
$api->group([
    'version' => 'v1',
    'namespace' => 'App\Api\V1\Controllers',
], function($api)
{
    $api->get('collections/{collection}', 'CollectionsController@show');
});
```

### Preparing the controller
For our final api we will be writing multiple controllers, one for every resource (we will discuss resources in a later part), and they will have some common code. To keep our code as [DRY](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself) as possible, we will start with a base api controller, which every api resource controller will extend.

```bash
$ touch app/Api/V1/Controllers/ApiController.php
```

Our `ApiController` must extend the basic laravel controller, so we need to `use` it, to make it available. We will be using the the dingo responses to return our collection, so we need to `use` the dingo helper `trait`. This needs to be declared within the class, as it is a [trait](http://php.net/manual/en/language.oop5.traits.php) and not a class that we import.

```php
<?php

namespace App\Api\V1\Controllers;

use Laravel\Lumen\Routing\Controller as BaseController;

class ApiController extends BaseController
{
    use \Dingo\Api\Routing\Helpers;
}
```

### The resource controller
Enough preparation, lets get into creating the collections controller which will retrieve all out entries and return them as a collection. Dingo uses the transformer idea to transform content intoit'sfinal form, so lets go ahead and create the transformer file as well.

```bash
$ touch app/Api/V1/Controllers/CollectionsController.php
$ touch app/Api/V1/Transformers/CollectionTransformers.php
```

In our `CollectionsController` we need to `use` the Model as well as the newly created transformer, as we will retrieve DB data via the model and format it using the transformer. For now only one method is needed: `show`. We retrieve all entries and pass them back as a collection using the dingo helper response. The response method takes the eloquent collection asit'sfirst argument and the transformer asit'ssecond argument. The transformer will automatically be applied to each item in the collection, so we do not need to deal with iterating over the collection within our transformer.

```php
<?php

namespace App\Api\V1\Controllers;

use App\Api\V1\Models\Collection;
use App\Api\V1\Transformers\CollectionTransformer;

class CollectionsController extends ApiController
{
    public function show()
    {
        $collections = Collection::all();

        return $this->response->collection($collections, new CollectionTransformer);
    }
}
```

### Transform your output
Before we can see the result in the browser, we need to add some code to the `CollectionTransformer`. We will look in depth into transformers in the next part, but for now we need to create a skeleton transformer so that at least we will see our data.

Dingo uses the excellent [Fractal](https://github.com/thephpleague/fractal) package, which comes with the transformer basics, so we need to `use` the fractal abstract transformer. Additionally we will need the collection model, so we can type hint it and make sure we get correct data.

We will add some logic to the `transform` method in the next part, for now we just accept the collection and return it as is. With this done you should be able to view your data by viewing the route `/collections/anything` in your browser.

```php
<?php

namespace App\Api\V1\Transformers;

use League\Fractal\TransformerAbstract;
use App\Api\V1\Models\Collection;

class CollectionTransformer extends TransformerAbstract
{

    public function transform(Collection $collection)
    {
        return $collection;
    }
}
```

In the next part we will implement the transformation logic based on some tests, to verify we return the format we want. Additionally we will look into retrieving individual entries.
