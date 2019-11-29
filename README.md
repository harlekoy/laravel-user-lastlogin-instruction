This package is inspired on Jonathan Reinink's blog on Dynamic relationships in Laravel using subqueries https://reinink.ca/articles/dynamic-relationships-in-laravel-using-subqueries

### Installation
Lets create first our Laravel App run the command below:
```
laravel new lastlogin-app
composer require laravel/ui
php artisan ui vue --auth
```

Then lets create our last login feature package based on Spatie's skeleton package boilerlate https://github.com/spatie/skeleton-php

```
git clone git@github.com:spatie/skeleton-php.git laravel-user-logins
```

Now lets `cd` to our laravel app and configure the package

```
cd lastlogin-app
./configure-skeleton.sh
```

Then add our package on our app under `composer.json` file

```diff
diff --git a/composer.json b/composer.json
index 288180d..1e7850d 100644
--- a/composer.json
+++ b/composer.json
@@ -11,7 +11,8 @@
         "php": "^7.2",
         "fideloper/proxy": "^4.0",
         "laravel/framework": "^6.2",
-        "laravel/tinker": "^1.0"
+        "laravel/tinker": "^1.0",
+        "harlekoy/laravel-user-logins": "dev-master"
     },
     "require-dev": {
         "facade/ignition": "^1.4",
@@ -30,6 +31,12 @@
             "dont-discover": []
         }
     },
+    "repositories": [
+        {
+            "type": "path",
+            "url": "../laravel-user-logins"
+        }
+    ],
     "autoload": {
         "psr-4": {
             "App\\": "app/"
```
> Note: You can change "harlekoy" to your github handle

Add since we have named our package to `harlekoy/laravel-user-lastlogin` we also need to update our package `composer.json` file

```diff
diff --git a/composer.json b/composer.json
index 9ddd951..defad87 100644
--- a/composer.json
+++ b/composer.json
@@ -1,17 +1,16 @@
 {
-    "name": "spatie/laravel-user-lastlogin",
+    "name": "harlekoy/laravel-user-lastlogin",
     "description": "Manage tool for user's recent login",
     "keywords": [
-        "spatie",
         "laravel-user-lastlogin"
     ],
-    "homepage": "https://github.com/spatie/laravel-user-lastlogin",
+    "homepage": "https://github.com/harlekoy/laravel-user-lastlogin",
     "license": "MIT",
     "authors": [
         {
             "name": "Harlequin Doyon",
             "email": "harlequin.doyon@gmail.com",
-            "homepage": "https://spatie.be",
+            "homepage": "http://harlekoy.com",
             "role": "Developer"
         }
     ],
     "require": {
-        "php": "^7.3"
+        "php": "^7.3",
+        "illuminate/support": "~6"
     },
     @@ -24,12 +24,12 @@
     },
     "autoload": {
         "psr-4": {
-            "Spatie\\Skeleton\\": "src"
+            "Harlekoy\\LastLogin\\": "src"
         }
     },
     "autoload-dev": {
         "psr-4": {
-            "Spatie\\Skeleton\\Tests\\": "tests"
+            "Harlekoy\\LastLogin\\Tests\\": "tests"
         }
     },
     "scripts": {
     @@ -42,10 +41,10 @@
     "extra": {
         "laravel": {
             "providers": [
-                "Spatie\\Skeleton\\SkeletonServiceProvider"
+                "Harlekoy\\LastLogin\\LastLoginServiceProvider"
             ],
             "aliases": {
-                "Skeleton": "Spatie\\Skeleton\\SkeletonFacade"
+                "LastLogin": "Harlekoy\\LastLogin\\Facade\\LastLogin"
             }
         }
     }
```

### Database Migration

Now lets create first our migration file

```
// Create the database directory
mkdir -p database/migrations

// Create the migration file stub
touch database/migrations/create_logins_table.php.stub
```

Add the script for migrating the necessary table and field

```php
<?php

use Illuminate\Support\Facades\Schema;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Database\Migrations\Migration;

class CreateLoginsTables extends Migration
{
    /**
     * Run the migrations.
     *
     * @return void
     */
    public function up()
    {
        // This will add the `last_login_id` in the users table
        Schema::table(config('lastlogin.user_table_name'),
            function (Blueprint $table) {
                $table->bigInteger('last_login_id')->nullable();
            }
        );

        // This will create the `logins` table
        Schema::create(config('lastlogin.table_name'), function (Blueprint $table) {
            $table->increments('id');
            $table->integer('user_id');
            $table->string('ip_address');
            $table->timestamp('created_at');
        });
    }

    /**
     * Reverse the migrations.
     *
     * @return void
     */
    public function down()
    {
        // Remove the `last_login_id` in the users table
        Schema::table(config('lastlogin.user_table_name'),
            function (Blueprint $table) {
                $table->dropColumn('last_login_id');
            }
        );

        // Drop the `logins` table
        Schema::drop(config('lastlogin.table_name'));
    }
}
```

