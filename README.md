# laravel-jetstream-impersonate
Adds the ability for administrators to impersonate users for testing

## My Project Architecture
Im using Laravel Jetstream with livewire. I then ripped out tailwind, and put bootstrap back in. Im not really using livewire anywhere other than whats built into jetstream. Im not using teams. I added jQuery, and mostly call the api's in order retrieve data. Then I populate templates with that data and render it.

## jQuery
Just as an FYI, here are my jquery to make my ajax calls. This is just to illistrate that when im making calls within the app, im not sending in the bearer token. Thats only used by 3rd party applications that want to consume my public api.

POST:
```javascript
 let url = `api/v1/my/path`,
    obj = {
        url: url,
        method: "POST",
        data: JSON.stringify(o),
        headers: {
            "X-XSRF-TOKEN": Cookies.get("XSRF-TOKEN"),
            "Content-Type": "application/json",
        },
    };


return $.ajax(obj)
    .fail(function () {
        alert("Call Failed.");
    })
    .done(function (data, textStatus, xhr) {
    });
```

GET:
```javascript
let url = "api/v1/my/path",
    obj = {
        url: url,
    };

return $.ajax(obj)
    .fail(function () {
        alert("Call Failed.");
    })
    .done(function (data, textStatus, xhr) {
    });
```

## User model
On the user model I added two funcitons, one that says only admin users can impersonate others, and another saying only non admins can be impersonated. I have a integer on the user table that corresponds to Role model, which is basically just an enum.
```php
 public function canImpersonate()
{
    return $this->role_id == Role::ADMIN;
}
public function canBeImpersonated()
{
    return $this->role_id != Role::ADMIN;
}
```

## Impersonate middleware
```php
 public function handle(Request $request, Closure $next): Response
    {
       
        // Check if impersonation is active
        if (Session::has('impersonate')) {
            $impersonatedUserId = Session::get('impersonate');

            // Find and set the impersonated user
            $impersonatedUser = \App\Models\User::find($impersonatedUserId);
            if ($impersonatedUser && $impersonatedUser.canBeImpersonated()) {
                Auth::setUser($impersonatedUser);
            }
        }

        return $next($request);
    }
```

## Kernel
Next we need to modify app\Http\Kernel.php to add the middleware
```php
protected $middlewareGroups = [
    'web' => [
        \App\Http\Middleware\EncryptCookies::class,
        \Illuminate\Cookie\Middleware\AddQueuedCookiesToResponse::class,
        \Illuminate\Session\Middleware\StartSession::class,
        \Illuminate\View\Middleware\ShareErrorsFromSession::class,
        \App\Http\Middleware\VerifyCsrfToken::class,
        \Illuminate\Routing\Middleware\SubstituteBindings::class,
        \App\Http\Middleware\Impersonate::class, // <<<< Added
    ],

    'api' => [
        \Laravel\Sanctum\Http\Middleware\EnsureFrontendRequestsAreStateful::class,
        \App\Http\Middleware\Impersonate::class, // <<<< Added
        \Illuminate\Routing\Middleware\ThrottleRequests::class . ':api',
        \Illuminate\Routing\Middleware\SubstituteBindings::class,
    ],
];
```

## Controller

```php
public function start(Request $request, $userId)
{
    // Ensure the current user has permission to impersonate
    if (!Auth::user()->can('impersonate')) {
        abort(403, 'Unauthorized action.');
    }

    Session::put('impersonate', $userId);

    return redirect()->route('dashboard');
}

public function stop()
{
    if (!session()->has('impersonate')) {
        return redirect()->route('dashboard');
    }

    // Remove impersonation data from the session
    session()->forget('impersonate');

    return redirect()->route('dashboard');
}
```

## Web Routes
I have routes that arent protected, and then the bulk of my routes are within the auth:santcum middleware. I also have a section within the auth:sanctum for the admin prefix for further protection. This is all to say that I add these two routes outside of all that, and its at the unprotected level.
```php
Route::middleware(['auth:web'])->group(function () {
    Route::post('/impersonate/{user}', [ImpersonationController::class, 'start'])->name('impersonate.start');
    Route::post('/impersonate/stop', [ImpersonationController::class, 'stop'])->name('impersonate.stop');
});
```

## Admin Dashboard
To test these new endpoints I added this code. You can see I statically have it set to impersonate user 5. This gets replaced once everything is verified to work. 
```blade
 @can('impersonate')
  <form method="POST" action="{{ route('impersonate.start', 5) }}">
      @csrf
      <button type="submit">Impersonate 5</button>
  </form>
  @endcan

  @if (session()->has('impersonate'))
  <form method="POST" action="{{ route('impersonate.stop') }}">
      @csrf
      <button type="submit">Stop Impersonating</button>
  </form>
  @endif
```
