# Azure Function Monitoring 

## With Health Checks
Azure Function Health is small library based on the DotNet Core HealthChecks feature. 
This library will allow you to register health checks for your dependencies and create an HTTP endpoint that can be used to monitor the health of your application.

## How it works

1. Add the following package to your Azure Function Project: Microsoft.Extensions.Diagnostics.HealthChecks
1. Add a Nuget package for the health check you would like to add: i.e. AspNetCore.HealthChecks.SqlServer
1. Add a reference to the FunctionHealthCheck project listed in this repository
1. Include a startup class to register the specific health checks required for your project.

```c#

using Microsoft.Azure.Functions.Extensions.DependencyInjection;
using Microsoft.Extensions.DependencyInjection;
using Function.HealthCheck.SQL;
using FunctionHealthCheck;
using Microsoft.Extensions.Diagnostics.HealthChecks;
using System;

[assembly: FunctionsStartup(typeof(Startup))]
namespace Function.HealthCheck.SQL
{
    public class Startup : FunctionsStartup
    {
        public override void Configure(IFunctionsHostBuilder builder)
        {
            ConfigureServices(builder.Services);
        }

        public void ConfigureServices(IServiceCollection services)
        {
            if (services == null)
            {
                throw new ArgumentNullException(nameof(services));
            }

            services.AddLogging();

            services.AddFunctionHealthChecks()
                .AddSqlServer(
                  connectionString: "Server=localhost;Database=yourdb;User Id=app;Password=test123");

        }
    }
}

```

5. Add a health check endpoint for your application to expose:

```c#
using System;
using System.IO;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Azure.WebJobs;
using Microsoft.Azure.WebJobs.Extensions.Http;
using Microsoft.AspNetCore.Http;
using Microsoft.Extensions.Logging;
using Newtonsoft.Json;
using Microsoft.Extensions.Diagnostics.HealthChecks;

namespace Function.HealthCheck.SQL
{
    public class HttpFunc
    {
        private readonly HealthCheckService _healthCheck;
        public HttpFunc(HealthCheckService healthCheck)
        {
            _healthCheck = healthCheck;
        }

        [FunctionName("Heartbeat")]
        public async Task<IActionResult> Heartbeat(
           [HttpTrigger(AuthorizationLevel.Anonymous, "get", Route = "heartbeat")] HttpRequest req,
           ILogger log)
        {
            log.Log(LogLevel.Information, "Received heartbeat request");

            var status = await _healthCheck.CheckHealthAsync();

            return new OkObjectResult(Enum.GetName(typeof(HealthStatus), status.Status));
        }

    }
}

```

6. Setup an external monitoring tool like Azure Monitor to create a ping test against this endpoint. https://docs.microsoft.com/en-us/azure/app-service/monitor-instances-health-check
