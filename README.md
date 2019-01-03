# Nova Page

Ever wanted to expose static content of an "About" page as editable fields in your app's administration without having to create specific models & migrations? Using this package, you'll be able to do so. By default, it will store the content in JSON files in the application's `resources/lang` directory, making them available for version control.

This package adds basic **flat-file CMS features** to Laravel Nova in a breeze using template configurations as it were administrable Laravel Models, meaning it allows the usage of all the available Laravel Nova fields and tools.

## Installation

In your terminal type : `composer require whitecube/nova-page` and provide "dev-master" as the version of the package. Or open up composer.json and add the following line under `require`:

```json
    {
        "require": {
            "whitecube/nova-page": "dev-master"
        }
    }
```

Register the package in the `providers` section of the app config file in `app/config/app.php`:

```php
    'providers' => [
        // ...
        
        /*
         * Package Service Providers...
         */
        Whitecube\NovaPage\NovaPageServiceProvider::class,
        // ...
    ],
```

Next, add the `Page` facade:

```php
    'aliases' => [
        // ...
        'Page' => Whitecube\NovaPage\NovaPageFacade::class,
        // ...
    ],
```

Finally, register the Nova tool in `app/Http/Providers/NovaServiceProvider.php`:

```php
    /**
     * Get the tools that should be listed in the Nova sidebar.
     *
     * @return array
     */
    public function tools()
    {
        return [
            \Whitecube\NovaPage\NovaPageTool::make(),
        ];
    }
```

Now you can publish the package's configuration file with the `php artisan vendor:publish` command. This will add a `app/config/novapage.php` file containing the package's default configuration.

## Templates

In order to assign fields (and even cards!) to page's edition form, we'll have to create a `Template` class and register this class on one or more routes. You'll see, it's quite easy.

### Creating Templates

Each Template is defined in an unique class, just as Resources. You can store those classes wherever you want, but a new `app/Nova/Templates` directory is probably a good choice.

```php
namespace App\Nova\Templates;

use Illuminate\Http\Request;
use Whitecube\NovaPage\Pages\Template;

class Aboutpage extends Template
{

    /**
     * Get the fields displayed by the resource.
     *
     * @param  \Illuminate\Http\Request $request
     * @return array
     */
    public function fields(Request $request)
    {
        return [];
    }

    /**
     * Get the cards available for the request.
     *
     * @param  \Illuminate\Http\Request  $request
     * @return array
     */
    public function cards(Request $request)
    {
        return [];
    }
}
```

