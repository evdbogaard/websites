---
title: "Health checks in ASP.Net Core web API"
date: 2021-09-14T10:00:00+02:00
draft: false
toc: false
images:
tags:
  - aspnetcore
  - healthchecks
  - azure
---

When you have an API running in the cloud it is important to know how healthy it is and if it might experience issues by itself or other services it relies on.
To help you with that health checks were added tot ASP.Net Core to allow near-real-time monitoring of information about the state of your system. With only a few lines of code you can enable it in your own API.
In your `ConfigureServices` method add the following line `services.AddHealthChecks` and in the `Configure` method under the `endpoints.MapControllers();` line add `endpoints.MapHealthChecks("/hc");`. Congratulations you now have created and endpoint that shows the status of your API. If you run your code and go to that url you can see it reports `Healthy`. As you haven't added any checks it doesn't really have any value yet, but the basis is set. Now let's look on how to add some checks to it.

## Custom health checks
You can add multiple health checks that need to be checked. Fortunately for us there is already a nice Github repository available with all kinds of health checks in it. You can check it out [here](https://github.com/Xabaril/AspNetCore.Diagnostics.HealthChecks). There are nuget packages for different kinds of services like db, service bus, storage, keyvault, etc.
Although it's nice these packages are already available for us, there will be cases where you need to write your own custom health check. For this they created the `IHealthCheck` interface.
You have to create a new class that implements that interface and you have created your own custom health check. Add `AddCheck<MyCustomHealthCheck>()` after the `AddHealthChecks` call, it's fluent to your `ConfigureServices` method and the health checks are automatically executed when the end point is called.

## Demo project
To make the above text a little more visual lets show it all through a small demo project. We're going to create a new dotnet web api project and add a check to CosmosDb and also create a custom check through which we can fake an unhealthy app.
So first lets create a new project. I prefer command line to do this, but there are of course other possible options.

- In the command line type: `dotnet new webapi -o Demo.HealthCheck.Api`
- Open the newly created project in your favorite editor
- In `Startup.cs` find `ConfigureServices` and at the end of the method add `services.AddHealthChecks();`.
- In the same file find the line `endpoints.MapControllers();` and under it add `endpoints.MapHealthChecks("/hc");`
- Run the code and verify that the endpoint `/hc` exists and returns `Healthy`

With this we have the basic setup working. It's now time to add some actual checks. For demo purposes I created a CosmosDb account in the Azure portal. As this is not a blog post on CosmosDb I assume you created these yourself or simply omit this checks from your own project. The idea behind it is valid for other services like Sql Server, MySql, Postgres, etc. You have to supply a connection string and it tries to connect to it. If it succeeds everything is good, else `Unhealthy` is returned.

There are nuget package available for different services. For CosmosDb we need to type the following in the command line: `dotnet add package AspNetCore.HealthChecks.CosmosDb`.
Now that we have the package installed we can alter our `AddHealthChecks()` line to include a check for CosmosDb. You can now add `AddCosmosDb()` as a check. It asks for a connection string and you can optionally also pass in a database name. The health check tries to connect to the database and reports if it experiences issues. Your code should look similar to the example below.

```
services.AddHealthChecks()
    .AddCosmosDb(cosmosDbConnString, "master");
```

You can try it and and pass in a non-existing database name or wrong connection string and go to the `/hc` endpoint. It shows `Unhealthy` in those cases.

Now lets create a custom check for us to implement. This can be anything you want. For demo purposes we're going to create a signleton class that holds a boolean called Healthy. If that boolean is true the check responds with healthy, else it will return unhealthy.

- Create a new class called `HealthService.cs`
- Inside that class add a `bool` property call `Healthy`, make it `true` by default.
- In `Startup.cs` inject the class as a singleton: `services.AddSingleton<HealthService>();`
- Create a new class called `CustomCheck.cs`
- Have the class implement the interface `IHealthCheck`
- Inject the `HealthService` we just created and have it return `Healthy` on true or `Unhealthy`.
- In `Startup.cs` add to the other health checks the code `AddCheck<CustomCheck>(nameof(CustomCheck));`

The code how it should look is below:

```
public class CustomCheck : IHealthCheck
    {
        readonly HealthService _healthService;

        public CustomCheck(HealthService healthService) => _healthService = healthService;

        public Task<HealthCheckResult> CheckHealthAsync(HealthCheckContext context, CancellationToken cancellationToken = default)
            => _healthService.Healthy
                ? Task.FromResult(HealthCheckResult.Healthy())
                : Task.FromResult(HealthCheckResult.Unhealthy());
    }
```

```
Startup.cs
services
    .AddSingleton<HealthService>()
    .AddHealthChecks()
    .AddCosmosDb(cosmosDbConnString, "master")
    .AddCheck<CustomCheck>(nameof(CustomCheck));
```

Some things of interest. The HealthCheckResults we return in our custom check are functions. They can be given a description as parameter to make it clearer why a specific response has failed. Secondly when adding our custom check we have to supply a name. This name is used internally to know which check we are talking about. We didn't supply it with the CosmosDb check as the nuget package implementation made sure a default one was set.

Besides the name we can supply other parameters. We can set a timeout how long a check can take and add tags and use those to better identify checks or even ignore checks with certain tags. 

If we run the code now it still shows `Healthy`, but when you toggle the bool to false it will show `Unhealthy`. Right now the only way to do this is by restarting the application, but if you add an endpoint that would update that value changes can be seen on refresh of the page.

## UI
Besides a simple text answer there is also a lightweight graphical UI provided that makes it more visual. It comes with its own nuget package and similar to the normal health checks can be added with only a few lines of code.

- In the command line type `dotnet add package AspNetCore.HealthChecks.UI`
- In the command line type `dotnet add package AspNetCore.HealthChecks.UI.InMemory.Storage` (There are different storage options for the UI like Sql, Postgress, etc. They all have their own nuget package. For the demo we use the simple InMemory storage)
- In the command line type 'dotnet add package AspNetCore.HealthChecks.UI.Client`
- Add the following line to `Startup.cs`: `services.AddHealthChecksUI().AddInMemoryStorage();`
- Update your `UseEndpoints()` code in `Startup.cs` to the code below:
```
endpoints.MapControllers();
endpoints.MapHealthChecks("/hc", new HealthCheckOptions
{
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});
endpoints.MapHealthChecksUI(options => options.UIPath = "/hc-ui");
```

The `ResponseWriter` change is needed for the UI to fully understand how to display everything. Before the health endpoint only pushed out a single word. Now it will push out a json object which the UI will read and know how to display. You can see this when you go to the health endpoint yourself.

If you run the project and go to `/hc-ui` you see an empty status screen. We setup the code for the UI, but forgot to tell it which checks to look at. Let's do that now. Update your `AddHealthChecksUI` code to the following:

```
services
    .AddHealthChecksUI(options =>
    {
        options.AddHealthCheckEndpoint("Healthcheck API", "/hc");
    })
    .AddInMemoryStorage();
```

We supply it with the health endpoint of this API and give it a name as well. If we look at our UI again we see the following:
![image](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/rnqrp7j7s5x9oj7tsyfk.png)

Write some code to update the bool we used for the custom check and see how the UI is updated. Or rotate the key used for your CosmosDb connection and see that the checks update correctly. You can inspect the details of each check and see when it failed.
![image](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/217qdml7xdff8cie0c8h.png)

In `appsettings.json` you can provide configuration for the UI to look at multiple APIs and display them all on a single place. This way you can group results of different APIs in a single place.

## To the cloud
It is good to have checks and being able to output the data, but we don't want to manually check the dashboard every time to see if there is a problem.
Luckily for us Azure has provided some features that automatically check the health endpoint of a web app and can take an unhealthy instance out of the load balancer until it's healthy again, or even restart or replace it.

In the Azure portal under App service we can setup a Health check.
![image](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ihb6h63cbx5683ynlzxh.png)
There is only one option to set initially and this is `Enable`. Once you select that option two extra options become available. The first one is the path to your health endpoint, which is /hc in our case. The second is the time an app can be unhealthy before it is removed from the load balancer. Once you save your changes every minute a call to the health endpoint will be made.

Azure only looks at the HTTP response the page gives. If the response is in the 2XX range the instance is considered healthy, else it is shown as degraded or unhealthy. It also doesn't follow redirects. If your webapp settings allow http the check by Azure is done over http. If it is set to HTTPS only then Azure used https calls to check the endpoint. In case your health checks return HTTP 307 response make sure you set your webapp to only use https or remove the redirect from your code (it's in there by default).

