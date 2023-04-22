# Laravel Sanctum's Session authentication with Next.js SSR

```
composer require laravel/ui

php artisan ui:auth
```
#### Laravel Sanctum
```
composer require laravel/sanctum

php artisan vendor:publish --provider="Laravel\Sanctum\SanctumServiceProvider"

php artisan migrate
```

Add Sanctum's middleware to your api middleware group within your `app/Http/Kernel.php` file:

```php

'api' => [
    \Laravel\Sanctum\Http\Middleware\EnsureFrontendRequestsAreStateful::class,
    'throttle:60,1',
    \Illuminate\Routing\Middleware\SubstituteBindings::class,
],
```

.env
```
...
SESSION_DRIVER=cookie
...
SANCTUM_STATEFUL_DOMAINS=localhost,localhost:8000,localhost:3000,127.0.0.1,127.0.0.1:8000,127.0.0.1:3000,::1
```

##### Create a User
```
php artisan tinker

User::create(['name'=>'John Doe', 'email'=>'johndoe@example.org', 'password'=>Hash::make('secret')]);
```

##### Configurations

In `routes/api.php`

```php
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Route;

Route::middleware('auth:sanctum')->get('/user', function (Request $request) {
    return $request->user();
});
```

In `config/cors.php`

```php
return [
    'paths' => ['*'],
    'allowed_methods' => ['*'],
    'allowed_origins' => ['*'],
    'allowed_origins_patterns' => [],
    'allowed_headers' => ['*'],
    'exposed_headers' => [],
    'max_age' => 0,
    'supports_credentials' => true,
];
```

## Next.js

```
src/
└── pages
    ├── home.js
    └── index.js
```

##### index.js
```javascript
'use client';
import axios from "axios"
import { useEffect } from "react";

export default function Home() {
  axios.defaults.withCredentials = true;

  useEffect(() => {
    (async () => {
      await axios.get("http://localhost:8000/sanctum/csrf-cookie");

      await axios.post("http://localhost:8000/login", {
        email: "johndoe@example.org",
        password: "secret"
      });

      window.location = '/home';
    })();
  }, [])

  return (
    <></>
  )
}
```

##### home.js
```javascript
import axios from "axios"

export async function getServerSideProps({ req }) {
  axios.defaults.withCredentials = true;
  axios.defaults.headers.Cookie = req.headers.cookie;
  axios.defaults.headers.Origin = "http://localhost:3000"

  const { data } = await axios.get('http://localhost:8000/api/user');

  return { props: { data: data } }
}

export default function Home({ data }) {
  return (
    <pre>{JSON.stringify(data, null, 2)}</pre>
  )
}
```