Fields and cards defintion is exactly the same as on regular [Laravel Nova Resources](https://nova.laravel.com/docs/1.0/resources/fields.html#defining-fields).

### Assigning templates

Once the template is defined, simply assign it to the chosen routes using the `template()` method (which is added to the original `Route` api by NovaPage):

```php
Route::get('/about', 'AboutController@show')
    ->template(\App\Nova\Templates\Homepage::class)
    ->name('about');
```

**Important**: Do not forget to name the routes you'll be using with NovaPage templates. Route names are used for retrieving and naming the JSON files.


## Loading pages for display

### Middleware autoloading

It is possible to load the page's static content automatically using the `LoadPageFromRouteName` middleware. This way, the application will fetch the page's data using the current route's name as identifier. Of course, this means you'll need to name the routes in order to get it to work.

Add `\Whitecube\NovaPage\Http\Middleware\LoadPageFromRouteName::class` to the `routeMiddleware` array located in the `App\Http\Kernel` file:

```php
    /**
     * The application's route middleware.
     *
     * These middleware may be assigned to groups or used individually.
     *
     * @var array
     */
    protected $routeMiddleware = [
        // ...
        'loadNovaPage' => \Whitecube\NovaPage\Http\Middleware\LoadPageFromRouteName::class,
    ];
```

You can now assign the `loadNovaPage` middleware to all routes that need it, or even add it to the `web` middleware group in the same `App\Http\Kernel` file:

```php
    /**
     * The application's route middleware groups.
     *
     * @var array
     */
    protected $middlewareGroups = [
        'web' => [
            // ...
            'loadNovaPage',
        ],
        // ...
    };
```

### Manual loading

At any time, pages can be loaded using the package's Page Manager. Simply type-hint the `Whitecube\NovaPage\Pages\Manager` dependency in a controller and call its `load($name, $template, $locale = null, $current = true)` method:

```php
use App\Nova\Templates\Aboutpage;
use Whitecube\NovaPage\Pages\Manager;

class AboutController extends Controller
{

    public function show(Manager $page)
    {
        $page->load('about', Aboutpage::class);
        return view('pages.about');
    }

}
```

In most cases, this will probably not be very convenient since we alreay registered the current route's template in our routes definitions. Therefore, it is also possible to load the page's content with an `Illuminate\Routing\Route` instance using the `loadFromRoute($route, $locale = null, $current = true)` method:

```php
use Illuminate\Http\Request;
use Whitecube\NovaPage\Pages\Manager;

class AboutController extends Controller
{

    public function show(Request $request, Manager $page)
    {
        $page->loadFromRoute($request->route());
        return view('pages.about');
    }

}
```

Anyway, if no locale is provided, NovaPage will use the application's current locale (using `App::getLocale()`). By default, loading a page's content will define it as the current page, making its attributes accessible with the `Page` facade. If you just want to load content without setting it as the current page, you should call `load()` or `loadFromRoute()` with `$current` set to `false`.

## Front-end Template Usage

Retrieving the page's static values in your application's blade templates is made possible using the `Page` facade and its different methods:

```blade
@extends('layout')

@section('pageName', Page::name())

@section('content')
    <h1>{{ Page::title('Default title', 'My website: ', ' • Awesome appended string') }}</h1>
    <p>Edited on <time datetime="{{ Page::date('updated_at')->format('c') }}">{{ Page::date('updated_at')->toFormattedDateString() }}</time></p>
    <p>{{ Page::get('introduction') }}</p>
    <a href="{!! Page::get('cta.href') !!}">{{ Page::get('cta.label') }}</a>
@endsection
```

### Useful Facade methods

#### `Page::name()`

Returns the page's name (usually the route's name).

#### `Page::title($default = null, $prepend = null, $append = null)`

Returns and optionally formats the page's title. The title is an automatic & required field and **is not** linked to or overwritten by an alternative `title` attribute you could add to the page's fields.

#### `Page::locale()`

Returns the locale code of the loaded page. Usually (and should be) the same as `App::getLocale()`.

#### `Page::date($timestamp = 'created_at')`

Returns a [Carbon](https://carbon.nesbot.com/) instance of the requested timestamp. Possible timestamps are:

- `created_at`
- `updated_at`

> **Note**: Since most UNIX systems do not have a creation date for their files, the `created_at` and `updated_at` timestamps are stored in the file's JSON attributes. Keep this in mind if you want to edit the files in a text editor.

#### `Page::get($attribute, $callback = null)`

Returns a defined field's value. Optionally, you can provide a callback `Closure` that will be applied to the returned value. 

### Dependency Injection

Alternatively, it's also possible to type-hint the current `Whitecube\NovaPage\Page\Template` in classes resolved by Laravel's [Service Container](https://laravel.com/docs/container), such as controllers. **The page needs to be loaded before** the `Page\Template` is requested, which can be easily achieved using the package's `LoadPageFromRouteName` middleware.

```php
use Whitecube\NovaPage\Pages\Template;

class HomepageController extends Controller
{

    public function show(Template $template)
    {
        // If needed, manipulate $template's attribute before passing it to the view.
        return view('pages.home', ['page' => $template]);
    }

}
```

And use it as a regular object in the `pages.home` template:

```blade
@extends('layout')

@section('pageName', $page->getName())

@section('content')
    <h1>{{ $page->getTitle('Default title', 'My website: ', ' • Awesome appended string') }}</h1>
    <p>Edited on <time datetime="{{ $page->getDate('updated_at')->format('c') }}">{{ $page->getDate('updated_at')->toFormattedDateString() }}</time></p>
    <p>{{ $page->introduction }}</p>
    <a href="{!! $page->cta->href !!}">{{ $page->cta->label }}</a>
@endsection
```

As you can see, for convenience regular attributes (= defined fields) can be directly retrieved as properties of the `Whitecube\NovaPage\Pages\Template` instance.