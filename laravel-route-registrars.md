---
title: Laravel routes without the facade or route files
slug: laravel-routes-without-the-facade-or-route-files
meta:
  keywords:
    - Laravel
    - Routing
    - Laravel Routing
    - Laravel Route Facade
    - Laravel Route Files
description: It has always felt weird that in a highly object-oriented framework, we'd still use procedural route files to define our routes, so in this article, I'm going to take a look at that and walk you through the process of using class-based registrars instead.
topics:
  - Laravel
  - Routing
publish_at: 02/02/28 06:66
---

For as long as I can remember, Laravel has had its routes defined in a PHP file that could arguably be considered procedural. In version 3, they were defined in `application/routes.php`, then in 4, it was `app/routes.php`, in 5.0, they moved to `app/Http/routes.php`, and in 5.3, the routes were split into `routes/web.php` and `routes/api.php`.

This particular approach has bugged me for some time, mostly because I often encounter applications with a single `routes/web.php` file that is huge, with few, if any, groups, and often, the routes won't have names, making it a nightmare for somebody that wasn't there for the creation of all of these routes.

When I was a contractor, I was often hired to tidy up Laravel applications and make them more manageable in the long term. One of the first things I did with every project was to break up the routes into groups, and then break them up further into separate files.

Once this was done, I would often try to tidy the routes further by replacing usages of the `Route` facade with an instance of Laravels router. This is a relatively simple thing to do, as the code that handles the inclusion of the files, as defined in the `RouteServiceProvider`, has the router available as `$router`. All you needed to do to make your routes compatible was to add a docblock to the top of the routes file.

```php
/**
 * @var \Illuminate\Routing\Router $router
 */
```

Then in any group defined using a closure, you'd add the router as a parameter.

```php
$router->group(function (Router $router) {
    // Routes here
});
```

I did this because I'm not too fond of facades and instead opt for dependency injection. Do not get me wrong; I'm not going to insist that you all stop using them or that they should be removed. I get that they have a purpose and that some people like using them, but I don't, so I don't.

For the longest time, this was how I defined my routes across all my projects, personal and client. 

Not too long ago, it suddenly struck me as odd the distance Laravel has come since I started using it back in 2012, and we're still using procedural files to define our routes. At this point, I had the idea to start using classes to group and define my routes, bringing them much more in line with OOP principles.

I'm aware this idea isn't unique, and there are probably many others who also do this, but I'd like to share it with you anyway.

## Defining the route registrar

It starts with the `RouteRegistrar` contract, which I will typically put in `app/Contracts`.

```php
namespace App\Contracts;

use Illuminate\Contracts\Routing\Registrar;

interface RouteRegistrar
{
    public function map(Registrar $router): void;
}
```

Route registrars are simple; they have a single method `map`, which takes an instance of `Registrar`[^registrar-router]. The body of this method will function the same way as the individual route definition files.

To give you a better idea of what I mean, let's take the default [`routes/web.php` file](https://github.com/laravel/laravel/blob/9.x/routes/web.php).

```php
<?php

use Illuminate\Support\Facades\Route;

/*
|--------------------------------------------------------------------------
| Web Routes
|--------------------------------------------------------------------------
|
| Here is where you can register web routes for your application. These
| routes are loaded by the RouteServiceProvider within a group which
| contains the "web" middleware group. Now create something great!
|
*/

Route::get('/', function () {
    return view('welcome');
});
```

And turn it into a route registrar.

### Creating the first route registrar

For this, I'm going to create a class called `DefaultRoutes` in a newly created directory `app/Http/Routes`[^route-location], which will implement the `RouteRegistrar` contract.

```php
namespace App\Http\Routes;

use App\Contracts\RouteRegistrar;
use Illuminate\Contracts\Routing\Registrar;

class DefaultRoutes implements RouteRegistrar
{
    public function map(Registrar $router): void
    {
    }
}
```

Once I have defined this class, I can redefine the route from the routes file by using the `$router` variable instead of the facade.

```php
namespace App\Http\Routes;

use App\Contracts\RouteRegistrar;
use Illuminate\Contracts\Routing\Registrar;

class DefaultRoutes implements RouteRegistrar
{
    public function map(Registrar $router): void
    {
        $router->get('/', function () {
            return view('welcome');
        });
    }
}
```

As you can see here, the most significant difference is that the route now exists within a class, which any modern IDE can help you navigate, either with a simple class search or by clicking on any reference to its type.

While this feels nicer and makes working with the routes nicer, the route service provider has no idea how to use the route registrars.

## Making use of route registrars

Instead of writing the functionality directly into the route service provider, I'm going to create a trait, commonly referred to as a concern in the world of Laravel, that encapsulates all of the functionality required to load route registrars. It will become clear later on in the article why I have done this.