There are two important configuration values you can set for your webapp.
- `WEBSITE_HEALTHCHECK_MAXPINGFAILURES`: How many failed requests are needed for the service to be removed from the load balancer (is also set by the slider in the portal)
- `WEBSITE_HEALTHCHECK_MAXUNHEALTHYWORKERPERCENT`: How many unhealthy instances will be excluded from the load balancer. 

When an instance is removed from the load balancer Azure keeps checking the health endpoint to see if it returns to being healthy. If this takes to long it will try restarting the underlying VM and if the instance in unhealthy for over an hour it will be replaced. The instance still counts towards your total instances for scaling rules, it only is removed from the load balancer. When scaling happens Azure first checks the health endpoint to make sure the new instance is working correctly before adding it.

Besides the automatic removal of an instance from the load balancer you can also setup alerting rules to get notified when a certain percentage of your instances is in an unhealthy state. 

## References
This is just a small view on what can be done with health checks in ASP.Net Core. In case you are interested feel free to check out the resources below who go into the subject matter a bit deeper.

- https://docs.microsoft.com/en-us/dotnet/architecture/microservices/implement-resilient-applications/monitor-app-health
- https://docs.microsoft.com/en-us/aspnet/core/host-and-deploy/health-check
- https://github.com/Xabaril/AspNetCore.Diagnostics.HealthChecks
- https://docs.microsoft.com/en-us/azure/app-service/monitor-instances-health-check

The demo project used can be found [here](https://github.com/evdbogaard/demo-healthcheck-api).