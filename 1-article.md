A few weeks ago I saved a link about keeping test emails away from real users. The plan was simple: add `Mail::alwaysTo()` to our staging setup and stop worrying. The Laravel docs list it for local development, and Laravel News plus at least five other blog posts I found carry it over to staging. None of them mention a limit.

Before shipping it, I opened the framework source to see what that call actually does. Turns out, it guards only one of your mailers.

## The Safety Net Everyone Copies

Here is the usual version, adapted from the docs and wrapped in an environment check:

```php
use Illuminate\Support\Facades\Mail;

public function boot(): void
{
    if (! $this->app->isProduction()) {
        Mail::alwaysTo('staging@example.com');
    }
}
```

Every message sent through it gets its `to` replaced, and `cc` and `bcc` are removed. Looks complete.

## The Problem

Now add a second mailer. Say transactional mail goes through the default one and invoices through another:

```php
// config/mail.php
'default' => env('MAIL_MAILER', 'smtp'),

'mailers' => [
    'smtp' => [/* ... */],
    'billing' => ['transport' => 'postmark'],
],
```

I tested this in a sandbox with Laravel 12.43, both mailers on the `array` transport, and `Mail::alwaysTo('staging@example.com')` in place:

```php
Mail::to('client@example.com')->send(new InvoiceMail());
// sent to: staging@example.com

Mail::mailer('billing')->to('client@example.com')->send(new InvoiceMail());
// sent to: client@example.com
```

Notifications have the same hole. `MailMessage` lets you pick a mailer too:

```php
public function toMail(object $notifiable): MailMessage
{
    return (new MailMessage)
        ->mailer('billing')
        ->line('Your invoice is ready.');
}
// sent to: the real customer
```

Even with scrubbed staging data, real addresses slip through: admins, seeded clients, imported leads. Each one gets the invoice.

Pointing `MAIL_MAILER` at Mailpit or `log` on staging doesn't close this either. It swaps the default mailer only, so if staging has the real Postmark key, `Mail::mailer('billing')` still uses it.

## Why It Happens

`Mail::alwaysTo()` is not a global setting. The facade points to `MailManager`, and `MailManager` forwards unknown calls to the **default** mailer only:

```php
// Illuminate\Mail\MailManager
public function __call($method, $parameters)
{
    return $this->mailer()->$method(...$parameters);
}
```

So `alwaysTo` is stored on one `Mailer` instance. When you ask for `Mail::mailer('billing')`, the manager builds a separate instance in `resolve()`, and that instance gets its global addresses from config:

```php
foreach (['from', 'reply_to', 'to', 'return_path'] as $type) {
    $this->setGlobalAddress($mailer, $config, $type);
}
```

`setGlobalAddress()` reads `mail.mailers.billing.to` first, then falls back to `mail.to`. Your `alwaysTo` call is never part of that.

This isn't new. [Issue #44024](https://github.com/laravel/framework/issues/44024) described the same thing in 2022 and was closed as a support question. The logic is the same in Laravel 12 and on the 13.x branch.

There is a second way it fails, under Octane. Service providers boot once, but Octane forgets the resolved mailers between requests, so the redirect set in `boot()` is lost even in a single-mailer app. I didn't run Octane for this one. A Laravel maintainer confirmed it in [discussion #53727](https://github.com/laravel/framework/discussions/53727).

## The Fix

Put the redirect where every configured mailer reads it: the config. `config/mail.php` has no `to` key by default, but `MailManager` picks one up if you add it:

```php
// config/mail.php
'to' => [
    'address' => env('MAIL_TO_ADDRESS'),
    'name' => env('MAIL_TO_NAME', 'Staging Inbox'),
],
```

```bash
# .env on staging only
MAIL_TO_ADDRESS=staging@example.com
```

In the same sandbox, both mailers now sent to `staging@example.com`. The address is applied each time a mailer is built, so it also survives `Mail::forgetMailers()`, which is what Octane does between requests. I checked that part with a plain call, not a running Octane server.

