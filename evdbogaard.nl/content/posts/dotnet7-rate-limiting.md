---
title: ".NET 7 Rate Limiting"
date: 2022-11-01T10:00:00+02:00
draft: false
toc: false
images:
tags:
  - dotnet
  - ratelimiting
  - csharp
  - dotnet7
---

## What is rate limiting?
With rate limiting we determine how much a specific resource can be accessed. If you for example know a resource can handle x amount of requests per minute you can block any request that goes over that so your resource isn't overwhelmed by the requests and starts throwing errors.

## Rate limiting algorithms
With .NET 7 a few different algorithms are included for us to use. If these do not do what you want it's always possible to create your own. Inside the `System.Threading.RateLimiting` namespace there are the abstract classes `ReplenishingRateLimiter` and `RateLimiter` which can be used to create a custom implementation.

In the basis all RateLimiters have a `PermitLimit`, `QueueLimit`, and `QueueProcessingOrder`. Besides that depending on the specific RateLimiter there are other options you can set.

The `PermitLimit` holds the number of permits a given rate limiter can give. When there are no permits left new requests are either put in a queue or rejected.

The `QueueLimit` holds the number of requests that can be queued when there are no permits left. When a new permit becomes available it is first given to a request that is already waiting in the queue. Based on the `QueueProcessingOrder` it is either given to the oldest request in the queue or the newest request. If there are no permits and the queue is already full a request gets rejected immediately.

### ConcurrencyLimiter
The number of permits given to this limiter tell it how many requests it can handle at the same time. When a request is made a permit is given out and when the request is done this permit is returned to the limiter.

### FixedWindowRateLimiter
For this limiter a time window needs to be specified. Within that window it can handle the specified amount of requests. Unlike the `ConcurrencyLimiter` the permits are not given back when a request is done. Only after the time window is expired the full amount of permits is reset and new requests can be handled.

### SlidingWindowRateLimiter
Similar to the `FixedWindowRateLimiter`, but the time window is separated in segment. Each segment keeps track of how many permits are given out in that segment and they replenish separately instead of only after the entire time window has passed.
For example a time window 1 hour which is split into 3 segments of 20 minutes. In the first segments 10 permits are used. The seconds segment 5 permits are used and in the third segment 15 permits are used. In total this means 30 permits are used. After 20 minutes the first segment is dropped and those 10 permits are free to use again.

### TokenBucketRateLimiter
Different to the other limiters this one uses tokens instead of permits. Similar to the `FixedWindowRateLimiter` is that once a token is used it is not given back. The way it replenishes new tokens is by setting a replenishment period. Every time that period expires a predetermined amount of tokens is added back, never exceeding the maximum amount of tokens that is specified.

## How to use
There are two packages we need to use rate limiting. The first one is `System.Threading.RateLimiting`. This package holds most of the rate limiting knowledge and the four algorithms described above.
The second package is `Microsoft.AspNetCore.RateLimiting` which has everything we need to add rate limiting to the middleware.

Each rate limiter has an `AcquireAsync` and `AttemtAcquire` method which can be used for asynchronous or synchronous retrieval of a permit. When there are no permits available, but there is still room in the queue these methods also keep waiting till a permit becomes available.
You get a `RateLimitLease` back which holds information if the system was successful in getting a permit. It is still your own responsibility to check this and take action if no permits were available. The lease implements `IDisposable` so it's important to call Dispose() before you are done (or use using statement to take care of it for you), as some rate limiters need it to know a permit is given back and can be reused again.

Below you can find some sample code of how this would look:
```
private static readonly ConcurrencyLimiter _limiter = new(new ConcurrencyLimiterOptions()
    {
        PermitLimit = 2,
        QueueProcessingOrder = QueueProcessingOrder.OldestFirst,
        QueueLimit = 0
    });

public async Task Get()
    {
        using var lease = await _limiter.AcquireAsync(HttpContext);

        if (!lease.IsAcquired)
            throw new Exception("No permits available");

        // Continue with your code
        ...
    }
```

### Middleware
Having to type `AcquireAsync` and check the response for every method is a bit cumbersome, so lucky for us it's possible to add rate limiters through middleware.