You can see in the code we are extracting information from the non-existing config file. So we need to create that file and add the necessary variables the we can use through out the app and can dynamically change on the fly.

```bash
// Create the config file
touch config/lastlogin.php

// Write this to the config file
echo "<?php

return [
    'path'            => 'logins',
    'user_table_name' => 'users',
    'table_name'      => 'logins',
    'user_model'      => App\\\User::class,
];" > config/lastlogin.php
```

Lets not forget to update the service provider so that we can publish the correct `config` file in our Laravel app

```diff
diff --git a/src/LastLoginServiceProvider.php b/src/LastLoginServiceProvider.php
index a1ea84f..cd808a6 100644
--- a/src/LastLoginServiceProvider.php
+++ b/src/LastLoginServiceProvider.php
@@ -13,7 +13,7 @@ class LastLoginServiceProvider extends ServiceProvider
     {
         if ($this->app->runningInConsole()) {
             $this->publishes([
-                __DIR__.'/../config/lastlogin.php' => config_path('skeleton.php'),
+                __DIR__.'/../config/lastlogin.php' => config_path('lastlogin.php'),
-            ], 'config');
+            ], 'lastlogin.config');

             $this->publishes([
@@ -35,6 +38,49 @@
     public function register()
     {
-        $this->mergeConfigFrom(__DIR__.'/../config/config.php', 'skeleton');
+        $this->mergeConfigFrom(__DIR__.'/../config/lastlogin.php', 'lastlogin');
+    }
+
+    /**
+     * Returns existing migration file if found, else uses the current timestamp.
+     *
+     * @param Filesystem $filesystem
+     * @return string
+     */
+    protected function getMigrationFileName(Filesystem $filesystem): string
+    {
+        $timestamp = date('Y_m_d_His');
+        return Collection::make($this->app->databasePath().DIRECTORY_SEPARATOR.'migrations'.DIRECTORY_SEPARATOR)
+            ->flatMap(function ($path) use ($filesystem) {
+                return $filesystem->glob($path.'*_create_logins_tables.php');
+            })->push($this->app->databasePath()."/migrations/{$timestamp}_create_logins_tables.php")
+            ->first();
+    }
```

Then next we need to rename our service provider from `SkeletonServiceProvider` to `LastLoginServiceProvider` and other classes

```
mv src/SkeletonServiceProvider.php src/LastLoginServiceProvider.php
mv src/SkeletonClass.php src/LastLogin.php
mkdir src/Facade && mv src/SkeletonFacade.php src/Facade/LastLogin.php
```

After, add this code inside `LastLoginServiceProvider` under `boot` method, the code below will let us publish the migration file to our Laravel app.

```diff
/**
 * Bootstrap the application services.
 */
public function boot()
{
    if ($this->app->runningInConsole()) {
        ...

+       $this->publishes([
+           __DIR__.'/../database/migrations/create_logins_table.php.stub' => $this->getMigrationFileName($filesystem),
+       ], 'lastlogin.migrations');

        ...
```

### Model and Traits

Lets create our first Model

```bash
echo "<?php

namespace Harlekoy\LastLogin\Models;

use Illuminate\Database\Eloquent\Model;

class Login extends Model
{
    /**
     * The attributes that are mass assignable.
     *
     * @var array
     */
    protected \$fillable = ['ip_address'];

    /**
     * Dont set any 'updated_at' field.
     *
     * @var string|null
     */
    const UPDATED_AT = null;

    /**
     * Get user.
     *
     * @return \Illuminate\Database\Eloquent\Relations\BelongsTo
     */
    public function user()
    {
        return \$this->belongsTo(config('lastlogin.user_model'));
    }
}" > src/Models/Login.php
```

Next, let's create our user trait for this last login feature

```bash
mkdir src/Traits && touch src/Traits/HasLogins.php
```

Then add this code in the `HasLogins` trait
```php
<?php

namespace Harlekoy\LastLogin\Traits;

use Harlekoy\LastLogin\Models\Login;
use Illuminate\Support\Str;

trait HasLogins
{
    /**
     * Boot `HasLogin` trait.
     *
     * @return void
     */
    protected static function bootHasLogins()
    {
        static::addGlobalScope(function ($query) {
            $query->withLastLogin();
        });
    }

    /**
     * Get user logins.
     *
     * @return \Illuminate\Database\Eloquent\Relations\HasMany
     */
    public function logins()
    {
        return $this->hasMany(Login::class);
    }

    /**
     * Get the user last login.
     *
     * @return \Illuminate\Database\Eloquent\Relations\BelongsTo
     */
    public function lastLogin()
    {
        return $this->belongsTo(Login::class);
    }

    /**
     * Scope a query to include user last login.
     *
     * @param \Illuminate\Database\Eloquent\Builder $query
     * @return \Illuminate\Database\Eloquent\Builder
     */
    public function scopeWithLastLogin($query)
    {
        $table = config('lastlogin.user_table_name');
        $id = Str::singular($table).'_id';

        $query->addSelect(['last_login_id' => Login::select('id')
            ->whereColumn($id, $table.'.id')
            ->latest()
        ])->with('lastLogin');
    }
}
```