When `MAIL_TO_ADDRESS` is not set, nothing changes, so production is unaffected without any `if` in a provider. Leave the key out of the production `.env` entirely. An empty `MAIL_TO_ADDRESS=` is still a value, and every send then fails with `RfcComplianceException`.

One trade-off: this fails open. Forget the variable on a new staging box and mail goes out for real. I'd add a guard for that:

```php
public function boot(): void
{
    if ($this->app->environment('staging') && blank(config('mail.to.address'))) {
        throw new RuntimeException('MAIL_TO_ADDRESS must be set on staging.');
    }
}
```

Heads up: two exceptions. A mailer with its own `to` key in `mail.mailers` wins over the global one, even `'to' => null`. And a mailer built on the fly with `Mail::build([...])` skips global addresses completely.

## A Test That Keeps It Fixed

The per-mailer `to` is the one that bites later. Someone adds a `to` to one mailer months later, never having read this config. So I'd rather have a test that checks every mailer against the real config. First, give the test environment the address:

```xml
<!-- phpunit.xml -->
<env name="MAIL_TO_ADDRESS" value="staging@example.com"/>
```

```php
use Illuminate\Support\Facades\Mail;
use Tests\TestCase;

final class MailRedirectTest extends TestCase
{
    public function test_every_mailer_sends_to_the_staging_inbox(): void
    {
        // Arrange
        $mailers = array_keys(config('mail.mailers'));

        foreach ($mailers as $name) {
            config(["mail.mailers.{$name}.transport" => 'array']);
        }

        Mail::forgetMailers();

        foreach ($mailers as $name) {
            // Act
            Mail::mailer($name)->raw('ping', fn ($message) => $message->to('client@example.com'));

            // Assert
            $sent = Mail::mailer($name)->getSymfonyTransport()->messages()->last()->getOriginalMessage();

            $this->assertSame('staging@example.com', $sent->getTo()[0]->getAddress(), "Mailer [{$name}] escaped the redirect");
        }
    }
}
```

It swaps every transport to `array`, so nothing is actually sent. `Mail::forgetMailers()` matters here: a mailer resolved before the test keeps its old transport, and with a `log` default the assertion crashes on `LogTransport`, which has no `messages()`.

With the nine mailers from the default config (smtp, ses, postmark, resend, sendmail, log, array, failover, roundrobin) it passes. Give `billing` its own `to` and it fails with `Mailer [billing] escaped the redirect`. Delete the `to` block from `config/mail.php` and it fails on `smtp`. Don't use `Mail::fake()` here: the fake replaces the manager, so it can't tell you anything about redirects.

## Final Thoughts

`Mail::alwaysTo()` is fine on your laptop, with one mailer and no Octane. For staging, where real addresses live in the database, I'd use `mail.to` in config, a boot guard for the missing variable, and a test that walks every mailer.

---

- [Laravel docs: Mail and Local Development](https://laravel.com/docs/12.x/mail#mail-and-local-development)
- [MailManager source on the 13.x branch](https://github.com/laravel/framework/blob/13.x/src/Illuminate/Mail/MailManager.php)
- [laravel/framework#44024: Mail::alwaysTo() is not overriding mailable](https://github.com/laravel/framework/issues/44024)
- [laravel/framework#53727: Mail::alwaysTo not working for synced mails (Octane)](https://github.com/laravel/framework/discussions/53727)
- [Redirecting Test Emails Safely in Laravel, by Paul Conroy](https://www.conroyp.com/articles/redirecting-test-emails-safely-in-laravel)

## Author's Note

Thanks for sticking around!
Find me on [dev.to](https://dev.to/tegos), [linkedin](https://www.linkedin.com/in/ivan-mykhavko/), or you can check out my work on [github](https://github.com/tegos).

**Laravel, after the happy path.**
