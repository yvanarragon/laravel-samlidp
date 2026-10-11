[![Latest Version on Packagist](https://img.shields.io/packagist/v/codegreencreative/laravel-samlidp.svg?style=flat-square)](https://packagist.org/packages/codegreencreative/laravel-samlidp)
[![Total Downloads](https://img.shields.io/packagist/dt/codegreencreative/laravel-samlidp.svg?style=flat-square)](https://packagist.org/packages/codegreencreative/laravel-samlidp)

[Buy me a coffee](https://www.buymeacoffee.com/upwebdesign) :coffee:

# Laravel SAML IdP 6.0.0

This package allows you to implement your own Identification Provider (idP) using the SAML 2.0 standard to be used with supporting SAML 2.0 Service Providers (SP).

- Starting in version ^5.1, Laravel 9 is supported.
- Starting in version ^5.2.4, Laravel 10 is supported.
- Starting in version ^5.2.9, Laravel 11 is supported.

Version 6.0.0 requires PHP ^8.4 and supports `litesaml/lightsaml` ^5.1 or ^6.0.
LightSAML 6 is recommended for the xmlseclibs 4 security hardening. LightSAML 4
and PHP 8.3 or earlier are no longer supported by this package version.
The Laravel dependency constraints remain 7–13; use a Laravel release and other
application dependencies that are compatible with PHP 8.4.

If you are looking for Laravel ^5.6 see [v1.0](https://github.com/codegreencreative/laravel-samlidp/tree/1.0)

If you are looking for Laravel ^6.0 use [v2.0](https://github.com/codegreencreative/laravel-samlidp/tree/2.0)

## Installation

```shell
composer require codegreencreative/laravel-samlidp:^6.0
```

## Upgrading from 5.4.0 to 6.0.0

This is a **breaking upgrade**. Back up your application's `composer.json`,
`composer.lock`, published `config/samlidp.php`, and IdP certificate/key before
upgrading. Test in staging with every configured service provider before deploying.

### 1. Upgrade the runtime and dependencies

Use PHP 8.4 or later in the PHP 8 series for Composer, web requests, CLI commands,
and queue workers. Keep the PHP extensions required by Laravel and SAML installed,
including DOM/XML, zlib, and OpenSSL (also used by the certificate command).
Update Laravel or other dependencies if they prevent installation on that runtime.

For the recommended LightSAML 6 / xmlseclibs 4 upgrade:

```shell
composer require codegreencreative/laravel-samlidp:^6.0 litesaml/lightsaml:^6.0 --with-all-dependencies
composer audit
```

Remove or update application-level pins on LightSAML 4 or xmlseclibs 3 first.
Composer will install `nyholm/psr7` to provide the PSR-17 response/stream factories
needed by the new bindings. Do not bypass platform requirements in production.

If another integration cannot yet use xmlseclibs 4, LightSAML ^5.1 is supported:

```shell
composer require codegreencreative/laravel-samlidp:^6.0 litesaml/lightsaml:^5.1 --with-all-dependencies
```

The minimum supported 5.x release is 5.1.0, not 5.0.0: 5.0.1 fixes XML signature
wrapping and 5.1.0 rejects duplicate/missing assertion IDs. The 5.x branch uses
xmlseclibs ^3.1.5 and does **not** include all the xmlseclibs 4 hardening in
LightSAML 6. Composer selects the latest compatible release; check your lock file.

### 2. Merge configuration and check cryptographic interoperability

- Change `digest_algorithm` in your **published** `config/samlidp.php` to
  `\RobRichards\XMLSecLibs\XMLSecurityDSig::SHA256`. The package default and fallback
  now use SHA-256, but an existing explicit SHA-1 setting is not overwritten.
  The signing key continues to use RSA-SHA256; this change concerns the reference
  digest, not the key or certificate.
- Encrypted assertions now use **RSA-OAEP** key transport on both supported
  LightSAML branches. Assertion data encryption remains AES-128-CBC.
  Every SP receiving encrypted assertions must support RSA-OAEP; update legacy
  SPs rather than relying on RSA-1.5. Existing certificates need not be regenerated
  solely for this upgrade.
- Preserve your SP destinations, certificates, guards, event listeners, and other
  application settings. Do not force-republish configuration over your custom file.
- Leave message signing and assertion encryption enabled where applicable.

### 3. Review custom LightSAML integrations

The package adapts its own SSO/SLO calls, so applications using only its public
events/routes do not need to implement PSR-7 conversion. Custom subclasses,
bindings, or code calling LightSAML directly must account for these upstream changes:

- `LightSaml\Model\Context\DeserializationContext` is now
  `LightSaml\Context\Model\DeserializationContext`; the serialization context moved
  similarly.
- HTTP bindings use PSR-7 requests/responses instead of Symfony HttpFoundation.
  Sending requires PSR-17 factories: a response factory for Redirect, and response
  plus stream factories for POST.
- Read POST HTML from the response body and redirect URLs from its `Location`
  header, rather than `getContent()` or `getTargetUrl()`.
- Review overrides and assertion listeners against the new typed method
  signatures, especially attribute values (strings or arrays of strings).
  Removed Pimple integrations and state `Serializable` implementations must be
  replaced if your application used them.
- With xmlseclibs 4, cryptographic keys use phpseclib rather than OpenSSL resources.
  Custom verification/decryption rejects DOCTYPEs, XPath transforms and RSA-1.5
  key transport by default, and enforces stricter signature/reference validation.
  Do not enable legacy modes globally to suppress migration errors.

These changes do not add inbound request-signature validation to this package.
Updating dependencies is not a substitute for reviewing any custom SAML
verification logic.

### 4. Validate and deploy

```shell
php artisan optimize:clear
```

Test SP-initiated login, signed responses/assertions, encrypted assertion decryption,
custom assertion attributes, RelayState round-trips, metadata, and single logout
(including signed HTTP-Redirect messages) with each SP. Check SHA-256 and RSA-OAEP
interoperability explicitly. Run your application's existing tests and
`composer audit`, deploy the tested lock file, rebuild any configuration caches,
and restart long-running workers using your normal deployment process.
No database migration is introduced by this upgrade.

Upstream references:

- [LightSAML issue #120](https://github.com/litesaml/lightsaml/issues/120)
- [LightSAML 5.0.0 release](https://github.com/litesaml/lightsaml/releases/tag/5.0.0)
- [LightSAML 6.0.0 release](https://github.com/litesaml/lightsaml/releases/tag/6.0.0)
- [LightSAML changelog, including 5.0.1 and 5.1.0 security fixes](https://github.com/litesaml/lightsaml/blob/6.0.0/CHANGELOG.md)
- [xmlseclibs 4 migration notes](https://github.com/robrichards/xmlseclibs/blob/4.0.0/README.md)

# Configuration

```shell
php artisan vendor:publish --tag="samlidp_config"
```

FileSystem configuration

```php
// config/filesystem.php

'disks' => [

        ...

        'samlidp' => [
            'driver' => 'local',
            'root' => storage_path() . '/samlidp',
        ]
],
```

Use the following command to create a self signed certificate for your IdP. If you change the certname or keyname to anything other than the default names, you will need to update your `config/samlidp.php` config file to reflect those new file names.

```shell
php artisan samlidp:cert [--days <days> --keyname <name> --certname <name>]
```

```shell
Options:
  --days=<days>      Days to add for the expiration date [default: 7800]
  --keyname=<name>   Name of the certificate key file [default: key.pem]
  --certname=<name>  Name of the certificate file [default: cert.pem]
```

Optionally, you can set the certificate and key using two environment variables: `SAMLIDP_CERT` and `SAMLIDP_KEY`.

## Usage

Within your login view, probably `resources/views/auth/login.blade.php` add the SAMLRequest directive beneath the CSRF directive:

```php
@csrf
@samlidp
```

The SAMLRequest directive will fill out the hidden input automatically when a SAMLRequest is sent by an HTTP request and therefore initiate a SAML authentication attempt. To initiate the SAML auth, the login and redirect processes need to be intervened. This is done using the Laravel events fired upon authentication.

## Config

After you publish the config file, you will need to set up your Service Providers. The key for the Service Provider is a base 64 encoded Consumer Service (ACS) URL. You can get this information from your Service Provider, but you will need to base 64 encode the URL and place it in your config. This is due to config dot notation.

You may use this command to help generate a new SAML Service Provider:

```shell
php artisan samlidp:sp
```

Example SP in `config/samlidp.php` file:

```php
<?php

return [
    // The URI to your login page
    'login_uri' => 'login',
    // The URI to the saml metadata file, this describes your idP
    'issuer_uri' => 'saml/metadata',
    // List of all Service Providers
    'sp' => [
        // Base64 encoded ACS URL
        'aHR0cHM6Ly9teWZhY2Vib29rd29ya3BsYWNlLmZhY2Vib29rLmNvbS93b3JrL3NhbWwucGhw' => [
            // ACS URL of the Service Provider
            'destination' => 'https://example.com/saml/acs',
            // Simple Logout URL of the Service Provider
            'logout' => 'https://example.com/saml/sls',
            // SP certificate
            // 'certificate' => '',
            // Turn off auto appending of the idp query param
            // 'query_params' => false,
            // Turn off the encryption of the assertion per SP
            // 'encrypt_assertion' => false
        ],
    ],
    // List of guards saml idp will catch Authenticated, Login and Logout events (thanks @abublihi)
    'guards' => ['web'],
];
```

#### Setting the service providers to be read from the database

Run migrations

```bash
php artisan migrate
```

Add a service provider to the database table `saml_service_providers`. The database table follows the same principles
as the config file.

```php
<?php

return [
    // ...
    'sp' => \CodeGreenCreative\SamlIdp\Models\SamlServiceProvider::class,
    // ...
];
```

### Setting the service provider certificate

There are three options to set the service provider certificate.

1. Provide the certificate as a string:

```php
<?php

return [
    // ...
    'sp' => [
        // Base64 encoded ACS URL
        'aHR0cHM6Ly9teWZhY2Vib29rd29ya3BsYWNlLmZhY2Vib29rLmNvbS93b3JrL3NhbWwucGhw' => [
            // ...
            // SP certificate
            // 'certificate' => "-----BEGIN CERTIFICATE-----\nb3BlbnNzaC1rZXktdjEA...LWdlbmVyYXRlZC1rZXkBAgM\n-----END CERTIFICATE-----"
        ],
    ],
    // ...
];
```

2. Load from a variable within the `.env` file.
   You can choose an appropriate variable name that best matches your projects requirements.

```php
<?php

return [
    // ...
    'sp' => [
        // Base64 encoded ACS URL
        'aHR0cHM6Ly9teWZhY2Vib29rd29ya3BsYWNlLmZhY2Vib29rLmNvbS93b3JrL3NhbWwucGhw' => [
            // ...
            // SP certificate
            // 'certificate' => env('SAML_SP_CERTIFICATE', '')
        ],
    ],
    // ...
];
```

3. Load the certificate from a file:

```php
<?php

return [
    // ...
    'sp' => [
        // Base64 encoded ACS URL
        'aHR0cHM6Ly9teWZhY2Vib29rd29ya3BsYWNlLmZhY2Vib29rLmNvbS93b3JrL3NhbWwucGhw' => [
            // ...
            // SP certificate
            // 'certificate' => 'file://' . storage_path('samlidp/service-provider.pem')
        ],
    ],
    // ...
];
```

## Log out of IdP after SLO

If you wish to log out of the IdP after SLO has completed, set `LOGOUT_AFTER_SLO` to `true` in your `.env` perform the logout action on the Idp.

```
// .env

LOGOUT_AFTER_SLO=true
```

## Redirect to SLO initiator after logout

If you wish to return the user back to the SP by which SLO was initiated, you may provide an additional query parameter to the `/saml/logout` route, for example:

```
https://idp.com/saml/logout?return_to=mysp.com
```

After all SP's have been logged out of, the user will be redirected to `mysp.com`. For this to work properly you need to add the `sp_slo_redirects` option to your `config/samlidp.php` config file, for example:

```php
<?php

// config/samlidp.php

return [
    // If you need to redirect after SLO depending on SLO initiator
    // key is beginning of HTTP_REFERER value from SERVER, value is redirect path
    'sp_slo_redirects' => [
        'mysp.com' => 'https://mysp.com',
    ],
];
```

## Attributes (optional)

Service providers may require more additional attributes to be sent via assertion. Its even possible that they require the same information but as a different Claim Type.

By Default this package will send the following Claim Types:

`ClaimTypes::EMAIL_ADDRESS` as `auth()->user()->email`
`ClaimTypes::GIVEN_NAME` as `auth()->user()->name`

This is because Laravel migrations, by default, only supply email and name fields that are usable by SAML 2.0.

To add additional Claim Types, you can subscribe to the Assertion event:

`CodeGreenCreative\SamlIdp\Events\Assertion`

Subscribing to the Event:

In your `App\Providers\EventServiceProvider` class, add to the already existing `$listen` property...

```php
protected $listen = [
    'App\Events\Event' => [
        'App\Listeners\EventListener',
    ],
    'CodeGreenCreative\SamlIdp\Events\Assertion' => [
        'App\Listeners\SamlAssertionAttributes'
    ]
];
```

Sample Listener:

```php
<?php

namespace App\Listeners;

use LightSaml\ClaimTypes;
use LightSaml\Model\Assertion\Attribute;
use CodeGreenCreative\SamlIdp\Events\Assertion;

class SamlAssertionAttributes
{
    public function handle(Assertion $event)
    {
        $event->attribute_statement
            ->addAttribute(new Attribute(ClaimTypes::PPID, auth()->user()->id))
            ->addAttribute(new Attribute(ClaimTypes::NAME, auth()->user()->name));
    }
}
```

## Digest Algorithm (optional)

See `\RobRichards\XMLSecLibs\XMLSecurityDSig` for all digest options.

```php
<?php

return [
    // Define what digest algorithm you want to use
    'digest_algorithm' => \RobRichards\XMLSecLibs\XMLSecurityDSig::SHA256,
];
```

[Buy me a coffee](https://www.buymeacoffee.com/upwebdesign) :coffee:
