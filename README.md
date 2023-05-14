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

export default function Index() {
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
  axios.defaults.headers.Origin = req.headers.host;

  const { data } = await axios.get('http://localhost:8000/api/user');

  return { props: { data: data } }
}

export default function Home({ data }) {
  return (
    <pre>{JSON.stringify(data, null, 2)}</pre>
  )
}
```

# Cookie Authentication
## Laravel
```php
class EncryptCookies extends Middleware
{
    /**
     * The names of the cookies that should not be encrypted.
     *
     * @var array<int, string>
     */
    protected $except = [
        'laravel_user',
    ];
}
```

```php
$cookie = cookie('laravel_user', Auth::user(), $config['lifetime'], $config['path'], $config['domain'], $config['secure'], false, false, $config['same_site'] ?? null);

return response()->noContent()->withCookie($cookie);
```

## Next.js
```javascript
import qs from 'querystring';
import Cookies from 'js-cookie';

export function getAuthenticatedUser({ req }) {
  try {
    const laravel_user = typeof window === 'undefined'
      ? qs.decode(req?.headers.cookie ?? '', '; ')?.laravel_user
      : Cookies.get('laravel_user');
    const auth = JSON.parse(laravel_user);

    return auth;
  } catch (error) {
    // TODO
  }
}
```

```javascript
MyApp.getInitialProps = async (context: AppContext) => {
  const initialProps = App.getInitialProps(context);

  const auth = getAuthenticatedUser({ req: context.ctx.req });

  return { ...initialProps, auth };
};

function MyApp({ Component, pageProps, auth }: AppProps) {
  if (!auth) {
    return <Component {...pageProps} />;
  }

  return (
    <>
      <Navbar auth={auth} />
      <Component {...pageProps} />
    </>
  );
}

export default AdmissionGuideApp;
```