In Program.cs you can call `app.UseRateLimiter()` which takes a `RateLimiterOptions` as parameter. There are three thing you can set here:
- `RejectionStatusCode`: The HTTP status code that is used when a request is rejected by the rate limiter. By default this is 503.
- `OnRejected`: A Func delegate that is called when a call is rejected. It has both the `HttpContext` and the `RateLimitLease` of the current request.
- `GlobalLimitter`: A specific limiter that is executed for every request that passed through the middleware. If a specific request has multiple limiters that are set up for it the GlobalLimitter is always called first.

Besides these options there are also some helper methods to easily create policies. For each rate limiting algorithm described above there is a specific Add helper method, like `AddConcurrencyLimiter`. Each helper method takes a string as first argument, which is the name used for the policy. And a second argument which are the options for that specific rate limiter.

There is also an `AddPolicy` method which can be used for custom created policies.

To use a policy you can use the `EnableRateLimitingAttribute`. It has similar behavior to for example role based authorization where you can put the attribute on a controller or method itself. The `DisableRateLimiting` attribute can be used if you want to exempt a method or controller from rate limiting.

```
[ApiController]
[Route("[controller]")]
[EnableRateLimiting("policyName")]
public class WeatherForecastController : ControllerBase
{ ... }
```

### PartitionedRateLimiter
The rate limiters we've seen so far cannot tell the difference between requests and handle everything the same. However in practice we probably want to have different behavior if someone is logged in or not. Or limit requests per ip address. For this the `PartitionedRateLimiter` is created. It basically creates a rate limiter per partition. You can pass in anything you want, but if you look at the `GlobalLimiter` we've seen earlier you see that it uses HttpContext. From that context we can then determine which value we want to use for the partition. To create on we can use a static method, an example of this can be seen below:

```
PartitionedRateLimiter.Create<HttpContext, string>(httpContext => RateLimitPartition.GetFixedWindowLimiter(httpContext.User.Identity.Name, options => new()
    {
        AutoReplenishment = true,
        PermitLimit = 100,
        QueueLimit = 0,
        QueueProcessingOrder = QueueProcessingOrder.OldestFirst,
        Window = TimeSpan.FromHours(1)
    }))
```

In this example we create a rate limiter that allows for 100 request per hour. This is linked to the `User.Identity.Name`, so two different users can have each 100 requests before their requests get rejected.
Now let's say that besides the 100 requests per hour we want to limit the total to 1000 per day. This can be done with the `CreateChained` helper method. You can chain multiple partitioned rate limiters together like this, and they don't even need to use the same partition. So you can for example rate limit on a user id and ip address at the same time.

```
PartitionedRateLimiter.CreateChained(
    PartitionedRateLimiter.Create<HttpContext, string>(httpContext =>
 RateLimitPartition.GetFixedWindowLimiter(httpContext.User.Identity.Name!, options => new()
    {
        AutoReplenishment = true,
        PermitLimit = 100,
        Window = TimeSpan.FromHours(1)
    })),
    PartitionedRateLimiter.Create<HttpContext, string>(httpContext =>
                RateLimitPartition.GetFixedWindowLimiter(httpContext.User.Identity?.Name!, options => new()
    {
        AutoReplenishment = true,
        PermitLimit = 1000,
        Window = TimeSpan.FromDays(1)
    })));
```

## Custom policies
In case you want to have a more advanced policy you can implement a custom one using the `IRateLimiterPolicy<T>` interface. There are two things you need to implement for this, `OnRejected`, which has similar behavior to the one used inside `UseRateLimiter`, and `GetPartition` which returns a specific `RateLimitPartition`. See below for a small example of where an authenticated user has no limiting, but otherwise there is a limit on ip address.

```
public class MyRateLimiterPolicy : IRateLimiterPolicy<string>
{
    public Func<OnRejectedContext, CancellationToken, ValueTask>? OnRejected { get; } = (context, cancellationToken) =>
    {
        context.HttpContext.Response.StatusCode = 429;
        return new();
    };

    public RateLimitPartition<string> GetPartition(HttpContext httpContext)
    {
        if (httpContext.User.Identity!.IsAuthenticated)
            return RateLimitPartition.GetNoLimiter(httpContext.User.Identity.Name!);

        return RateLimitPartition.GetFixedWindowLimiter(httpContext.Connection.RemoteIpAddress!.ToString(), options => new()
        {
            PermitLimit = 100,
            Window = TimeSpan.FromHours(1)
        });
    }
}
```

## Links
https://devblogs.microsoft.com/dotnet/announcing-rate-limiting-for-dotnet/