### Routes and Controller

Lets create the `routes` and `Controller` files

```bash
// Create routes file
mkdir src/Http && touch src/Http/routes.php

echo "<?php

Route::post('/', 'LoginController@index');
" > src/Http/routes.php

// Create controller file
mkdir src/Http/Controllers && touch src/Http/Controllers/LoginController.php

// Add the controller logic
echo "<?php

namespace Harlekoy\LastLogin\Http\Controllers;

use Harlekoy\LastLogin\Models\Login;
use Illuminate\Routing\Controller;

class LoginController extends Controller
{
    /**
     * Display a listing of the resource.
     *
     * @return \Illuminate\Http\Response
     */
    public function index()
    {
        return view('lastlogin::layout', [
            'logins' => Login::with('user')->latest()->get(),
        ]);
    }
}
" > src/Http/Controllers/LoginController.php
```

### User Interface

Now let start working on the UI, first let us uncomment the loading of views in our service provider

```diff
...skipping...
diff --git a/src/LastLoginServiceProvider.php b/src/LastLoginServiceProvider.php
index a1ea84f..a767ae4 100644
--- a/src/LastLoginServiceProvider.php
+++ b/src/LastLoginServiceProvider.php
@@ -20,13 +20,13 @@ class LastLoginServiceProvider extends ServiceProvider
                 __DIR__.'/../database/migrations/create_logins_tables.php.stub' => $this->getMigrationFileName($filesystem),
             ], 'migrations');

-            /*
-            $this->loadViewsFrom(__DIR__.'/../resources/views', 'skeleton');

             $this->publishes([
-                __DIR__.'/../resources/views' => base_path('resources/views/vendor/skeleton'),
+                __DIR__.'/../resources/views' => base_path('resources/views/vendor/lastlogin'),
             ], 'views');
-            */
         }
+
+        $this->registerRoutes();
+
+        $this->loadViewsFrom(__DIR__.'/../resources/views', 'lastlogin');
     }
+
+    /**
+     * Get the Telescope route group configuration array.
+     *
+     * @return array
+     */
+    private function routeConfiguration()
+    {
+        return [
+            'domain'     => config('lastlogin.domain', null),
+            'namespace'  => 'Harlekoy\LastLogin\Http\Controllers',
+            'prefix'     => config('lastlogin.path'),
+            'middleware' => 'web',
+        ];
+    }
+
+    /**
+     * Register the package routes.
+     *
+     * @return void
+     */
+    private function registerRoutes()
+    {
+        Route::group($this->routeConfiguration(), function () {
+            $this->loadRoutesFrom(__DIR__.'/Http/routes.php');
+        });
+    }
 }
```

After that lets create the view files

```bash
mkdir -p resources/views && touch resources/views/layout.blade.php
```

And add this HTML codes in the file, here I am using TailwindCSS CDN to style the admin dashboard we created

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Logins</title>
    <link href="https://unpkg.com/tailwindcss@^1.0/dist/tailwind.min.css" rel="stylesheet"></head>
    <body class="font-sans leading-none text-grey-darkest antialiased bg-gray-100">
        <div class="container mx-auto mt-20 px-24">
            <h1 class="mb-8 font-bold text-3xl">Logins</h1>
            <div class="bg-white rounded shadow overflow-x-auto">
                <table class="w-full whitespace-no-wrap">
                    <tr class="text-left font-bold">
                        <th class="px-6 pt-6 pb-4">Time</th>
                        <th class="px-6 pt-6 pb-4">Date</th>
                        <th class="px-6 pt-6 pb-4">Name</th>
                        <th class="px-6 pt-6 pb-4">Email</th>
                    </tr>
                    @if($logins->count())
                    @foreach($logins as $login)
                    <tr class="hover:bg-grey-lightest focus-within:bg-grey-lightest">
                        <td class="border-t">
                            <div class="px-6 py-4 flex items-center focus:text-indigo">
                                {{ $login->created_at->toTimeString() }}
                            </div>
                        </td>
                        <td class="border-t">
                            <div class="px-6 py-4 flex items-center" tabindex="-1">{{ $login->created_at->toDateString() }}</div>
                        </td>
                        <td class="border-t">
                            <div class="px-6 py-4 flex items-center" tabindex="-1">{{ $login->user->name }}</div>
                        </td>
                        <td class="border-t w-px">
                            <div class="px-4 flex items-center" tabindex="-1">
                                {{ $login->user->email }}
                            </div>
                        </td>
                    </tr>
                    @endforeach
                    @else
                    <tr>
                        <td class="border-t px-6 py-4 text-center" colspan="4">No users found.</td>
                    </tr>
                    @endif
                </table>
            </div>
        </div>
    </body>
</html>
```
> About HTML you can view it in `logins` page
