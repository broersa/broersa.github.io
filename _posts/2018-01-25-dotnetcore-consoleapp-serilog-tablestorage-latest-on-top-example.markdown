---
layout: post
title:  "DotNetCore ConsoleApp Serilog Azure Tablestorage latest on top example"
date:   2018-01-25
categories: dotnetcore
---
# Arrange the Azure Storage

First thing to have in place is a tablestorage account. I will use the Azure Storage Emulator on Windows. To start it press the windows key and type Microsoft Azure Storage Emulator. Run it and it will start the local storage emulator. The connectionstring is:

```
UseDevelopmentStorage=true
```

To explore the storage I use Visual Studio.

# The appliction

```
mkdir Com.Bekijkhet.SerilogTablestorageExample
cd Com.Bekijkhet.SerilogTablestorageExample
dotnet new console
```

We need an extra class to generate the PartitionKey and the RowKey. By default it generates ticks. But than the latest lines are added to the end when browsing the logfiles. We generate the keys so that they are getting smaller during the time. Now the latest lines are automatically on top.

These are the pieces of code needed:

Com.Bekijkhet.SerilogTablestorageExample.csproj:

``` xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>netcoreapp2.0</TargetFramework>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.Extensions.DependencyInjection" Version="2.0.0" />
    <PackageReference Include="Microsoft.Extensions.Logging" Version="2.0.0" />
    <PackageReference Include="Microsoft.Extensions.Logging.Console" Version="2.0.0" />
    <PackageReference Include="Serilog" Version="2.6.0" />
    <PackageReference Include="Serilog.Extensions.Logging" Version="2.0.2" />
    <PackageReference Include="Serilog.Sinks.AzureTableStorage" Version="4.0.0" />
    <PackageReference Include="WindowsAzure.Storage" Version="8.7.0" />
  </ItemGroup>

</Project>
```

Program.cs:

``` csharp
using System;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Logging;
using Microsoft.WindowsAzure.Storage;
using Serilog;
using Serilog.Events;

namespace Com.Bekijkhet.SerilogTablestorageExample
{
    public class Program
    {
        public static void Main(string[] args)
        {
            var serviceCollection = new ServiceCollection();
            
            ConfigureServices(serviceCollection);

             var serviceProvider = serviceCollection.BuildServiceProvider();
 
            serviceProvider.GetService<App>().Run();
        }

        private static void ConfigureServices(IServiceCollection serviceCollection) {
            serviceCollection.AddSingleton(new LoggerFactory()
                .AddSerilog());
            serviceCollection.AddLogging();

            CloudStorageAccount logStorageAccount = CloudStorageAccount.Parse("UseDevelopmentStorage=true");

            // Initialize serilog logger
            Log.Logger = new LoggerConfiguration()
                 .WriteTo.AzureTableStorage(logStorageAccount, LogEventLevel.Verbose, null, "combekijkhetserilogtablestorageexample", false, null, null, new MyKeyGenerator())
                 .MinimumLevel.Debug()
                 .Enrich.FromLogContext()
                 .CreateLogger();

            serviceCollection.AddTransient<App>();
        }
    }
}
```

App.cs:

``` csharp
using Microsoft.Extensions.Logging;

namespace Com.Bekijkhet.SerilogTablestorageExample {
    public class App
    {
        private readonly ILogger<App> _logger;
    
        public App(ILogger<App> logger)
        {
            _logger = logger;
        }
    
        public void Run()
        {
            _logger.LogDebug("Testing 123");
            _logger.LogDebug("Testing 456");
            _logger.LogDebug("Testing 789");
            System.Console.ReadKey();
        }
    }
}
```

MyKeyGenerator.cs:

``` csharp
using System;
using Serilog.Events;
using Serilog.Sinks.AzureTableStorage.KeyGenerator;

namespace Com.Bekijkhet.SerilogTablestorageExample {
    public class MyKeyGenerator : IKeyGenerator
    {
        string IKeyGenerator.GeneratePartitionKey(LogEvent logEvent)
        {
            return (DateTime.MaxValue.Ticks - logEvent.Timestamp.Ticks).ToString().Substring(0, 9);
        }

        string IKeyGenerator.GenerateRowKey(LogEvent logEvent, string suffix)
        {
            return (DateTime.MaxValue.Ticks - logEvent.Timestamp.Ticks).ToString()+"."+Guid.NewGuid().ToString();
        }
    }
}
```

The result in descending order:

![Tablestorage logfile]({{ "/assets/20180125storage.png" | absolute_url }})

Comments are welcome via my twitter account.

The code can be found on my github:
[https://github.com/broersa/Com.Bekijkhet.SerilogTablestorageExample](https://github.com/broersa/Com.Bekijkhet.SerilogTablestorageExample)
