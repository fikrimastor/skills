---
name: laravel-teams-exception-notifications
description: "Use when adding Microsoft Teams exception/error notifications to a Laravel app, or when asked to port Lodestar's Teams alerting pattern to another Laravel project. Covers non-blocking Adaptive Card alerts for HTTP, CLI, and queue-job failures, with recursion guarding, deduplication, and sensitive-data scrubbing. Trigger on \"Teams notification\", \"Teams webhook\", \"notify Teams on error\", \"Teams alert for exceptions\", \"error reporting to Teams\", or any Laravel + Microsoft Teams exception-reporting request."
---

# Laravel → Microsoft Teams Exception Notifications

Battle-tested pattern (source: Lodestar, `app/Modules/Central/Support/Observability/*`, `app/Notifications/Teams*`) for sending a non-blocking Adaptive Card to a Teams channel whenever an exception is reported — HTTP request, CLI command, or failed queue job — without risking notification storms, blocking the request, or leaking secrets.

## When to use this

User wants: "send Teams alert on error", "notify Teams when a job fails", "Teams webhook for exceptions", or is porting this exact capability from another Laravel project.

## Package

```bash
composer require laravel-notification-channels/microsoft-teams
```

Verify the installed major version in `composer.json`/`composer.lock` before writing code — API surface (`MicrosoftTeamsAdaptiveCard`, `ContentBlocks\*`, `Actions\*`) can shift between majors. Don't trust old design docs; trust the lockfile.

## Config

`config/services.php`:

```php
'microsoft_teams' => [
    'webhook_url' => env('TEAMS_ERROR_WEBHOOK_URL'),
],
```