The trait I'm going to create will be called `MapsRouteRegistrars` and will exist in the newly created directory `app/Concerns`.

```php
namespace App\Concerns;

trait MapsRouteRegistrars
{
    
}
```

The trait itself will be straightforward; it will contain a single protected method called `mapRoutes`, which takes an instance of the `Registrar` contract[^registrar-router], and an array of route registrar classes.

```php
protected function mapRoutes(Registrar $router, array $registrars): void
{
    
}
```

Inside this method, it's going to loop through the `$registrars` array, create new instances, and call the `RouteRegistrar::map()` method.

```php
protected function mapRoutes(Registrar $router, array $registrars): void
{
    foreach ($registrars as $registrar) {
        (new $registrar)->map($router);
    }
}
```

While this works, it is prone to bugs, as there's nothing in here to ensure that the values provided in `$registrars` are classes or even that they implement the correct contract[^defensive-coding]. 

So, if the array item for the current loop isn't a valid class, we'll want to throw an exception.

```php
protected function mapRoutes(Registrar $router, array $registrars): void
{
    foreach ($registrars as $registrar) {
        if (! class_exists($registrar)) {
            throw new RuntimeException(sprintf(
                'Cannot map routes \'%s\', it is not a valid routes class',
                $registrar
            ));
        }
        
        (new $registrar)->map($router);
    }
}
```

We can now be sure it's a class, but not that it implements the correct interface. This can be done in one of two ways.

The first is to create the instance and then check it using an `instanceof` check.

```php
protected function mapRoutes(Registrar $router, array $registrars): void
{
    foreach ($registrars as $registrar) {
        if (! class_exists($registrar)) {
            throw new RuntimeException(sprintf(
                'Cannot map routes \'%s\', it is not a valid routes class',
                $registrar
            ));
        }
        
        $instance = new $registrar;
        
        if (! ($instance instanceof RouteRegistrar)) {
            throw new RuntimeException(sprintf(
                'Cannot map routes \'%s\', it is not a valid routes class',
                $registrar
            ));
        }
        
        $instance->map($router);
    }
}
```

I'm not too fond of this approach because it duplicates the exception, which means that you probably want to abstract that out to a method so that it can be called without duplicating code, which is a little excessive for a straightforward method.

You could combine them by declaring the `$instance` variable in the `if` condition.

```php
protected function mapRoutes(Registrar $router, array $registrars): void
{
    foreach ($registrars as $registrar) {
        if (! class_exists($registrar) || ! (($instance = new $registrar) instanceof RouteRegistrar)) {
            throw new RuntimeException(sprintf(
                'Cannot map routes \'%s\', it is not a valid routes class',
                $registrar
            ));
        }
        
        $instance->map($router);
    }
}
```

But despite this being an approach commonly taken within the Laravel codebase, it's something that's never sat right with me. On top of that, also, I'm not too fond of the idea of instantiating a class that could be anything, as you don't know what the side effects could be, and you'd probably end up spending a long time debugging any potential issues.

So instead, I'm going to go for the second option, which is to use the [`is_subclass_of` function](https://www.php.net/is_subclass_of) as part of the `if` condition.

```php
protected function mapRoutes(Registrar $router, array $registrars): void
{
    foreach ($registrars as $registrar) {
        if (! class_exists($registrar) || ! is_subclass_of($registrar, RouteRegistrar::class)) {
            throw new RuntimeException(sprintf(
                'Cannot map routes \'%s\', it is not a valid routes class',
                $registrar
            ));
        }
        
        (new $registrar)->map($router);
    }
}
```

This is a much cleaner approach to the first one and avoids all the potential issues caused by instantiating an unknown class. It also makes the initial `class_exists` check redundant because if `$registrar` is not a valid class, then it can't be a subclass of the `RouteRegistrar` contract.

```php
protected function mapRoutes(Registrar $router, array $registrars): void
{
    foreach ($registrars as $registrar) {
        if (! is_subclass_of($registrar, RouteRegistrar::class)) {
            throw new RuntimeException(sprintf(
                'Cannot map routes \'%s\', it is not a valid routes class',
                $registrar
            ));
        }
        
        (new $registrar)->map($router);
    }
}
```

## Mapping the registrars

