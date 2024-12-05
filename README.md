# laravel-jetstream-impersonate
Adds the ability for administrators to impersonate users for testing

## My Project Architecture
Im using Laravel Jetstream with livewire. Im not really using livewire anywhere other than whats built into jetstream. Im not using teams. I added jQuery, and mostly call the api's in order retrieve data. Then I populate templates with that data and render it.

## jQuery
Just as an FYI, here is my jquery code to make my ajax calls. This is just to illistrate that when I'm making calls within the app, I'm not sending in the bearer token. Thats only used by 3rd party applications that want to consume my public api.

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
On the user model `app\Models\User.php` I added two funcitons, one that says only admin users can impersonate others, and another saying only non admins can be impersonated. I have an integer on the user table that corresponds to Role model, which is basically just an enum.
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
The middleware `app\Http\Middleware\Impersonate.php` is the magic here, if the session has the impersonate attribute, we look up the user and then set the user. Even though this middleware will be called with every web and api request, it should be quick to execute because of the if.
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
Next we need to modify `app\Http\Kernel.php` to add the middleware to both the web and api stacks.
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
        \Illuminate\Routing\Middleware\ThrottleRequests::class . ':api',
        \Illuminate\Routing\Middleware\SubstituteBindings::class,
        \App\Http\Middleware\Impersonate::class, // <<<< Added
    ],
];
```

## Controller
This controller `app\Http\Controllers\ImpersonationController.php` is was sets/destroys the impersonate attribute in the session. It kicks off and ends the session, so the middleware acts accordingly.
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
    // Remove impersonation data from the session
    session()->forget('impersonate');

    return redirect()->route('dashboard');
}
```

## Web Routes
The bulk of my routes in `routes\web.php` are within the auth:santcum middleware protection which requires the user to be logged in. I placed these routes within the auth:sanctum main block.
```php
Route::post('/impersonate/{user}', [ImpersonationController::class, 'start'])->name('impersonate.start');
Route::get('/impersonate/stop', [ImpersonationController::class, 'stop'])->name('impersonate.stop');
```

## Main Layout
In `resources\views\layouts\app.blade.php` I have the following code. This will ensure you remember your impersonating a user. You could put this link in the main navigation, or wherever, but I choose to have it be a banner at the top of the window. I also have a test button defined. Once everything is set up and working we will test using this button as a regular user and as an admin. In my case I am impersonating user 5 which is a regular user. Of course you will remove this form & button once everything is tested.
```blade
@if (session()->has('impersonate'))
<div class="alert alert-danger text-center" role="alert">
    <a href="{{ route('impersonate.stop') }}" class="btn btn-danger">Leave Impersonation</a>
</div>
@endif

<form method="POST" action="{{ route('impersonate.start', 5) }}">
    @csrf
    <button type="submit">Impersonate 5</button>
</form>
```

## Admin Dashboard
Now that I have everything tested and working, I went to the admin panel. I wrote functionality to list all user, with search and pagination. On each user row there is a dropdown menu with an impersonate option. When that is clicked I get the user id, then i use that to call a js method.
```blade
<form id="frmImpersonate" method="POST">
    @csrf
</form>
```
```javascript
// When admin clicks on the impersonate button
$("#contUsers").on("click", ".user_impersonate", function (e) {
    e.preventDefault();

    // Get the user
    let idx = $(this).closest(".userRow").index(),
        user = app.users.list[idx];

    // Log the user
    console.log(user);

    // Populate the forms action url 
    $('#frmImpersonate').attr('action', 'impersonate/' + user.id);

    // Submit the form
    $('#frmImpersonate').submit();
});
```
## Test Stop impersonate as regular user
Since we are just using a `GET` to stop the impersonation, you can test as a regular user and as an admin, whether you are impersonating or not, by going to `https://mysite/impersonate/stop`

## Links
Here are some tuts that I used to help get this working. There are also two libs if you dont want to roll your own.
-  https://mauricius.dev/easily-impersonate-any-user-in-a-laravel-application/
-  https://laracasts.com/discuss/channels/laravel/sanctum-still-sees-auth-as-main-user-when-impersonating
-  https://github.com/404labfr/laravel-impersonate (How To: https://laravel-news.com/laravel-impersonate)
-  https://github.com/OctopyID/LaraPersonate