Blank/missing webhook URL must be a silent no-op (dev/local envs won't have one) — never throw for it.

## Architecture (5 pieces, no custom queue job, no service provider)

```
HTTP/CLI exception ──► bootstrap/app.php withExceptions()->report() ─┐
                                                                       ├──► TeamsErrorReporter::reportSafely()
Queue JobFailed event ─► auto-discovered listener ────────────────────┘         │
                                                                                  ▼
                                                              guard + dedup + build DTO
                                                                                  │
                                                                                  ▼
                                              Notification::route(SafeChannel::class, null)
                                                            ->notify(TeamsExceptionNotification)
                                                                                  │
                                              (ShouldQueue ⇒ Laravel's own SendQueuedNotifications)
                                                                                  ▼
                                                      SafeMicrosoftTeamsChannel (decorator)
                                                    activates guard → delegates to vendor
                                                    MicrosoftTeamsChannel → catches ALL
                                                    failures, logs (scrubbed), never rethrows
                                                    → deactivates guard in finally
```

### 1. Wiring — `bootstrap/app.php` (no AppServiceProvider registration needed)

```php
->withEvents(discover: [
    __DIR__.'/../app/Modules/*/Infrastructure/Listeners', // wherever your listeners live
])
->withExceptions(function (Exceptions $exceptions): void {
    $exceptions->report(function (Throwable $e): void {
        TeamsErrorReporter::reportSafely($e);
    });
})
```

Hooking `withExceptions()->report()` (not a Monolog handler, not middleware) is deliberate: it is the one place that sees HTTP, CLI, *and* queue-job exceptions uniformly.

Queue-job failures don't flow through `report()` — wire a listener:

```php
final class ReportQueueFailureToTeams
{
    public function handle(JobFailed $event): void
    {
        TeamsErrorReporter::reportSafely($event->exception);
    }
}
```
Place it wherever `withEvents(discover: [...])` scans; Laravel auto-registers it — zero manual binding.

### 2. Reporter — guard, config check, dedup, dispatch

```php
final class TeamsErrorReporter
{
    private const DEDUP_TTL_MINUTES = 5;

    public static function report(Throwable $e): void
    {
        if (TeamsErrorContext::isActive()) {
            return; // a Teams notification attempt caused this — do not recurse
        }

        $webhookUrl = config('services.microsoft_teams.webhook_url');
        if (blank($webhookUrl)) {
            return; // no webhook configured for this env
        }

        // Atomic dedup — Cache::add returns false if the key already exists.
        $dedupKey = 'teams-error:'.app()->environment().':'.$e::class
            .':'.md5(substr($e->getTraceAsString(), 0, 1024));
        if (! Cache::add($dedupKey, true, now()->addMinutes(self::DEDUP_TTL_MINUTES))) {
            return; // already notified for this error within the TTL
        }

        Notification::route(SafeMicrosoftTeamsChannel::class, null)
            ->notify(new TeamsExceptionNotification(ExceptionReportPayload::fromThrowable($e), $webhookUrl));
    }

    // Entry point from bootstrap/app.php — any failure inside report() (e.g. Cache::add
    // throwing) must never mask the *original* exception being reported.
    public static function reportSafely(Throwable $e): void
    {
        try {
            self::report($e);
        } catch (Throwable $te) {
            Log::error('Teams error reporter failed', ['error' => $te->getMessage()]);
        }
    }
}
```

### 3. Recursion guard — plain static flag, two touch points

```php
final class TeamsErrorContext
{
    private static bool $active = false;
    public static function isActive(): bool { return self::$active; }
    public static function activate(): void { self::$active = true; }
    public static function deactivate(): void { self::$active = false; }
}
```

Why this exists: without it, a failure *sending* the Teams notification could itself get reported to Teams → infinite loop. Belt-and-suspenders: the channel wrapper (below) also ensures the queued job never actually throws, so `JobFailed` never fires for the notification's own job in the first place. Keep both layers — cheap insurance.

### 4. Sanitized DTO — never queue the raw Throwable

```php
final readonly class ExceptionReportPayload
{
    public function __construct(
        public string $class, public string $message, public string $file, public int $line,
        public string $traceHash, public string $trace, public string $fullTrace,
        public string $environment, public ?string $url, public ?string $userId,
        public ?string $requestId, public float $timestamp,
    ) {}

    public static function fromThrowable(Throwable $e): self
    {
        $traceString = $e->getTraceAsString();
        $trimmedTrace = substr(implode("\n", array_slice(explode("\n", $traceString), 0, 10)), 0, 2048);
        $fullTrace = substr(implode("\n", array_slice(explode("\n", $traceString), 0, 40)), 0, 8192);
        $url = request()?->url();

        return new self(
            class: $e::class,
            message: SensitiveDataScrubber::scrubString($e->getMessage()),
            file: $e->getFile(), line: $e->getLine(),
            traceHash: md5(substr($traceString, 0, 1024)),
            trace: SensitiveDataScrubber::scrubString($trimmedTrace),
            fullTrace: SensitiveDataScrubber::scrubString($fullTrace),
            environment: (string) app()->environment(),
            url: $url !== null ? SensitiveDataScrubber::scrubString($url) : null,
            userId: ($id = auth()->id()) !== null ? (string) $id : null,
            requestId: request()?->ip(),
            timestamp: microtime(true),
        );
    }
}
```

Why a DTO and not the Throwable: exceptions aren't reliably serializable across the queue boundary, and this is the one place to enforce trace-size caps + scrubbing before anything leaves the process.

`SensitiveDataScrubber` (static utility, reusable outside Teams too): `scrub(array)` redacts by key (`EXACT_MATCH = ['name','mail','sig']`, `SUBSTRING_MATCH = ['authorization','cookie','email','nric','passport','payroll','salary','secret','signature','token','wage']`); `scrubString(string)` regex-redacts emails, NRIC-like digit groups, and `key=value`/`key: value` tokens matching the same key lists. Adapt the key lists to your domain's PII (this app is Malaysian HR/payroll — NRIC, payroll, salary are domain-specific).

### 5. Notification + channel decorator — async via ShouldQueue, not a custom job

```php
class TeamsExceptionNotification extends Notification implements ShouldQueue
{
    use Queueable;

    public function __construct(
        private readonly ExceptionReportPayload $payload,
        private readonly string $webhookUrl,
    ) {}

    public function payload(): ExceptionReportPayload { return $this->payload; }

    /** @return list<class-string> */
    public function via(mixed $notifiable): array
    {
        return [SafeMicrosoftTeamsChannel::class]; // FQCN channel — see note below
    }

    public function toMicrosoftTeams(mixed $notifiable): MicrosoftTeamsAdaptiveCard
    {
        return MicrosoftTeamsAdaptiveCard::create()
            ->to($this->webhookUrl)
            ->title("[{$this->payload->environment}] ".class_basename($this->payload->class))
            ->content([
                TextBlock::create()->setText($this->payload->message)->setWrap(true)->setFontType('Monospace'),
                FactSet::create()->setSeparator(true)->setFacts([
                    Fact::create()->setTitle('Environment')->setValue($this->payload->environment),
                    Fact::create()->setTitle('File')->setValue("{$this->payload->file}:{$this->payload->line}"),
                    // ...
                ]),
            ])
            ->actions([
                ActionOpenUrl::create()->setMode('Primary')->setTitle('View Full Trace')->setUrl($this->logViewerUrl()),
            ]);
    }
}
```

```php
final class SafeMicrosoftTeamsChannel
{
    public function __construct(private readonly MicrosoftTeamsChannel $inner) {}

    public function send(mixed $notifiable, Notification $notification): void
    {
        TeamsErrorContext::activate();
        try {
            $this->inner->send($notifiable, $notification);
        } catch (Throwable $e) {
            Log::error('Teams notification failed', [
                // Vendor exception messages embed the webhook URL (a credential) — scrub it.
                'error' => SensitiveDataScrubber::scrubString($e->getMessage()),
            ]);
        } finally {
            TeamsErrorContext::deactivate();
        }
    }
}
```

**Key lesson (learned the hard way — see `docs/superpowers/plans/2026-07-06-teams-notification-channel-wrapper.md`):** do NOT write a custom `ShouldQueue` job that calls `Notification::send()`. The `Notification` class already implements `ShouldQueue` and Laravel's built-in `SendQueuedNotifications` job handles async dispatch for free. A custom job duplicates that mechanism. Instead, put your two pieces of *custom* behaviour — recursion guard, log-only failure handling — into a thin channel **decorator**, and let `via()` return the decorator's FQCN.

**Key lesson (Laravel 12 specific):** returning a **class name** (not a string driver key) from `via()` needs no `Notification::extend()` and no service-provider registration. `Illuminate\Notifications\ChannelManager` has a container-resolution fallback for FQCN channels — it works as long as the class is concrete with a single typed constructor parameter (here, the vendor's `MicrosoftTeamsChannel`, which the container already knows how to build). Confirm this behavior against the installed Laravel version if porting to something other than v12.

### 6. Optional: custom Adaptive Card elements the vendor package doesn't expose

If you need Adaptive Card features the notification-channel wrapper package hasn't wrapped (e.g. `Action.ToggleVisibility`, or a `TextBlock` with `id`/`isVisible` for expand/collapse), duck-type a class with a `toArray(): array` method matching the schema — the package's card builder accepts anything with `toArray()`. See `app/Notifications/Teams/{ActionToggleVisibility,IdentifiableTextBlock}.php` for the pattern (extend the vendor block/action class where possible, add the missing fields, override `toArray()` to merge `parent::toArray()` with the new keys).

## Testing (Pest)

```php
beforeEach(function (): void {
    Cache::flush();
    config()->set('services.microsoft_teams.webhook_url', 'https://example.webhook.office.com/x');
});
afterEach(fn () => TeamsErrorContext::deactivate()); // guard must never leak between tests

it('dispatches the teams notification for a reported exception', function (): void {
    Notification::fake();
    TeamsErrorReporter::report(new RuntimeException('boom'));
    Notification::assertSentOnDemand(TeamsExceptionNotification::class);
});

it('does not dispatch when no webhook url is configured', function (): void {
    Notification::fake();
    config()->set('services.microsoft_teams.webhook_url', null);
    TeamsErrorReporter::report(new RuntimeException('boom'));
    Notification::assertNothingSent();
});

it('deduplicates identical errors within the ttl window', function (): void {
    Notification::fake();
    $e = new RuntimeException('boom');
    TeamsErrorReporter::report($e);
    TeamsErrorReporter::report($e);
    Notification::assertSentOnDemandTimes(TeamsExceptionNotification::class, 1);
});

it('swallows vendor exceptions and logs a scrubbed error, never rethrowing', function (): void {
    Notification::shouldReceive('route')->andThrow(new RuntimeException('teams down'));
    Log::shouldReceive('error')->once()->withArgs(fn (string $m) => $m === 'Teams notification failed');
    (new SafeMicrosoftTeamsChannel(app(MicrosoftTeamsChannel::class)))->send(null, $notification);
    expect(TeamsErrorContext::isActive())->toBeFalse(); // guard reset even on failure
});

it('is wired into the exception handler', function (): void {
    Notification::fake();
    Route::get('/_test/throws', fn () => throw new RuntimeException('wired boom'));
    $this->get('/_test/throws')->assertStatus(500);
    Notification::assertSentOnDemand(TeamsExceptionNotification::class);
});
```

Use `Notification::assertSentOnDemand*` (not `assertSentTo`) because dispatch goes through `Notification::route(...)`, not a notifiable model. Use `Log::shouldReceive()->once()->with(...)` to pin the exact scrubbed message when testing failure paths.

## Porting checklist for a new Laravel project

1. `composer require laravel-notification-channels/microsoft-teams` — check the version's API before copying card-building code verbatim.
2. Add `services.microsoft_teams.webhook_url` + `TEAMS_ERROR_WEBHOOK_URL` env var.
3. Port `TeamsErrorContext` (static guard) and `SensitiveDataScrubber` (adapt PII key lists to the domain) as standalone, dependency-free classes.
4. Port `ExceptionReportPayload` DTO — adjust fields to taste, keep the trace-size caps and scrub-before-store discipline.
5. Write `SafeXChannel` decorator around the vendor channel: activate guard → delegate → catch-log-never-rethrow → deactivate in `finally`. Scrub the caught exception's message (webhook URL is often embedded in vendor errors).
6. Write the `Notification` class: `implements ShouldQueue`, `via()` returns `[SafeXChannel::class]` — do **not** write a custom dispatch job.
7. Write `XErrorReporter::report()` (guard check → config blank check → atomic `Cache::add` dedup → `Notification::route(SafeXChannel::class, null)->notify(...)`) and `reportSafely()` (try/catch/log around `report()`).
8. Wire exactly two call sites: `bootstrap/app.php` `withExceptions()->report()` for HTTP/CLI, and a `JobFailed` listener (auto-discovered via `withEvents(discover:)` or manually registered) for queue failures. No service provider needed.
9. Write the Pest tests above before considering it done — the guard-reset-on-failure and dedup-TTL tests are the ones that catch regressions when someone "simplifies" this later.
