## Laravel Multiple Authentication
This repository was made to help Laravel developers in making a fully-customizable multiple authentication, without depending on the default `auth` middleware. We'll make a new middleware `role` to handle all roles and logging in redirections in your Laravel project.

## Pre-requirements
- You need to setup Laravel authentication previously. 
- Add a `role` column in **users** table. 
  > I prefer to make it **INT**. 0 for admins role, 1 for regular users role and make 1 its default value.
- Create 2 separate dashboards blades, routes and Controllers for admins and users.

## Get Started
Go to **_app/Http/Controllers/Auth/LoginController.php_** and find redirect method line:
```php
protected $redirectTo = RouteServiceProvider::HOME;
```
Then, replace it with the below new method which checks role and returns role's route.
```php
public function redirectTo() {
   $role = Auth::user()->role;    // Getting role from users table.
   switch ($role) {
       case 0:    // 0 = Admin
           return '/admin';
           break;
       case 1:    // 1 = User
           return '/user';
           break;
       default:
           return '/home';
           break;
   }
}
```

And because we're using `Auth` class, we need to include it at the top of **_LoginController.php_** by:
```php
use Illuminate\Support\Facades\Auth;
```

Also, we've to apply the same conditional logic to another file, so go to **_app/Http/Middleware/RedirectIfAuthenticated.php_**
Find `handle()` method and replace all of it to the below method
```php
public function handle($request, Closure $next, $guard = null) {
    if (Auth::guard($guard)->check()) {
        $role = Auth::user()->role;
        switch ($role) {
            case 0:
                return redirect('/admin');
                break;
            case 1:
                return redirect('/user');
                break;
            default:
                return redirect('/home');
                break;
        }
    }
    return $next($request);
}
```

Then, create a trait named RolesTrait. It'll be the redirect guide and will return the respective login route depending on the requested page.
Go to **_app/_** and make a directory named **_Traits_** and create a file named **_RolesTrait.php_** and put the below trait into it:
```php
<?php

namespace App\Traits;

trait RolesTrait {
    public function rolesLoginRoutes($role) {
        $roles = [
            '0' => route('admin.login'), // Route of admin login
            '1' => route('login'), // Route of regular user login
        ];
        return $roles[$role];
    }
}
```

Now we have to protect our routes! We don't want a regular user visit admin dashboard, so we'll create a new middleware named **Role** by running this command in the terminal:
```shell
php artisan make:middleware Role
```

Then, go to **_app/Http/Middleware/Role.php_** the file we've created and write this method to handle which role'll be used now:
```php
<?php

namespace App\Http\Middleware;
use App\Traits\RolesTrait;
use Closure;
use Illuminate\Support\Facades\Auth;

class Role {
    use RolesTrait;
    public function handle($request, Closure $next, $role) {
        if (!Auth::check())
            return redirect($this->rolesLoginRoutes($role));
        $user = Auth::user();
        if($user->role == $role)
            return $next($request);
        return redirect('/home');
    }
}
```

Don't forget to register the `Role` middleware at **_app/Http/Kernal.php_**, go to it and register the middleware we created like below:
```php
protected $routeMiddleware = [
    # ...
    # ... past registered middlewares 
    # ...
    'role' => \App\Http\Middleware\Role::class, // Add this.
];
```

Finally, apply this middleware in your routes like below:
```php
Route::get('/user', 'Auth\YourController@user')->middleware('role:1');    // 1 = User
Route::group(['prefix' => 'admin', 'namespace' => 'Auth'], function () {
    Route::get('/', 'YourController@admin')->middleware('role:0');    // 0 = Admin
    Route::get('/login', 'YourController@adminLogin')->name('admin.login');
});
```
.  
.  
.
### Then watch how the world is running!
