# نماذج برمجية مقترحة لتطوير Laravel Socialite

## 1. نموذج لدعم OpenID Connect (OIDC)

لإضافة دعم OIDC بشكل أصلي، يمكن إنشاء فئة جديدة مثل `OidcProvider` ترث من `\Laravel\Socialite\Two\AbstractProvider` وتضيف منطق التحقق من رمز الهوية (ID Token) واستخراج المطالبات (Claims).

```php
<?php

namespace Laravel\Socialite\Two;

use Firebase\JWT\JWT;
use Firebase\JWT\Key;
use Illuminate\Support\Arr;

class OidcProvider extends AbstractProvider
{
    /**
     * Get the ID Token from the response and validate it.
     *
     * @param  string  $token
     * @return array
     */
    protected function validateAndDecodeIdToken($token)
    {
        // 1. Fetch the public keys (JWKS) from the provider's discovery endpoint.
        // This is a complex step that requires an HTTP call and caching.
        // For simplicity, we assume a hardcoded key or a fetched key set.
        
        // Example: Fetching JWKS and selecting the correct key.
        // $jwks = $this->getJwks();
        // $key = $this->findKeyInJwks($jwks, $token);

        // For prototype, use a placeholder key (In a real scenario, this is dynamic)
        $key = new Key('YOUR_PUBLIC_KEY_OR_JWKS_KEY', 'RS256');

        // 2. Decode and validate the token.
        $decoded = (array) JWT::decode($token, $key);

        // 3. Perform standard OIDC validation checks (iss, aud, exp, nonce).
        if (Arr::get($decoded, 'iss') !== $this->getConfig('issuer')) {
            throw new InvalidArgumentException('Invalid issuer in ID Token.');
        }
        
        // ... other validation checks (audience, expiration, nonce)

        return $decoded;
    }

    /**
     * Map the raw user array to a Socialite User instance.
     *
     * @param  array  $user
     * @return \Laravel\Socialite\Two\User
     */
    protected function mapUserToObject(array $user)
    {
        $idToken = Arr::get($user, 'id_token');
        $claims = $this->validateAndDecodeIdToken($idToken);

        return (new User)->setRaw($user)->map([
            'id' => Arr::get($claims, 'sub'),
            'nickname' => Arr::get($claims, 'preferred_username'),
            'name' => Arr::get($claims, 'name'),
            'email' => Arr::get($claims, 'email'),
            'avatar' => Arr::get($claims, 'picture'),
        ])->setIdToken($idToken); // New method to set ID Token
    }
    
    /**
     * Get the authentication URL for the provider.
     *
     * @param  string  $state
     * @return string
     */
    protected function getAuthUrl($state)
    {
        // Ensure 'openid' scope is always present for OIDC
        $this->scopes = array_unique(array_merge($this->scopes, ['openid']));
        
        return $this->buildAuthUrlFromBase($this->getBaseAuthUrl(), $state);
    }
}
```

## 2. نموذج لتحسين أدوات الاختبار (Testing Fakes)

لتحسين مرونة `Socialite::fake()`، يمكن تعديل طريقة `fake()` في `SocialiteManager` للسماح بتمرير بيانات المستخدم المخصصة، وتعديل `FakeProvider` لاستخدام هذه البيانات.

**التعديل المقترح في `SocialiteManager`:**

```php
// في \Laravel\Socialite\SocialiteManager

/**
 * Set the fake provider to be used for testing.
 *
 * @param  array|callable|null  $user
 * @return void
 */
public function fake($user = null)
{
    $this->app->singleton(Factory::class, function ($app) use ($user) {
        return new FakeSocialiteManager($app, $user);
    });
}
```

**فئة `FakeSocialiteManager` المقترحة:**

```php
// فئة جديدة تستخدم في بيئة الاختبار

namespace Laravel\Socialite;

use Laravel\Socialite\Contracts\Factory;
use Laravel\Socialite\Contracts\Provider;
use Laravel\Socialite\Two\User;

class FakeSocialiteManager implements Factory
{
    protected $app;
    protected $user;

    public function __construct($app, $user = null)
    {
        $this->app = $app;
        $this->user = $user;
    }

    public function driver($driver = null): Provider
    {
        // إذا تم تمرير دالة، قم بتنفيذها للحصول على بيانات المستخدم
        $userData = is_callable($this->user) ? call_user_func($this->user, $driver) : $this->user;

        // بيانات افتراضية إذا لم يتم تمرير أي شيء
        $defaultUser = [
            'id' => 12345,
            'nickname' => 'testuser',
            'name' => 'Test User',
            'email' => 'test@example.com',
            'avatar' => 'https://example.com/avatar.jpg',
        ];

        $user = (new User)->map(array_merge($defaultUser, (array) $userData));

        return new FakeProvider($user);
    }
}
```

**استخدام النموذج في الاختبارات:**

```php
// في ملف الاختبار

// 1. تزييف (Fake) مع بيانات مخصصة
Socialite::fake([
    'github' => [
        'id' => 98765,
        'name' => 'GitHub Test',
        'email' => 'github@test.com',
    ],
    'google' => [
        'id' => 112233,
        'name' => 'Google Test',
        'email' => 'google@test.com',
    ],
]);

// 2. تزييف مع دالة (Callback)
Socialite::fake(function ($driver) {
    return [
        'id' => rand(1000, 9999),
        'name' => 'User from ' . $driver,
    ];
});
```