With the trait complete, I can go to the [`RouteServiceProvider`](https://github.com/laravel/laravel/blob/9.x/app/Providers/RouteServiceProvider.php)[^route-service-provider] and add it.

```php
namespace App\Providers;

use Illuminate\Foundation\Support\Providers\RouteServiceProvider as ServiceProvider;
use App\Concerns\MapsRouteRegistrars;

class RouteServiceProvider extends ServiceProvider
{
    use MapsRouteRegistrars;
    
    public function boot()
    {
        $this->routes(function () {
            Route::middleware('api')
                ->prefix('api')
                ->group(base_path('routes/api.php'));

            Route::middleware('web')
                ->group(base_path('routes/web.php'));
        });
    }
}
```

With the trait in place, I will create a new property called `registrars` to hold the `RouteRegistrar` classes.

```php
namespace App\Providers;

use Illuminate\Foundation\Support\Providers\RouteServiceProvider as ServiceProvider;
use App\Concerns\MapsRouteRegistrars;
use App\Http\Routes\DefaultRoutes;

class RouteServiceProvider extends ServiceProvider
{
    use MapsRouteRegistrars;
    
    protected array $registrars = [
        DefaultRoutes::class
    ];
    
    public function boot()
    {
        $this->routes(function () {
            Route::middleware('api')
                ->prefix('api')
                ->group(base_path('routes/api.php'));

            Route::middleware('web')
                ->group(base_path('routes/web.php'));
        });
    }
}
```

Since the closure passed into the `routes` method is passed through the container, you can use its parameters for injection, which will allow me to inject the core router using the `Registrar` contract, and then finally pass both it and the `registrars` property into the newly created method provided by the trait.

```php
namespace App\Providers;

use Illuminate\Foundation\Support\Providers\RouteServiceProvider as ServiceProvider;
use Illuminate\Contracts\Routing\Registrar;
use App\Concerns\MapsRouteRegistrars;
use App\Http\Routes\DefaultRoutes;

class RouteServiceProvider extends ServiceProvider
{
    use MapsRouteRegistrars;
    
    protected array $registrars = [
        DefaultRoutes::class
    ];
    
    public function boot()
    {
        $this->routes(function (Registrar $router) {
            $this->mapRoutes($router, $this->registrars)
        });
    }
}
```

This is now complete, with a full route registrar implementation. That being said, a significant issue still needs to be addressed.

## Recreating the default routes and groups

The old way of mappings routes created two groups that applied a particular middleware set to them, which is needed for a good chunk of Laravels functionality to function well.

First, there's the [`API` group](https://github.com/laravel/laravel/blob/9.x/app/Http/Kernel.php#L41).

```php
Route::middleware('api')
    ->prefix('api')
    ->group(base_path('routes/api.php'));
```

And then there's the [`web` group](https://github.com/laravel/laravel/blob/9.x/app/Http/Kernel.php#L32).

```php
Route::middleware('web')
    ->group(base_path('routes/web.php'));
```

 This brings me back to the reason I created the `MapsRouteRegistrars` trait. Back in the `app/Http/Routes` directory, I will make a `WebRoutes` class that implements `RouteRegistrar`.

```php
namespace App\Http\Routers;

use App\Contracts\RouteRegistrar;
use Illuminate\Contracts\Routing\Registrar;

class WebRoutes implements RouteRegistrar
{
    public function map(Registrar $router): void
    {
    }
}
```

But unlike the previously created `DefaultRoutes`, where I defined routes, I'm going to mirror the same process as the route service provider by using the trait, adding a property, and calling `mapRoutes`, but from inside a group definition that matches the previous one.

```php
namespace App\Http\Routers;

use App\Contracts\RouteRegistrar;
use Illuminate\Contracts\Routing\Registrar;
use App\Concerns\MapsRouteRegistrars;

class WebRoutes implements RouteRegistrar
{
    use MapsRouteRegistrars;
    
    protected array $registrars = [
        DefaultRoutes::class
    ];
    
    public function map(Registrar $router): void
    {
        $router->group([
            'middleware' => 'web'
        ], function (Registrar $router) {
	        $this->mapRoutes($router, $this->registrars);
        });
    }
}
```

It is worth noting that the `Registrar` contract only allows access to the old `group` method that requires all of the attributes in an array as the first argument.

I'm now going to duplicate this to a new class called `ApiRoutes` and update the route group to match the previously defined API route group.

```php
namespace App\Http\Routers;

use App\Contracts\RouteRegistrar;
use Illuminate\Contracts\Routing\Registrar;
use App\Concerns\MapsRouteRegistrars;

class ApiRoutes implements RouteRegistrar
{
    use MapsRouteRegistrars;
    
    protected array $registrars = [];
    
    public function map(Registrar $router): void
    {
        $router->group([
            'prefix'     => 'api',
            'middleware' => 'api'
        ], function (Registrar $router) {
	        $this->mapRoutes($router, $this->registrars);
        });
    }
}
```

I can separate the individual web and API groups with these route registrars while maintaining easy access.

For the default API route that is typically part of the `routes/api.php` file, I need to define a route registrar, but since `DefaultRoutes` is already taken, I'm going to have to reorganise. To keep with the spirit of separation, I'm going to move the existing default routes class to `app/Http/Routes/Web`[^default-routes-namespace] and update the `WebRoutes` registrar to use that namespace.

```php
namespace App\Http\Routers;

use App\Contracts\RouteRegistrar;
use Illuminate\Contracts\Routing\Registrar;
use App\Concerns\MapsRouteRegistrars;
use App\Http\Routes\Web;

class WebRoutes implements RouteRegistrar
{
    use MapsRouteRegistrars;
    
    protected array $registrars = [
        Web\DefaultRoutes::class
    ];
    
    public function map(Registrar $router): void
    {
        $router->group([
            'middleware' => 'web'
        ], function (Registrar $router) {
	        $this->mapRoutes($router, $this->registrars);
        });
    }
}
```

Now I can safely create another `DefaultRoutes` in `app/Http/Routes/Api`, which contains the default API route.

```php
namespace App\Http\Routes\Api;

use App\Contracts\RouteRegistrar;
use Illuminate\Contracts\Routing\Registrar;
use Illuminate\Http\Request;

class DefaultRoutes implements RouteRegistrar
{
    public function map(Registrar $router): void
    {
        $router->get('/user', function (Request $request) {
    		return $request->user();
        })->middleware('auth:sanctum');
    }
}
```

Then I can add this to the registrar's array in `ApiRoutes`.

```php
namespace App\Http\Routers;

use App\Contracts\RouteRegistrar;
use Illuminate\Contracts\Routing\Registrar;
use App\Concerns\MapsRouteRegistrars;
use App\Http\Routes\Api;

class ApiRoutes implements RouteRegistrar
{
    use MapsRouteRegistrars;
    
    protected array $registrars = [
        Api\DefaultRoutes::class
    ];
    
    public function map(Registrar $router): void
    {
        $router->group([
            'prefix'     => 'api',
            'middleware' => 'api'
        ], function (Registrar $router) {
	        $this->mapRoutes($router, $this->registrars);
        });
    }
}
```

And finally, I can go back to the `RouteServiceProvider` and update to use these two registrars.

```php
namespace App\Providers;

use Illuminate\Foundation\Support\Providers\RouteServiceProvider as ServiceProvider;
use Illuminate\Contracts\Routing\Registrar;
use App\Concerns\MapsRouteRegistrars;
use App\Http\Routes\ApiRoutes;
use App\Http\Routes\WebRoutes;

class RouteServiceProvider extends ServiceProvider
{
    use MapsRouteRegistrars;
    
    protected array $registrars = [
        ApiRoutes::class,
        WebRoutes::class
    ];
    
    public function boot()
    {
        $this->routes(function (Registrar $router) {
            $this->mapRoutes($router, $this->registrars)
        });
    }
}
```

It should be noted that although the route definitions are broken up into multiple files, the order in which they are defined is still important, as ultimately, they're all defined on the same underlying router instance, which is why I have `ApiRoutes` about `WebRoutes`.

## Taking it further

There's no limit to how you want this to work; the benefit of having the `MapsRouteRegistrars` trait is that you can create route registrars that function as groups, registering yet more registrars.

One of the things I also like to do is altogether remove the `routes` directory, removing the reference `routes/channels.php` from the `BroadcastServiceProvider`, and the reference to `routes/console.php` from the console `Kernel`. This isn't strictly necessary, but it's worth deleting `web.php` and `api.php` to clean up.

This approach won't be for everyone; some of you who lean more towards my way of thinking may find that it makes routes feel nicer and cleaner, but some of you will like the current approach or consider it an over-abstraction. 

Whatever you do with this, and however you feel about it, I hope there was something in here that helped, something for you to take away.



[^registrar-router]: I opt to use the [`Registrar` contract](https://github.com/laravel/framework/blob/9.x/src/Illuminate/Contracts/Routing/Registrar.php) for my parameter type, but you can use the [`Router` concrete](https://github.com/laravel/framework/blob/9.x/src/Illuminate/Routing/Router.php) if you want, as it is the default implementation.
[^route-location]: The location of your route registrars is unimportant as long as the composer autoloader knows how to find them.
[^defensive-coding]: You may be okay with the risk, but I strongly advise that you attempt to mitigate any potential issues, where possible. There's nothing wrong with defensive coding.
[^route-service-provider]: This is the default service provider tidied up a little to strip out the irrelevant bits.
[^default-routes-namespace]: Make sure to add the `Web` part to the namespace declaration in the `DefaultRoutes.php` file.